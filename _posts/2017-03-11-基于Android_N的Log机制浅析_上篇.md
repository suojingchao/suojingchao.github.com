

## 写在前面
由于工作原因最近研究了一下Android的Log实现机制，在这里做一个记录，如果有理解不到位的地方也请各位大哥大姐们指点指点。以下涉及的代码均基于Android N。下面开始咯。

-------------------

> Android N中Log机制的总线：
	Log的输出是由不同的进程发起的，但真正实现Log的输入与输出的是一个叫logd的模块，他是一个单独的native进程，当一个应用进程需要打印一条Log时，其实是通过socket与logd模块通信最终由logd来完成Log的处理工作的。
	
> 本系列文章计划按调用栈的顺序分上下两篇，上篇记录从java中调用常用的log打印接口后一直到连接到logd模块的过程。下篇详细记录logd中处理log信息的过程。


## Log Priority
首先，在Android中输出的Log是有优先级的，如下所示列出了Android中Log的所有优先级：
```
	/**
     * Priority constant for the println method; use Log.v.
     */
    public static final int VERBOSE = 2;

    /**
     * Priority constant for the println method; use Log.d.
     */
    public static final int DEBUG = 3;

    /**
     * Priority constant for the println method; use Log.i.
     */
    public static final int INFO = 4;

    /**
     * Priority constant for the println method; use Log.w.
     */
    public static final int WARN = 5;

    /**
     * Priority constant for the println method; use Log.e.
     */
    public static final int ERROR = 6;

    /**
     * Priority constant for the println method.
     */
    public static final int ASSERT = 7;

```
Verbose优先级最低，跟着到Debug优先级依次提高，在这里看到的优先级最高的是Assert。后面会说到基于Log的优先级，提供了一套支持设置Log过滤的方法。

## 从调用开始
我们在平时开发过程中需要在程序中加入一些Log，这个自然大家都知道，Android中为我们提供了以下一些常用的接口实现Log的输出：

```
	/**
     * Send a {@link #VERBOSE} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int v(String tag, String msg) {
        return println_native(LOG_ID_MAIN, VERBOSE, tag, msg);
    }

    /**
     * Send a {@link #DEBUG} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int d(String tag, String msg) {
        return println_native(LOG_ID_MAIN, DEBUG, tag, msg);
    }

    /**
     * Send an {@link #INFO} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int i(String tag, String msg) {
        return println_native(LOG_ID_MAIN, INFO, tag, msg);
    }

    /**
     * Send a {@link #WARN} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int w(String tag, String msg) {
        return println_native(LOG_ID_MAIN, WARN, tag, msg);
    }

    /**
     * Send an {@link #ERROR} log message.
     * @param tag Used to identify the source of a log message.  It usually identifies
     *        the class or activity where the log call occurs.
     * @param msg The message you would like logged.
     */
    public static int e(String tag, String msg) {
        return println_native(LOG_ID_MAIN, ERROR, tag, msg);
    }

    /** @hide */ public static native int println_native(int bufID,
            int priority, String tag, String msg);
```
这几个接口分别实现不同优先级的log的输出，可以看到他们内部都是通过一个native方法println_native实现的。这里需要注意的是println_native函数的第一个参数，它是一个整形，表示当前Log是往哪一个buffer里面去写，后面讲到logd时会看到这里的这个参数控制了我们输出的Log该往logd中的那一块buffer里面填充。println_native的实现在android_util_Log.cpp文件中：

```
/*
 * In class android.util.Log:
 *  public static native int println_native(int buffer, int priority, String tag, String msg)
 */
static jint android_util_Log_println_native(JNIEnv* env, jobject clazz,
        jint bufID, jint priority, jstring tagObj, jstring msgObj)
{
    const char* tag = NULL;
    const char* msg = NULL;

    if (msgObj == NULL) {
        jniThrowNullPointerException(env, "println needs a message");
        return -1;
    }

    if (bufID < 0 || bufID >= LOG_ID_MAX) {
        jniThrowNullPointerException(env, "bad bufID");
        return -1;
    }

    if (tagObj != NULL)
        tag = env->GetStringUTFChars(tagObj, NULL);
    msg = env->GetStringUTFChars(msgObj, NULL);

    int res = __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg);

    if (tag != NULL)
        env->ReleaseStringUTFChars(tagObj, tag);
    env->ReleaseStringUTFChars(msgObj, msg);

    return res;
}

/*
 * JNI registration.
 */
static const JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */
    { "isLoggable",      "(Ljava/lang/String;I)Z", (void*) android_util_Log_isLoggable },
    { "println_native",  "(IILjava/lang/String;Ljava/lang/String;)I", (void*) android_util_Log_println_native },
    { "logger_entry_max_payload_native",  "()I", (void*) android_util_Log_logger_entry_max_payload_native },
};
```
可以看到在android_util_Log.cpp中println_native通过JNI映射后的实现函数其实是android_util_Log_println_native，在android_util_Log_println_native函数中对参数做了一些检查后最终是通过__android_log_buf_write方法，他的参数分别表示需要写入Log的buffer，写入log的优先级，写入log的tag，以及写入log的内容。该方法的实现在/system/core/liblog/logger_write.c中。到这里就由JNI调用进入到了liblog的代码中了。（liblog只是一个so库，logd才是一个单独的进程）。

## liblog
到这一步时，整个程序的Context还是处在发起Log打印的那个进程的上下文中，所以下面的逻辑也是在发起Log打印的这个进程中执行的。接着上面的分析内容我们现在进入到了liblog.so的代码中，__android_log_buf_write方法的实现如下所示：

```
...

static int (*write_to_log)(log_id_t, struct iovec *vec, size_t nr) = __write_to_log_init;

...

LIBLOG_ABI_PUBLIC int __android_log_buf_write(int bufID, int prio,
                                              const char *tag, const char *msg)
{
    struct iovec vec[3];
    char tmp_tag[32];

    if (!tag)
        tag = "";

    /* XXX: This needs to go! */
    if ((bufID != LOG_ID_RADIO) &&
         (!strcmp(tag, "HTC_RIL") ||
        !strncmp(tag, "RIL", 3) || /* Any log tag with "RIL" as the prefix */
        !strncmp(tag, "IMS", 3) || /* Any log tag with "IMS" as the prefix */
        !strcmp(tag, "AT") ||
        !strcmp(tag, "GSM") ||
        !strcmp(tag, "STK") ||
        !strcmp(tag, "CDMA") ||
        !strcmp(tag, "PHONE") ||
        !strcmp(tag, "SMS"))) {
            bufID = LOG_ID_RADIO;
            /* Inform third party apps/ril/radio.. to use Rlog or RLOG */
            snprintf(tmp_tag, sizeof(tmp_tag), "use-Rlog/RLOG-%s", tag);
            tag = tmp_tag;
    }

#if __BIONIC__
    if (prio == ANDROID_LOG_FATAL) {
        android_set_abort_message(msg);
    }
#endif

    vec[0].iov_base = (unsigned char *)&prio;
    vec[0].iov_len  = 1;
    vec[1].iov_base = (void *)tag;
    vec[1].iov_len  = strlen(tag) + 1;
    vec[2].iov_base = (void *)msg;
    vec[2].iov_len  = strlen(msg) + 1;

    return write_to_log(bufID, vec, 3);
}
```

该方法首先针对某些应该归类到RADIO_BUFFER一些特殊tag做了一些处理，使包含有这些特殊的tag的Log以固定的tag前缀输入到RADIO_BUFFER中。这里可以看一下logger_name.c文件，其中定义了所有目前支持的buffer类型：

```
/* In the future, we would like to make this list extensible */
static const char *LOG_NAME[LOG_ID_MAX] = {
    [LOG_ID_MAIN] = "main",
    [LOG_ID_RADIO] = "radio",
    [LOG_ID_EVENTS] = "events",
    [LOG_ID_SYSTEM] = "system",
    [LOG_ID_CRASH] = "crash",
    [LOG_ID_SECURITY] = "security",
    [LOG_ID_KERNEL] = "kernel",
};
```

在处理完特殊tag的RADIO_BUFFER的log后，接下来用iovec结构化我们传入进来的log信息，最终通过write_to_log函数将结构化了的log往指定的Buffer中写，注意这里的write_to_log实际上是一个函数指针，它在一开始指向的其实是__write_to_log_init函数，所以write_to_log调用其实就是调用了__write_to_log_init函数，接下来看看__write_to_log_init函数是如何实现的：

```
static int __write_to_log_init(log_id_t log_id, struct iovec *vec, size_t nr)
{
    __android_log_lock();

    if (write_to_log == __write_to_log_init) {
        int ret;

        ret = __write_to_log_initialize();
        if (ret < 0) {
            __android_log_unlock();
            if (!list_empty(&__android_log_persist_write)) {
                __write_to_log_daemon(log_id, vec, nr);
            }
            return ret;
        }

        write_to_log = __write_to_log_daemon;
    }

    __android_log_unlock();

    return write_to_log(log_id, vec, nr);
}
```

在这个方法中看到，当首次调用时write_to_log函数指针就是指向__write_to_log_init函数的，所以满足条件判断后会调用__write_to_log_initialize做一些初始化的工作，之后会将write_to_log函数指针指向__write_to_log_daemon函数，接着最后通过write_to_log调用的就变成__write_to_log_daemon函数了。所以现在得去看看__write_to_log_daemon是怎么处理我们的log了，由于其方法代码比较多，这里我只截取了部分代码：

```
static int __write_to_log_daemon(log_id_t log_id, struct iovec *vec, size_t nr)
{
	struct android_log_transport_write *node;
	...

    ret = 0;
    i = 1 << log_id;
    write_transport_for_each(node, &__android_log_transport_write) {
        if (node->logMask & i) {
            ssize_t retval;
            retval = (*node->write)(log_id, &ts, vec, nr);
            if (ret >= 0) {
                ret = retval;
            }
        }
    }

	...
	
    return ret;
}
```

这里看到接下来要打印log的操作是通过node这个指针的write这个函数指针指向的函数继续执行的，参数ts是一个时间戳。在往下走之前先看一下node的类型：

```
struct android_log_transport_write {
  struct listnode node;
  const char *name;
  unsigned logMask; /* cache of available success */
  union android_log_context context; /* Initialized by static allocation */

  int (*available)(log_id_t logId);  
  // logd_writer.c  logdAvailable()
  
  int (*open)();                     
  // logd_writer.c  logdOpen()
  
  void (*close)();                   
  // logd_writer.c  logdClose()
  
  int (*write)(log_id_t logId, struct timespec *ts, struct iovec *vec, size_t nr);  
  // logd_writer.c  logdWrite()
};
```

node是自定义的一个结构体android_log_transport_write类型的，可以看到其中有很多函数指针，正是通过这些函数指针指向的函数以socket的方式跟logd进程通信来完成Log打印工作的，这里我直接在注释里给出这个结构体内的几个函数指针真正指向的函数，至于怎么指过去的中间涉及比较复杂的代码和一些数据结构的处理，可以自己通过查阅代码能更清楚的理解，我这里就不过多的粘贴代码了。这里所指向的函数都是在logd_writer.c文件下的。下面我们看看他们的实现：

```
LIBLOG_HIDDEN struct android_log_transport_write logdLoggerWrite = {
    .node = { &logdLoggerWrite.node, &logdLoggerWrite.node },
    .context.sock = -1,
    .name = "logd",
    .available = logdAvailable,
    .open = logdOpen,
    .close = logdClose,
    .write = logdWrite,
};

static int logdOpen()
{
    int i, ret = 0;

    if (logdLoggerWrite.context.sock < 0) {
        i = TEMP_FAILURE_RETRY(socket(PF_UNIX, SOCK_DGRAM | SOCK_CLOEXEC, 0));
        if (i < 0) {
            ret = -errno;
        } else if (TEMP_FAILURE_RETRY(fcntl(i, F_SETFL, O_NONBLOCK)) < 0) {
            ret = -errno;
            close(i);
        } else {
            struct sockaddr_un un;
            memset(&un, 0, sizeof(struct sockaddr_un));
            un.sun_family = AF_UNIX;
            strcpy(un.sun_path, "/dev/socket/logdw");

            if (TEMP_FAILURE_RETRY(connect(i, (struct sockaddr *)&un,
                                           sizeof(struct sockaddr_un))) < 0) {
                ret = -errno;
                close(i);
            } else {
                logdLoggerWrite.context.sock = i;
            }
        }
    }

    return ret;
}

static void logdClose()
{
    if (logdLoggerWrite.context.sock >= 0) {
        close(logdLoggerWrite.context.sock);
        logdLoggerWrite.context.sock = -1;
    }
}

static int logdAvailable(log_id_t logId)
{
    if (logId > LOG_ID_SECURITY) {
        return -EINVAL;
    }
    if (logdLoggerWrite.context.sock < 0) {
        if (access("/dev/socket/logdw", W_OK) == 0) {
            return 0;
        }
        return -EBADF;
    }
    return 1;
}

static int logdWrite(log_id_t logId, struct timespec *ts,
                     struct iovec *vec, size_t nr)
{
    ssize_t ret;
    static const unsigned headerLength = 1;
    struct iovec newVec[nr + headerLength];
    android_log_header_t header;
    size_t i, payloadSize;
    static atomic_int_fast32_t dropped;
    static atomic_int_fast32_t droppedSecurity;

    if (logdLoggerWrite.context.sock < 0) {
        return -EBADF;
    }

    /* logd, after initialization and priv drop */
    if (__android_log_uid() == AID_LOGD) {
        /*
         * ignore log messages we send to ourself (logd).
         * Such log messages are often generated by libraries we depend on
         * which use standard Android logging.
         */
        return 0;
    }

    /*
     *  struct {
     *      // what we provide to socket
     *      android_log_header_t header;
     *      // caller provides
     *      union {
     *          struct {
     *              char     prio;
     *              char     payload[];
     *          } string;
     *          struct {
     *              uint32_t tag
     *              char     payload[];
     *          } binary;
     *      };
     *  };
     */

    header.tid = gettid();
    header.realtime.tv_sec = ts->tv_sec;
    header.realtime.tv_nsec = ts->tv_nsec;

    newVec[0].iov_base = (unsigned char *)&header;
    newVec[0].iov_len  = sizeof(header);

    if (logdLoggerWrite.context.sock > 0) {
        int32_t snapshot = atomic_exchange_explicit(&droppedSecurity, 0,
                                                    memory_order_relaxed);
        if (snapshot) {
            android_log_event_int_t buffer;

            header.id = LOG_ID_SECURITY;
            buffer.header.tag = htole32(LIBLOG_LOG_TAG);
            buffer.payload.type = EVENT_TYPE_INT;
            buffer.payload.data = htole32(snapshot);

            newVec[headerLength].iov_base = &buffer;
            newVec[headerLength].iov_len  = sizeof(buffer);

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, 2));
            if (ret != (ssize_t)(sizeof(header) + sizeof(buffer))) {
                atomic_fetch_add_explicit(&droppedSecurity, snapshot,
                                          memory_order_relaxed);
            }
        }
        snapshot = atomic_exchange_explicit(&dropped, 0, memory_order_relaxed);
        if (snapshot && __android_log_is_loggable(ANDROID_LOG_INFO,
                                                  "liblog",
                                                  ANDROID_LOG_VERBOSE)) {
            android_log_event_int_t buffer;

            header.id = LOG_ID_EVENTS;
            buffer.header.tag = htole32(LIBLOG_LOG_TAG);
            buffer.payload.type = EVENT_TYPE_INT;
            buffer.payload.data = htole32(snapshot);

            newVec[headerLength].iov_base = &buffer;
            newVec[headerLength].iov_len  = sizeof(buffer);

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, 2));
            if (ret != (ssize_t)(sizeof(header) + sizeof(buffer))) {
                atomic_fetch_add_explicit(&dropped, snapshot,
                                          memory_order_relaxed);
            }
        }
    }

    header.id = logId;

    for (payloadSize = 0, i = headerLength; i < nr + headerLength; i++) {
        newVec[i].iov_base = vec[i - headerLength].iov_base;
        payloadSize += newVec[i].iov_len = vec[i - headerLength].iov_len;

        if (payloadSize > LOGGER_ENTRY_MAX_PAYLOAD) {
            newVec[i].iov_len -= payloadSize - LOGGER_ENTRY_MAX_PAYLOAD;
            if (newVec[i].iov_len) {
                ++i;
            }
            break;
        }
    }

    /*
     * The write below could be lost, but will never block.
     *
     * ENOTCONN occurs if logd dies.
     * EAGAIN occurs if logd is overloaded.
     */
    ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, i));
    if (ret < 0) {
        ret = -errno;
        if (ret == -ENOTCONN) {
            __android_log_lock();
            logdClose();
            ret = logdOpen();
            __android_log_unlock();

            if (ret < 0) {
                return ret;
            }

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, i));
            if (ret < 0) {
                ret = -errno;
            }
        }
    }

    if (ret > (ssize_t)sizeof(header)) {
        ret -= sizeof(header);
    } else if (ret == -EAGAIN) {
        atomic_fetch_add_explicit(&dropped, 1, memory_order_relaxed);
        if (logId == LOG_ID_SECURITY) {
            atomic_fetch_add_explicit(&droppedSecurity, 1,
                                      memory_order_relaxed);
        }
    }

    return ret;
}
```
这里就很明显了，logdOpen通过connect系统调用建立与/dev/socket/logdw的socket通信，并得到一个描述符保存在context.sock中，后面的logdWrite便可以通过writev系统调用往该描述符写入我们前面步骤已经结构化完成的Log信息，从而将工作正式交接到logd模块中。这里着重讨论一下logdWrite函数：

```
static int logdWrite(log_id_t logId, struct timespec *ts,
                     struct iovec *vec, size_t nr)
{
    ssize_t ret;
    static const unsigned headerLength = 1;
    struct iovec newVec[nr + headerLength];
    android_log_header_t header;
    size_t i, payloadSize;
    static atomic_int_fast32_t dropped;
    static atomic_int_fast32_t droppedSecurity;

    if (logdLoggerWrite.context.sock < 0) {
        return -EBADF;
    }

    /* logd, after initialization and priv drop */
    if (__android_log_uid() == AID_LOGD) {
        /*
         * ignore log messages we send to ourself (logd).
         * Such log messages are often generated by libraries we depend on
         * which use standard Android logging.
         */
        return 0;
    }

    /*
     *  struct {
     *      // what we provide to socket
     *      android_log_header_t header;
     *      // caller provides
     *      union {
     *          struct {
     *              char     prio;
     *              char     payload[];
     *          } string;
     *          struct {
     *              uint32_t tag
     *              char     payload[];
     *          } binary;
     *      };
     *  };
     */

    header.tid = gettid();
    header.realtime.tv_sec = ts->tv_sec;
    header.realtime.tv_nsec = ts->tv_nsec;

    newVec[0].iov_base = (unsigned char *)&header;
    newVec[0].iov_len  = sizeof(header);

    if (logdLoggerWrite.context.sock > 0) {
        int32_t snapshot = atomic_exchange_explicit(&droppedSecurity, 0,
                                                    memory_order_relaxed);
        if (snapshot) {
            android_log_event_int_t buffer;

            header.id = LOG_ID_SECURITY;
            buffer.header.tag = htole32(LIBLOG_LOG_TAG);
            buffer.payload.type = EVENT_TYPE_INT;
            buffer.payload.data = htole32(snapshot);

            newVec[headerLength].iov_base = &buffer;
            newVec[headerLength].iov_len  = sizeof(buffer);

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, 2));
            if (ret != (ssize_t)(sizeof(header) + sizeof(buffer))) {
                atomic_fetch_add_explicit(&droppedSecurity, snapshot,
                                          memory_order_relaxed);
            }
        }
        snapshot = atomic_exchange_explicit(&dropped, 0, memory_order_relaxed);
        if (snapshot && __android_log_is_loggable(ANDROID_LOG_INFO,
                                                  "liblog",
                                                  ANDROID_LOG_VERBOSE)) {
            android_log_event_int_t buffer;

            header.id = LOG_ID_EVENTS;
            buffer.header.tag = htole32(LIBLOG_LOG_TAG);
            buffer.payload.type = EVENT_TYPE_INT;
            buffer.payload.data = htole32(snapshot);

            newVec[headerLength].iov_base = &buffer;
            newVec[headerLength].iov_len  = sizeof(buffer);

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, 2));
            if (ret != (ssize_t)(sizeof(header) + sizeof(buffer))) {
                atomic_fetch_add_explicit(&dropped, snapshot,
                                          memory_order_relaxed);
            }
        }
    }

    header.id = logId;

    for (payloadSize = 0, i = headerLength; i < nr + headerLength; i++) {
        newVec[i].iov_base = vec[i - headerLength].iov_base;
        payloadSize += newVec[i].iov_len = vec[i - headerLength].iov_len;

        if (payloadSize > LOGGER_ENTRY_MAX_PAYLOAD) {
            newVec[i].iov_len -= payloadSize - LOGGER_ENTRY_MAX_PAYLOAD;
            if (newVec[i].iov_len) {
                ++i;
            }
            break;
        }
    }

    /*
     * The write below could be lost, but will never block.
     *
     * ENOTCONN occurs if logd dies.
     * EAGAIN occurs if logd is overloaded.
     */
    ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, i));
    if (ret < 0) {
        ret = -errno;
        if (ret == -ENOTCONN) {
            __android_log_lock();
            logdClose();
            ret = logdOpen();
            __android_log_unlock();

            if (ret < 0) {
                return ret;
            }

            ret = TEMP_FAILURE_RETRY(writev(logdLoggerWrite.context.sock, newVec, i));
            if (ret < 0) {
                ret = -errno;
            }
        }
    }

    if (ret > (ssize_t)sizeof(header)) {
        ret -= sizeof(header);
    } else if (ret == -EAGAIN) {
        atomic_fetch_add_explicit(&dropped, 1, memory_order_relaxed);
        if (logId == LOG_ID_SECURITY) {
            atomic_fetch_add_explicit(&droppedSecurity, 1,
                                      memory_order_relaxed);
        }
    }

    return ret;
}
```
函数中的参数vec是我们传进来的已经结构化好的log信息，这里一开始以vec.length+1的长度构造了一个新的iovec类型的数组newVec，并且看到定义了两个static的原子变量dropped和droppedSecurity。接着构造一个log的head信息放到newVec[0]的位置，接下来的会根据上面定义的两个原子变量dropped和droppedSecurity的值判断是否需要将log丢弃并往EVENT_BUFFER中输出一条记录log打印本身发生异常的一条log。这两个原子变量的赋值是在函数的结尾根据writev系统调用是否成功来决定的，当往logd的socket描述符写入包含head信息的结构化log信息时，如果返回EAGAIN类型的错误则会对两个原子变量赋值。跳过中间跟异常处理相关的逻辑，接下来开始把传入的vec信息拷贝到newVec中，最终newVec将构成一个包含head信息的结构化log实体。然后通过writev系统条用将数据扔给logd处理。

到这里我觉得上篇差不多了，这里我只是把大致的流程做了一个介绍和记录，关系某些细节部分其实还是阅读源码来的清晰明了。关于logd对log究竟怎样处理的我计划在下篇中详细介绍。额 貌似代码贴的有点多了 -_-' 。

多谢~


