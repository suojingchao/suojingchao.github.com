
>打入MessageQueue内部（Java篇）

>消息发送，接受，处理相关细节深入分析


这篇文章主要分享一下自己在研究Android消息队列时总结的几个知识点：
1.Handler,Looper,MessageQueue之间的关系。
2.消息的发送流程。
3.消息的获取处理流程。
4.障碍消息介绍。
5.异步消息介绍。
6.IdleHandler接口的使用。

 

##基本概念

这也是对自己研究学习过程的一个记录。开门见山，就从大家最熟悉的Handler开始叨叨吧。
平常使用Handler最常见就是有两种方式：
1.调用无参的构造方法
**Handler.java**

```java
/**
* Default constructor associates this handler with the {@link Looper} for the
* current thread.
*
* If this thread does not have a looper, this handler won't be able to receive messages
* so an exception is thrown.
*/
public Handler() {
  this(null, false);
}
/**
* Use the {@link Looper} for the current thread with the specified callback interface
* and set whether the handler should be asynchronous.
*
* Handlers are synchronous by default unless this constructor is used to make
* one that is strictly asynchronous.
*
* Asynchronous messages represent interrupts or events that do not require global ordering
* with respect to synchronous messages. Asynchronous messages are not subject to
* the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
*
* @param callback The callback interface in which to handle messages, or null.
* @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
* each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
*
* @hide
*/
public Handler(Callback callback, boolean async) {
  if (FIND_POTENTIAL_LEAKS) {
    final Class<? extends Handler> klass = getClass();
    if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
       (klass.getModifiers() & Modifier.STATIC) == 0) {
       Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
       klass.getCanonicalName());
    }
  }
  //初始化Handler中的Looper
  mLooper = Looper.myLooper();
  if (mLooper == null) {
    throw new RuntimeException(
    "Can't create handler inside thread that has not called Looper.prepare()");
  }
  //这里看到，原来Handler中的消息队列和Looper中的消息队列指向的是同一个MessageQueue
  mQueue = mLooper.mQueue;
  mCallback = callback;
  //参数mAsynchronous跟稍后介绍的障碍消息和异步消息有关。后面会详细介绍
  mAsynchronous = async;
}
```


 

第一种构造方式没有传入任何信息,代码注释里已经写的比较清楚了，这里主要做了三件事情：

1）.获取当前线程绑定的Looper。
2）.初始化Handler中的MessageQueue，这里将mQueue初始化为当前Looper中的MessageQueue的一份引用。
3）.赋值相关传入参数。
可一看到Handler中主要的成员就是Looper和MessageQueue。之所以能通过Handler发送消息，这些发送的消息都是发送到MessageQueue中的，并且该MessageQueue正是Looper中的MessageQueue。当调用无参的构造方法时，Handler中初始化的Looper就是当前线程所对应的Looper。

2.传入Looper的构造方法：

**Handler.java**

```java
/**
* Use the provided {@link Looper} instead of the default one.
*
* @param looper The looper, must not be null.
*/
public Handler(Looper looper) {
  this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
  mLooper = looper;
  mQueue = looper.mQueue;
  mCallback = callback;
  mAsynchronous = async;
}
```



第二种方式允许我们在构造时传了指定的Looper，这个时候Handler初始化过程中使用的Looper将由我们传入的Looper决定。使用哪个Looper直接决定了Handler的消息发送将发送到哪个MessageQueue中。可见Looper才是大boss，到这里我们先了解一下Looper：

**Looper.java**

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

private Looper(boolean quitAllowed) {
  mQueue = new MessageQueue(quitAllowed);
  mThread = Thread.currentThread();
}

public static void prepare() {
  prepare(true);
}

private static void prepare(boolean quitAllowed) {
  if (sThreadLocal.get() != null) {
    throw new RuntimeException("Only one Looper may be created per thread");
  }
  sThreadLocal.set(new Looper(quitAllowed));
  //...
}

/**
* Return the Looper object associated with the current thread. Returns
* null if the calling thread is not associated with a Looper.
*/
public static Looper myLooper() {
  //返回当前线程TLS中保存的Looper
  return sThreadLocal.get();
}
//...
```



上面列出了Looper的几个主要方法，
构造函数中初始化了与Handler共用的MessageQueue，之前看到Handler的构造函数最后调用了Looper.myLooper方法，Looper.myLooper方法很简单,直接调用类型为ThreadLocal的sThreadLocal成员变量的get方法去尝试返回当前线程TLS中保存的对应的Looper对象，ThreadLocal你可以理解它为线程级别的存储介质，即线程本地存储空间，各个线程的ThreadLocal空间相互独立互不影响，一个线程只对应一份ThreadLocal空间，所以将Looper存储到ThreadLocal中便意味着一个线程中只会存在一个Looper。
从上面Looper的代码可以看出，往当前线程的TLS中存放Looper是在prepare方法中实现的。再回到我们的Handler的构造方法中，如果通过Looper.myLooper方法返回为空，抛出一个熟悉的异常：
**Handler.java**

```java
throw new RuntimeException(
"Can't create handler inside thread that has not called Looper.prepare()");
```



所以在使用Handler时必须保证与其绑定的线程的Looper是存在的，即该Looper的prepare方法是被调用过的。在实际情况中我们往往通过手动调用prepare或者使用系统提供的HandlerThread类来保证。实际HandlerThread也只是在run方法中帮我们调用了一下prepare而已。
到这里，Handler,Looper,MessageQueue三者的关系已经很清楚了，Handler内部保存了一个Looper和一个MessageQueue，Looper中也保存了一个MessageQueue。并且Handler和与其绑定的Looper中的MessageQueue是同一个消息队列。
所以，Handler其实只是一个封装了发送消息和处理消息的逻辑的类而已，它的存在只是为了我们更方便的向MessageQueue发送消息以及处理消息。调用Handler的发送消息系列的方法时，将消息往Handler中的MessageQueue中发送，而实际上就是往Looper中的MessageQueue中发送消息。然后由Looper.loop()启动的消息循环就会从MessageQueue中获取消息来处理。多么典型的生产者消费者模型。Handler彻头彻尾的扮演一个生产者的角色，Looper通过loop启动消息循环后扮演一个消费者的角色。MessageQueue则是消息存储的地方。接下来我们分析一下消息的发送与接收处理的过程。

说到消息的发送接收，那我们得先来看下这个过程中的主角Message类

**Message.java**

```java
public final class Message implements Parcelable {

//消息处理的时间
/*package*/ long when;

//消息携带的数据
/*package*/ Bundle data;

//消息处理的目标（即消息又谁来处理）
/*package*/ Handler target;

//接受到消息后的处理回调
/*package*/ Runnable callback;

//指向下一个消息，通过持有一个同类型的next变量实现了一个链表结构
// sometimes we store linked lists of these things

/*package*/ Message next;

//Message消息池(静态变量)
private static Message sPool;

//消息池的大小
private static int sPoolSize = 0;

//...
}
```


 

除去Message中大家都认识的what,arg,obj等成员外，这里我列出了跟我后面要说的消息发送和接受相关的一些主要的成员变量，这些变量的含义在上面注释中已经指出。后面在提到时会详细解释。

 

##消息发送
 

上面讲到Handler实际上就是个消息处理和发送的封装，在使用Handler发送消息时，我们常用sendMessage系列和post系列的方法，这两个系列分别都有很多形式，下面咱们一路向北，看看这些个方法是怎么把消息发到MessageQueue中去的

先看sendMessage系列：

**Handler.java**

```java
public final boolean sendMessage(Message msg){
  return sendMessageDelayed(msg, 0);
}



public final boolean sendEmptyMessage(int what){
  return sendEmptyMessageDelayed(what, 0);
}



public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
  Message msg = Message.obtain();
  msg.what = what;
  return sendMessageDelayed(msg, delayMillis);
}

public final boolean sendEmptyMessageAtTime(int what, long uptimeMillis) {
  Message msg = Message.obtain();
  msg.what = what;
  return sendMessageAtTime(msg, uptimeMillis);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis){
  if (delayMillis < 0) {
    delayMillis = 0;
  }
  return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

//最终都会调用到这个方法中
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
  MessageQueue queue = mQueue;
  if (queue == null) {
    RuntimeException e = new RuntimeException(
      this + " sendMessageAtTime() called with no mQueue");
    Log.w("Looper", e.getMessage(), e);
    return false;
  }
  return enqueueMessage(queue, msg, uptimeMillis);
}

//该方法在消息队列的对头插入消息，实现起来非常简单，只用传入 enqueueMessage的第三个参数为0
public final boolean sendMessageAtFrontOfQueue(Message msg) {
  MessageQueue queue = mQueue;
  if (queue == null) {
    RuntimeException e = new RuntimeException(
      this + " sendMessageAtTime() called with no mQueue");
    Log.w("Looper", e.getMessage(), e);
    return false;
  }
  return enqueueMessage(queue, msg, 0);
}
```

可以看到不管调用哪一个方法，最终都是通过 enqueueMessage方法来处理消息的。我们稍后再去看enqueueMessage方法，先看看post系列的方法：

**Handler.java**

```java
public final boolean post(Runnable r){
  return sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean postAtTime(Runnable r, long uptimeMillis){
  return sendMessageAtTime(getPostMessage(r), uptimeMillis);
}

public final boolean postAtTime(Runnable r, Object token, long uptimeMillis){
  return sendMessageAtTime(getPostMessage(r, token), uptimeMillis);
}

public final boolean postDelayed(Runnable r, long delayMillis){
  return sendMessageDelayed(getPostMessage(r), delayMillis);
}

public final boolean postAtFrontOfQueue(Runnable r){
  return sendMessageAtFrontOfQueue(getPostMessage(r));
}
```


这就更很简单了，一目了然，直接调用了对应的sendMessage系列的方法，不过中间经过了一层 getPostMessage的封装，我们看看 getPostMessage方法：

**Handler.java**

```java
private static Message getPostMessage(Runnable r) {
  Message m = Message.obtain();
  //重要 后面提到消息的处理时会看到其用处
  m.callback = r;
  return m;
}

private static Message getPostMessage(Runnable r, Object token) {
  Message m = Message.obtain();
  m.obj = token;
  //重要 后面提到消息的处理时会看到其用处
  m.callback = r;
  return m;
}
```

 

终于露出了真面目，原来post的消息最终也被封装为Message通过sendMessage发送的，只不过将我们需要执行的Runnable封装到了Message的callback成员中。callback的用处等到后面讲到消息处理时会详细叙述。

现在好了，两条发送消息的路都指向 enqueueMessage。ok，多说无益，睁大眼睛看看吧：

**Handler.java**

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
  //重要 原来通过Handler向MessageQueue发送的Message都将其target成员指向了Handler自己，也
  //就意味着该消息后续的处理将由Handler自己来接管
  msg.target = this;
  /*通过Handler的构造函数传入的参数决定，若该Handler设置为异步Handler，则该Handler发送的消 
  息都将是异步消息，通过调用Message的setAsynchronous(true)来将消息设置为异步消息。
  注意这里的异步消息并不是通常意义上的真正的跑在不同线程的异步消息，
  这里的异步消息是与其他消息处在同一个消息队列的异步消息，通过MessageQueue中另一个叫做障碍消息的概念来配合实现异步*/
  if (mAsynchronous) {
    msg.setAsynchronous(true);
  }
  //通过调用 MessageQueue的 enqueueMessage方法，最终将工作阵营成功的转入了MessageQueue中
  return queue.enqueueMessage(msg, uptimeMillis);
}
```

注释中说到一个异步消息的概念，也解释了其不是真正的异步。需要一个叫做障碍消息的东西来配合，我靠，障碍消息是什么鬼，我们就先去看看什么叫障碍消息，往消息队列中插入一个障碍消息是在MessageQueue中实现的，它不经过Handler，通过Looper.postSyncBarrier直接调用：

**MessageQueue.java**

```java
int enqueueSyncBarrier(long when) {
  // Enqueue a new sync barrier token.
  // We don't need to wake the queue because the purpose of a barrier is to stall it.
  synchronized (this) {
    final int token = mNextBarrierToken++;
    
    //构造Message
    final Message msg = Message.obtain();
    msg.markInUse();
    msg.when = when;
    msg.arg1 = token;
    Message prev = null;
    Message p = mMessages;

    //when不等于0，说明需要在when表示的这个时间点去处理该消息
    if (when != 0) {

      //遍历链表并根据消息处理的时间点寻找新消息插入到消息队列的位置
      while (p != null && p.when <= when) {
        prev = p;
        p = p.next;
      }
    }

    //队列不为空，插入msg
    if (prev != null) { // invariant: p == prev.next
      msg.next = p;
      prev.next = msg;
    }
    //队列为空，插入msg
    else {
      msg.next = p;
      mMessages = msg;
    }
  return token;
 }
}
```

这个方法是MessageQueue中的方法，在MessageQueue中消息队列其实就是通过mMessages成员实现的，mMessages是Message类型，前面提到Message通过其内部的next形成了一个链表的结构。所以MessageQueue中这一个mMessages，就构成了java层真正的消息队列。可以看到 enqueueSyncBarrier在生成了一个Message后并没有经过其他步骤，直接开始在消息队列中寻找合适位置并将新消息插进去，感觉有什么不妥？这个接口是在MessageQueue中的，从消息的生成到插入真正的消息队列，在这一个方法搞定，那么问题来了，这个Message交由谁处理呢？换句话说，它的target成员没有赋值啊！的确，它的target就是null,这就是障碍消息。（你可以简单的认为消息队列中target为null的消息就是所谓的障碍消息）。至于它干嘛用的，在讲到消息处理时给您揭晓。

看完了奇葩的障碍消息的介绍，我们接着上面Handler的分析，转战到MessageQueue的 enqueueMessage来继续跟踪消息的发送：

**MessageQueue.java**

```java
boolean enqueueMessage(Message msg, long when) {
  /*这里严格把控了任何消息的target都必须不为null，这个方法是通过Handler发送消息的方法走进来的，换句话说，
  我们通过Handler发送的普通消息的target是不可能为null的。所以说消息队列中如果存在target为null的消息，
  那只能是上面说的通过enqueueSyncBarrier插入的障碍消息了*/
  if (msg.target == null) {
    throw new IllegalArgumentException("Message must have a target.");
  }

  if (msg.isInUse()) {
    throw new IllegalStateException(msg + " This message is already in use.");
  }

  synchronized (this) {
    if (mQuitting) {
      IllegalStateException e = new IllegalStateException(
        msg.target + " sending message to a Handler on a dead thread");
      Log.w("MessageQueue", e.getMessage(), e);
      msg.recycle();
      return false;
    }

    msg.markInUse();
    msg.when = when;
    
    //此处可以简单理解p为一个指向mMessages这个消息队列队头的指针
    Message p = mMessages;
    boolean needWake;
    
    /*如果该消息队列目前为空，或者通过sendMessageAtFrontOfQueue接口调用传进来的，
    或者msg的处理时间点在消息队列队头的消息的处理时间点之前，则将msg插入队头*/
    if (p == null || when == 0 || when < p.when) {
      // New head, wake up the event queue if blocked.
      msg.next = p;
      mMessages = msg;
      needWake = mBlocked;
    } else {

      /* 官方注释说的很明白，当消息队列中有内容并且不是往消息队列的队头插入新消息时，
      只有当当前消息队列队头的消息为障碍消息（看到吧，判断条件中正是通过target==null来断定消息是否是障碍消息），
      并且msg是消息队列中的最早的异步消息时，才需要唤醒消息队列。*/
      // Inserted within the middle of the queue. Usually we don't have to wake
      // up the event queue unless there is a barrier at the head of the queue
      // and the message is the earliest asynchronous message in the queue.
      needWake = mBlocked && p.target == null && msg.isAsynchronous();
      Message prev;
      //寻找合适的插入新消息的位置
      for (;;) {
        prev = p;
        p = p.next;
        //遍历消息队列并按消息执行时间先后顺序查找新消息合适的插入位置。
        if (p == null || when < p.when) {
          break;
        }
        if (needWake && p.isAsynchronous()) {
          needWake = false;
        }
      }

      //新消息插入消息队列中，可以看到就是改变Message中next成员的指向，将新消息接入到消息队列链表中。
      msg.next = p; // invariant: p == prev.next
      prev.next = msg;
    }

    // We can assume mPtr != 0 because mQuitting is false.
    if (needWake) {
      //如果需要唤醒，则调用该方法在Native层唤醒。
      nativeWake(mPtr);
    }
  }
  return true;
}
```

代码中的注释已经讲的比较清楚了，这里只提醒一下真正的消息队列是通过mMessages这个成员通过链表数据结构来表示的。

到这里，消息总算是顺利到达消息队列了，消息的发送过程先告一段落。可是 nativeWake是唤醒啥呢？处理消息的步骤又是在哪里阻塞等待唤醒的呢？接下来一起看看消息处理的过程也许会有答案。

##消息处理
 

上面介绍了Looper是线程唯一的，我们通常也说通过Looper.loop方法来开启消息循环。那咱们就从这个方法入手，看看这个方法怎样开启消息循环的：

**Looper.java**

```java
public static void loop() {
  final Looper me = myLooper();
  if (me == null) {
    throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
  }
  final MessageQueue queue = me.mQueue;

  // Make sure the identity of this thread is that of the local process,
  // and keep track of what that identity token actually is.
  Binder.clearCallingIdentity();
  final long ident = Binder.clearCallingIdentity();

  for (;;) {
    //通过MessageQueue.next方法获取消息队列中下一个待处理的消息
    Message msg = queue.next(); // might block
    if (msg == null) {
      // No message indicates that the message queue is quitting.
      return;
    }

    // This must be in a local variable, in case a UI event sets the logger
    Printer logging = me.mLogging;
    if (logging != null) {
      logging.println(">>>>> Dispatching to " + msg.target + " " +
      msg.callback + ": " + msg.what);
    }

    /*这里进一步说明target的重要性，获取到的消息是通过消息的target成员来决定由谁派发处理的，
    前面看到消息的target都被赋值为Handler，所以这里实际会跑到Handler的dispatchMessage中*/
    msg.target.dispatchMessage(msg);

    if (logging != null) {
      logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
    }

    // Make sure that during the course of dispatching the
    // identity of the thread wasn't corrupted.
    final long newIdent = Binder.clearCallingIdentity();
    if (ident != newIdent) {
      Log.wtf(TAG, "Thread identity changed from 0x"
      + Long.toHexString(ident) + " to 0x"
      + Long.toHexString(newIdent) + " while dispatching to "
      + msg.target.getClass().getName() + " "
      + msg.callback + " what=" + msg.what);
    }
    msg.recycleUnchecked();
  }
}
```

上面看到，关键点就是通过MessageQueue.next方法获取消息，注释也说了，might block，这里是会阻塞滴。

在获取到消息后将交由Handler. dispatchMessage来决定该消息何去何从。下面先看下MessageQueue.next方法：

**MessageQueue.java**

```java
Message next() {
  // Return here if the message loop has already quit and been disposed.
  // This can happen if the application tries to restart a looper after quit
  // which is not supported.
  //mPrt保存的是Native层MessageQueue的引用
  final long ptr = mPtr;
  if (ptr == 0) {
    return null;
  }

  int pendingIdleHandlerCount = -1; // -1 only during first iteration
  int nextPollTimeoutMillis = 0;
  for (;;) {
    if (nextPollTimeoutMillis != 0) {
      Binder.flushPendingCommands();
    }
    //这个方法将进入Native层，本文不打算详细讲Native层的代码，将放到后面的文章中详细介绍，
    //你现在只要清楚，这个方法进入Native层，next方法的阻塞正是由于这里在Native层没有消息时阻塞住的。
    //其最终是通过 Linux 的 epoll 模型来实现。从这里也可以看出，消息的处理会优先处理Native层的消息，
    //其次才处理Java层的消息。
    nativePollOnce(ptr, nextPollTimeoutMillis);
    synchronized (this) {
      // Try to retrieve the next message. Return if found.
      final long now = SystemClock.uptimeMillis();
      Message prevMsg = null;
      //msg是最后查找到的消息，这里初始化为消息队列的队头消息
      Message msg = mMessages;

      //如果队列头第一个消息就是障碍消息，则从队列头开始查找第一个异步消息，并将找到的异步消息作为结果返回，
      //如果队列头只是一个普通消息，则将队列头消息作为准备返回的结果
      if (msg != null && msg.target == null) {
        // Stalled by a barrier. Find the next asynchronous message in the queue.
        do {
          prevMsg = msg;
          msg = msg.next;
        } while (msg != null && !msg.isAsynchronous());
      }
      if (msg != null) {
        if (now < msg.when) {
          // Next message is not ready. Set a timeout to wake up when it is ready.
          nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
        } else {
          // Got a message.
          //从消息队列中删除待返回的msg(剪断链表)
          mBlocked = false;
          if (prevMsg != null) {
            prevMsg.next = msg.next;
          } else {
            mMessages = msg.next;
          }
          msg.next = null;
          if (false) Log.v("MessageQueue", "Returning message: " + msg);
          return msg;
        }
      } else {
        // No more messages.
        nextPollTimeoutMillis = -1;
      }

      // Process the quit message now that all pending messages have been handled.
      if (mQuitting) {
        dispose();
        return null;
      }

      //当消息队列为空或者消息队列中的消息为到执行时间时，检查是否有注册IdleHandler
      // If first time idle, then get the number of idlers to run.
      // Idle handles only run if the queue is empty or if the first message
      // in the queue (possibly a barrier) is due to be handled in the future.
      if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
        //注册的IdleHandler存放在 mIdleHandlers中
        pendingIdleHandlerCount = mIdleHandlers.size();
      }
      if (pendingIdleHandlerCount <= 0) {
        // No idle handlers to run. Loop and wait some more.
        mBlocked = true;
        continue;
      }

      if (mPendingIdleHandlers == null) {
        mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
      }
      mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
    }

    //运行注册的 IdleHandlers
    // Run the idle handlers.
    // We only ever reach this code block during the first iteration.
    for (int i = 0; i < pendingIdleHandlerCount; i++) {
      final IdleHandler idler = mPendingIdleHandlers[i];
      mPendingIdleHandlers[i] = null; // release the reference to the handler
      boolean keep = false;
      try {
        keep = idler.queueIdle();
      } catch (Throwable t) {
        Log.wtf("MessageQueue", "IdleHandler threw exception", t);
      }
      //根据注册的 IdleHandlers的运行返回结果决定是否要保存该IdleHandlers
      if (!keep) {
        synchronized (this) {
          mIdleHandlers.remove(idler);
        }
      }
    }
    // Reset the idle handler count to 0 so we do not run them again.
    pendingIdleHandlerCount = 0;
    // While calling an idle handler, a new message could have been delivered
    // so go back and look again for a pending message without waiting.
    nextPollTimeoutMillis = 0;
  }
}
```

通过上面代码的注释，整个获取消息的流程已经比较清楚了。消息在消息队列中的保存是在消息插入队列时就已经按消息处理的时间点顺序排好序了，所以一般的消息处理只要依次获取队头的消息就可以了。上面注释中说到 nativePollOnce会阻塞当前线程，那么何时唤醒呢？没错，就是前面讲消息发送时提到的MessageQueue.nativeWake 方法,这个方法会进入Native层并最终调用NativeMessageQueue的wake方法通过epoll机制实现唤醒阻塞的线程的目的。

这里再主要说点你不知道的：

1.障碍消息和异步消息非同寻常的关系。

在同一个MessageQueue中，正常情况下获取消息时是由队列头开始获取，在消息插入到MessageQueue时已经保证了该MessageQueue的顺序按照消息的执行时间点排序。Handler中所谓的异步消息只是不保证该消息与同处一个MessageQueue中的其他消息的执行顺序，但他们还是同处于一个消息队列中。这是怎么做到的？答案正是依靠障碍消息。从代码中可以看到，当消息队列的队头是一个障碍消息时，此时只会去找该消息队列中的异步消息，消息队列中的其他普通消息好像被屏蔽了似得都将被阻塞住，好像此时消息队列变成一个只有异步消息的队列一样，从这个意义上讲，确实有点异步的意思。那么问题来了，障碍消息何时被删除呢？在上面的处理过程中并没有看到障碍消息有被删除，当队头为障碍消息时返回的是找到的异步消息，从消息队列删除的也是找到的异步消息。如果障碍消息一直待在队列头部，其他普通消息将永远无法执行了。障碍消息的删除同添加一样，是通过外部方法Looper.removeSyncBarrier来控制的，也就是说，当使用了Looper.postSyncBarrier向消息队列发送一个障碍消息会返回一个int的token值，保存这个token值之后在某个时间点必须保证调用Looper.removeSyncBarrier并传入对应的token来删除对应的障碍消息。障碍消息的主要作用就是用来作为一个障碍，阻塞MessageQueue中的普通消息执行，只执行我们特殊的异步消息。

2.IdleHandler的说明使用。

IdleHandler是一个接口，该接口只有一个方法queueIdle，这个方法返回boolean类型。代码注释中也写了，当消息队列为空或者消息队列中的消息未到执行时间时，会去遍历该MessageQueue注册的IdleHandler列表，并挨个执行其queueIdle方法。所以在queueIdle方法中我们可以指定一些工作来在线程空闲时执行，其返回值决定了该工作是一次性的还是可重复的，若返回false则在执行完该方法后将其从注册列表中删除，若返回true则在下一次线程空闲时依旧可以执行该动作。IdleHandler可以通过MessageQueue的addIdleHandler方法添加，该方法是public的，只要通过Looper.myQueue()获取到MessageQueue实例后，就可以调用addIdleHandler加入自己的IdleHandler了

回到上面说的Looper.loop()中，在MessageQueue,next()返回了一个从消息队列中获取的消息后，将通过msg.target.dispatchMessage(msg)来分发处理消息。前面已经说了target其实就是Handler，所以这里我们直接来看Handler. dispatchMessage()方法是什么鬼：

**Handler.java**

```java
public void dispatchMessage(Message msg) {
  //在调用post系列方法时，封装成的Message的callback属性就是post传入的Runnable对象
  if (msg.callback != null) {
    handleCallback(msg);
  } else {
    // mCallback是Handler内部定义的Callback接口类型，在Handler构造方法时可以当作参数传入
    if (mCallback != null) {
      if (mCallback.handleMessage(msg)) {
        return;
      }
    }
    //通过Handler自己的handleMessage()方法处理消息
    handleMessage(msg);
  }
}


private static void handleCallback(Message message) {
  //执行post的Runnable任务的run方法
  message.callback.run();
}

public interface Callback {
  public boolean handleMessage(Message msg);
}
```

这里看到Handler处理消息主要通过三种途径：

1.直接执行Message.callbcak的run方法。这种情况主要针对通过post系列的方法向消息队列发送的消息处理。

2.通过Handler.Callback接口的handleMessage方法处理消息。在Handler的构造方法中可以传入一个Handler.Callback类型的参数，这个参数会保存在Handler的mCallback成员变量中，并用来处理消息。

3.通过Handler.handleMessage方法处理消息。这是多数情况下我们定义Handler时会重写的一个方法，一般来说我们是通过重写该方法来实现消息的处理的。

从上面的代码也可以看出Handler处理消息的一个优先级：Message.callback > Handler.Callback.handleMessage > Handler.handleMessage

到这里消息处理的流程也大致走完了。

 

下面稍微总结下这篇文章说了说么鬼：

1.Handler,Looper,MessageQueue之间错综复杂的三角恋关系。

2.通过Handler往MessageQueue发送消息的流程分析。

3.从MessageQueue获取消息并分发处理的流程分析。

其中顺带介绍了下障碍消息和同一消息队列的异步消息是个什么鬼

关于障碍消息和异步消息的使用Android将相关api都hide了。因为有关障碍消息的异步消息的配合使用可以看到还是很危险的，一旦消息队列队头存放了一个障碍消息，那么该消息队列的所有普通消息将会被阻塞只能处理异步消息，直到障碍消息删除。而且在ViewRootImpl的scheduleTraversals方法处理界面刷新时可以看到障碍消息的使用，通过障碍消息和异步消息的配合屏蔽了触发界面刷新之后发送的Message，直到下一帧到来。感兴趣可以去看看ViewRootImpl中的实现。所以说如果开放相关的api，上层应用可以自己向主线程中postSyncBarrier的话是非常不安全的行为，至少目前的实现来看是不安全的。但Android又留了一个Message.setAsynchronous()方法，这个方法之前一直也都是hide的，直到SDK22（Android5.1）才公开给上层应用。我可不可以大胆推测，障碍消息和异步消息的亲密关系将很快公之于众。

 

大功告成，打完收功。


