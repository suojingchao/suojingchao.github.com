> 在上篇中从上层使用Log的java接口讲到了liblog中，最终通过socket将要打印的Log交给了logd模块处理，本篇就继续来看一下logd模块内部对Log的处理过程。

## Logd的启动
Logd模块是单独跑在一个进程中的，该进程是直接由init进程fork出来的子进程。

```
shell@M3X:/ $ ps | grep zygote
root      343   1     2137408 86344 poll_sched 0000000000 S zygote64
root      344   1     1579728 75304 poll_sched 0000000000 S zygote
shell@M3X:/ $ ps | grep logd                                                   
logd      268   1     27136  3724  sigsuspend 0000000000 S /system/bin/logd
shell@M3X:/ $ ps | grep init                                                   
root      1     0     13112  1364  SyS_epoll_ 0000000000 S /init
system    293   1     13316  1644  dev_char_r 0000000000 S /system/bin/ccci_mdinit
system    295   1     12284  1636  dev_char_r 0000000000 S /system/bin/ccci_mdinit
```
通过ps可以看到logd进程和zygote一样，他们的ppid都是1，而1正是init进程的pid，所以也印证了他们都是由init进程fork出来的结论，当然你也可以去查看一下系统中的init.rc文件，里面也详细描述了启动logd的时机。

Logd启动后的入口是/system/core/logd/main.cpp中的main方法，本文聚焦上层打印Log后Logd的处理过程，因此省略部分代码：

```
int main(int argc, char *argv[]) {
    
    ...

    // LogBuffer is the object which is responsible for holding all
    // log entries.

    logBuf = new LogBuffer(times);

    // LogReader listens on /dev/socket/logdr. When a client
    // connects, log entries in the LogBuffer are written to the client.

    LogReader *reader = new LogReader(logBuf);
    if (reader->startListener()) {
        exit(1);
    }

    // LogListener listens on /dev/socket/logdw for client
    // initiated log messages. New log entries are added to LogBuffer
    // and LogReader is notified to send updates to connected clients.

    LogListener *swl = new LogListener(logBuf, reader);
    // Backlog and /proc/sys/net/unix/max_dgram_qlen set to large value
    if (swl->startListener(600)) {
        exit(1);
    }

    ...
    
}
```

上面的代码在Logd进程启动后执行，省略了部分初始化工作的代码，这里主看到构造了一个LogBuffer类型的对象logBuf，它的作用是真正存放写入的log。然后构造了一个LogReader类型的对象reader，并将上面构造的logBuf当做参数传给reader，它的作用主要是监听LogBuffer中log的变化或者是否有进程想要读取logd中的log，在有新的log写入时能够被及时通知到，然后可以通过socket通知log的读取方将log取出，通过其startListener()方法开始监听socket是否有请求（比如adb shell logcat）。接下来构造了一个LogListener类型的对象swl，并将reader作为构造方法的参数传递给它，它的作用是监听是否有进程通过socket向logd发起了写入log的请求，通过startListener(600)开始监听。这里的600是socket中的backlog值，具体的含义可以参考[这篇文章](http://www.tuicool.com/articles/EJbi2yn%20%E8%BF%99%E7%AF%87%E6%96%87%E7%AB%A0)。
这里可以理解为logd本身运行在一个独立的进程，他是socket的服务端，跟它对接的client进程有两种角色，一种是将log往logd中写的程序，另一种是往logd中读log的程序。

## 往Logd中写数据

上面分析完后logd已经正常在系统中跑起来了，并且开始监听指定的socket是否有相应的请求。现在就可以接着上篇的调用栈往下看，当有程序需要打印log时，logd时如何处理的:

**LogListener.cpp**

```
bool LogListener::onDataAvailable(SocketClient *cli) {
    static bool name_set;
    if (!name_set) {
        prctl(PR_SET_NAME, "logd.writer");
        name_set = true;
    }

    char buffer[sizeof_log_id_t + sizeof(uint16_t) + sizeof(log_time)
        + LOGGER_ENTRY_MAX_PAYLOAD];
    struct iovec iov = { buffer, sizeof(buffer) };

    char control[CMSG_SPACE(sizeof(struct ucred))] __aligned(4);
    struct msghdr hdr = {
        NULL,
        0,
        &iov,
        1,
        control,
        sizeof(control),
        0,
    };

    int socket = cli->getSocket();

    // To clear the entire buffer is secure/safe, but this contributes to 1.68%
    // overhead under logging load. We are safe because we check counts.
    // memset(buffer, 0, sizeof(buffer));
    ssize_t n = recvmsg(socket, &hdr, 0);
    
    ...

    if (logbuf->log((log_id_t)header->id, header->realtime,
            cred->uid, cred->pid, header->tid, msg,
            ((size_t) n <= USHRT_MAX) ? (unsigned short) n : USHRT_MAX) >= 0) {
        reader->notifyNewLog();
    }

    return true;
}
```

这里只保留了部分主要代码。刚刚也说了监听是否有程序有往logd中写入数据的请求是由LogListener来完成的，它本身继承于SocketListener，LogListener会监听/dev/socket/logdw端口，当它监听的socket有请求时会回调LogListener实现的onDataAvailable回调方法。上篇中可以知道待打印的Log是通过socket传递到logd进程的，所以这里的onDataAvailable回调在监听到有请求后首先完成一些数据结构的初始化工作，其实它主要完成了三件事情：
1.通过recvmsg(socket, &hdr, 0);将数据取到hdr结构中。
2.通过logbuf->log((log_id_t)header->id, header->realtime, cred->uid, cred->pid, header->tid, msg, ((size_t) n <= USHRT_MAX) ? (unsigned short) n : USHRT_MAX)方法将log信息存储到logd的缓存中。
3.如果logbuf->log方法正常将log存储到logd的缓存中，那么通过LogReader的notifyNewLog方法唤醒LogReader。

**LogBuffer.cpp**

```
int LogBuffer::log(log_id_t log_id, log_time realtime,
                   uid_t uid, pid_t pid, pid_t tid,
                   const char *msg, unsigned short len) {
    if ((log_id >= LOG_ID_MAX) || (log_id < 0)) {
        return -EINVAL;
    }

    LogBufferElement *elem = new LogBufferElement(log_id, realtime,
                                                  uid, pid, tid, msg, len);
    
    ...


    pthread_mutex_lock(&mLogElementsLock);

    // Insert elements in time sorted order if possible
    //  NB: if end is region locked, place element at end of list
    LogBufferElementCollection::iterator it = mLogElements.end();
    LogBufferElementCollection::iterator last = it;
    while (last != mLogElements.begin()) {
        --it;
        if ((*it)->getRealTime() <= realtime) {
            break;
        }
        last = it;
    }

    if (last == mLogElements.end()) {
        mLogElements.push_back(elem);
    } else {
        uint64_t end = 1;
        bool end_set = false;
        bool end_always = false;

        LogTimeEntry::lock();

        LastLogTimes::iterator times = mTimes.begin();
        while(times != mTimes.end()) {
            LogTimeEntry *entry = (*times);
            if (entry->owned_Locked()) {
                if (!entry->mNonBlock) {
                    end_always = true;
                    break;
                }
                if (!end_set || (end <= entry->mEnd)) {
                    end = entry->mEnd;
                    end_set = true;
                }
            }
            times++;
        }

        if (end_always
                || (end_set && (end >= (*last)->getSequence()))) {
            mLogElements.push_back(elem);
        } else {
            mLogElements.insert(last,elem);
        }

        LogTimeEntry::unlock();
    }

    stats.add(elem); //将log加入到一些不同维度的统计数据结构中，并且会唤醒LogReader.
    maybePrune(log_id);
    pthread_mutex_unlock(&mLogElementsLock);

    return len;
}
```

长话短说，这里也是省略部分代码。在log()方法中首先将log的封装成了一个LogBufferElement实体，这个实体就是log真正在logd中保存的结构化对象了。mLogElements是一个list列表，接下来的while循环主要是要在mLogElements中找到一个合适的地方插入当前的这个LogBufferElement。在mLogElements列表中的log主要是以时间的顺序来排列的，越新的log排在列表的越前面。log插入到mLogElements后，logd就基本完成这条log的“打印”工作了，没错，logd只是把上层应用想要打印的log“打印”到了内存中，然后通过唤醒LogReader将log从内存中读出来后通过socket还给上层应用完成特定的工作的。这里我们还需要特别关注下末尾的maybePrune(log_id);方法，这里的参数log_id就是上篇中也提到过的需要往哪个Buffer写log的Buffer_id，每个Buffer的大小是有限制的，超过限制后logd会对该Buffer中的log做一些清理工作。每个Buffer的大小是在LogBuffer的构造方法中设置的，默认大小为256K，我们可以通过修改persist.logd.size.XXX的值来自定义某个Buffer的大小，其中XXX代表的是Buffer的名字。有如下几种Buffer:

```
    [LOG_ID_MAIN] = "main",
    [LOG_ID_RADIO] = "radio",
    [LOG_ID_EVENTS] = "events",
    [LOG_ID_SYSTEM] = "system",
    [LOG_ID_CRASH] = "crash",
    [LOG_ID_SECURITY] = "security",
    [LOG_ID_KERNEL] = "kernel",
```
那么到这里上层应用准备打印的log算是安全“打印”到logd的buffer中了，但这只完成了一半工作，上层应用调用接口打印的Log需要通过不同的工具真正打印出来，比如adb shell logcat。这类工具其实也是一个独立的程序，他们是上面提到的往logd中读数据的上层进程，至于数据读出去后是打印还是其他的方式处理，完全取决于上层工具程序的实现逻辑，所以logd要做的其实就是通知这些程序，有log数据更新了。

## 从Logd中读数据

在logd的main方法中除了LogListener外还有一个LogReader监听器，LogReader监听的是/dev/socket/logdr端口，并且在构造LogListener时传入的参数是LogReader的对象。所以在监听到有client向/dev/socket/logdr端口发起请求时，LogReader的onDataAvailable方法也会被回调。在onDataAvailable中会将log返回给client处理。同时再回到上面说的LogListener::onDataAvailable方法中提到的完成的三项主要任务，最后一项便是通过LogListener中的LogReader对象的notifyNewLog方法也可以通知logd的client，这些client是另一个角色，他们是往logd中读出数据去处理的进程，而不是往logd中写数据的进程。notifyNewLog方法中通过一系列的调用最终会通过LogBuffer::flushTo方法来向client发送数据：

**LogBuffer.cpp**

```
uint64_t LogBuffer::flushTo(
        SocketClient *reader, const uint64_t start,
        bool privileged, bool security,
        int (*filter)(const LogBufferElement *element, void *arg), void *arg) {
    LogBufferElementCollection::iterator it;
    uint64_t max = start;
    uid_t uid = reader->getUid();

    pthread_mutex_lock(&mLogElementsLock);


	...


    for (; it != mLogElements.end(); ++it) {
        LogBufferElement *element = *it;

        if (!privileged && (element->getUid() != uid)) {
            continue;
        }

        if (!security && (element->getLogId() == LOG_ID_SECURITY)) {
            continue;
        }

        if (element->getSequence() <= start) {
            continue;
        }

        // NB: calling out to another object with mLogElementsLock held (safe)
        if (filter) {
            int ret = (*filter)(element, arg);
            if (ret == false) {
                continue;
            }
            if (ret != true) {
                break;
            }
        }

        pthread_mutex_unlock(&mLogElementsLock);

        // range locking in LastLogTimes looks after us
        max = element->flushTo(reader, this, privileged);

        if (max == element->FLUSH_ERROR) {
            return max;
        }

        pthread_mutex_lock(&mLogElementsLock);
    }
    pthread_mutex_unlock(&mLogElementsLock);

    return max;
}
```
前面写log正是把log写到这里遍历的mLogElements列表中的，所以这里通过遍历它将logd中的log数据取出，但中间会涉及一些log过滤器的实现，关于log过滤器的逻辑后面如果有机会当做专题文章在做记录吧，这里还是聚焦在最基本的log打印流程上。mLogElements存放的是LogBufferElement类型的对象，遍历每一个LogBufferElement并调用LogBufferElement::flushTo()方法完成对每一个log实体的处理工作：

**LogBufferElement.cpp**

```
uint64_t LogBufferElement::flushTo(SocketClient *reader, LogBuffer *parent,
                                   bool privileged) {
    struct logger_entry_v4 entry;

    memset(&entry, 0, sizeof(struct logger_entry_v4));

    entry.hdr_size = privileged ?
                         sizeof(struct logger_entry_v4) :
                         sizeof(struct logger_entry_v3);
    entry.lid = mLogId;
    entry.pid = mPid;
    entry.tid = mTid;
    entry.uid = mUid;
    entry.sec = mRealTime.tv_sec;
    entry.nsec = mRealTime.tv_nsec;

    struct iovec iovec[2];
    iovec[0].iov_base = &entry;
    iovec[0].iov_len = entry.hdr_size;

    char *buffer = NULL;

    if (!mMsg) {
        entry.len = populateDroppedMessage(buffer, parent);
        if (!entry.len) {
            return mSequence;
        }
        iovec[1].iov_base = buffer;
    } else {
        entry.len = mMsgLen;
        iovec[1].iov_base = mMsg;
    }
    iovec[1].iov_len = entry.len;

    uint64_t retval = reader->sendDatav(iovec, 2) ? FLUSH_ERROR : mSequence;

    if (buffer) {
        free(buffer);
    }

    return retval;
}
```
在LogBufferElement::flushTo()方法中将log重新结构化为iovec的结构类型，并通过LogReader::sendDatav()方法通知到上层的应用中，由它们觉得将log打印还是其他处理。

## 总结
本文只是简单的梳理记录了logd中对整个log从存储到打印的流程。很多细节部分没有展开介绍，受于篇幅所限也没办法展开介绍，想搞清楚logd内部更多有趣细节的朋友们还是阅读代码来的比较畅快淋漓。
其实logd的基本工作模式就是基于两种类型的Client在，针对想要打印log的Client，logd只是将待打印的log写到自己的buffer中，我们真正看到log的打印是由于logd跟另一类(想要处理log的)Client通信的结果。logd将自己buffer中的log通知给想要处理log的这一类Client，由他们来处理这些log，他们可以选择简单的将log打印，也可以做一些其他的更有趣的事情。

- 1.想要打印log的Client
- 2.logd
- 3.想要处理log的Client
三者构成了一个三个进程间或者两个进程间（1和3可能在同一个进程实现）协同处理的和谐社会。


小弟才疏学浅，如果有什么错误还请大哥大姐们帮忙指点迷津~
多谢~





