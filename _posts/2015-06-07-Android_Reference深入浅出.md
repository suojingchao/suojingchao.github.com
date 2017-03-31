> 写在前面： Reference本身是一个接口，表示一个引用，不能直接使用，有四个它的派生类供我们使用，它们分别是：SoftReference,WeakReference,PhantomReference，FinalizerReference .其中SoftReference，WeakReference和 PhantomReference的区别与使用Google一下已经有大把的介绍资料，因此本文对此只简单说明带过，主要给大家介绍你不知道的Reference．

一． SoftReference
-----------------

SoftReference表示一个对象的软引用， SoftReference所引用的对象在发生GC时，如果该对象只被这个SoftReference所引用，那么在内存使用情况已经比较紧张的情况下会释放其所占用的内存，若内存比较充实，则不会释放其所占用的内存．比较常用于一些Cache的实现．
其构造函数中允许传入一个ReferenceQueue．其代码如下所示：
**SoftReference.java**

```
public class SoftReference<T> extends Reference<T> {

    /**
     * Constructs a new soft reference to the given referent. The newly created
     * reference is not registered with any reference queue.
     *
     * @param r the referent to track
     */
    public SoftReference(T r) {
        super(r, null);
    }

    /**
     * Constructs a new soft reference to the given referent. The newly created
     * reference is registered with the given reference queue.
     *
     * @param r the referent to track
     * @param q the queue to register to the reference object with. A null value
     *          results in a weak reference that is not associated with any
     *          queue.
     */
    public SoftReference(T r, ReferenceQueue<? super T> q) {
        super(r, q);
    }
}
```

这个ReferenceQueue才是本文重点之一，后面会专门说到．

二．WeakReference
---------------

WeakReference表示一个对象的弱引用，WeakReference所引用的对象在发生GC时，如果该对象只被这个WeakReference所引用，那么不管当前内存使用情况如何都会释放其所占用的内存．
其构造函数中允许传入一个ReferenceQueue．这个ReferenceQueue才是本文重点之一，后面会专门说到．WeakReference与SoftReference一样派生于Reference类：
**WeakReference.java**

```
public class WeakReference<T> extends Reference<T> {

    /**
     * Constructs a new weak reference to the given referent. The newly created
     * reference is not registered with any reference queue.
     *
     * @param r the referent to track
     */
    public WeakReference(T r) {
        super(r, null);
    }

    /**
     * Constructs a new weak reference to the given referent. The newly created
     * reference is registered with the given reference queue.
     *
     * @param r the referent to track
     * @param q the queue to register to the reference object with. A null value
     *          results in a weak reference that is not associated with any
     *          queue.
     */
    public WeakReference(T r, ReferenceQueue<? super T> q) {
        super(r, q);
    }
}
```

三． PhantomReference
--------------------

PhantomReference表示一个虚引用， 说白了其无法引用一个对象，即对对象的生命周期没有影响．
其代码如下：
**PhantomReference.java**

```
public class PhantomReference<T> extends Reference<T> {

    /**
     * Constructs a new phantom reference and registers it with the given
     * reference queue. The reference queue may be {@code null}, but this case
     * does not make any sense, since the reference will never be enqueued, and
     * the {@link #get()} method always returns {@code null}.
     *
     * @param r the referent to track
     * @param q the queue to register the phantom reference object with
     */
    public PhantomReference(T r, ReferenceQueue<? super T> q) {
        super(r, q);
    }

    /**
     * Returns {@code null}.  The referent of a phantom reference is not
     * accessible.
     *
     * @return {@code null} (always)
     */
    @Override
    public T get() {
        return null;
    }
}
```

可以看到他重写了Reference的get方法直接返回null．所以说它并不是为了改变某个对象的生命周期而存在的．它常用于跟踪某些对象的生命周期状态，它只有一个接受ReferenceQueue的构造方法．正是这个ReferenceQueue的神奇功效帮助PhantomReference实现了跟踪对象生命周期的功能．这里忍不住再一次铺垫，ReferenceQueue马上就来．

四．ReferenceQueue
-----------------

在介绍 ReferenceQueue 之前，先关注下前面介绍的三个引用类的共同的父类Reference．
**Reference.java**

```
public abstract class Reference<T> {

	 ...

    /**
     * The object to which this reference refers.
     * VM requirement: this field <em>must</em> be called "referent"
     * and be an object.
     */
    volatile T referent;

    /**
     * If non-null, the queue on which this reference will be enqueued
     * when the referent is appropriately reachable.
     * VM requirement: this field <em>must</em> be called "queue"
     * and be a java.lang.ref.ReferenceQueue.
     */
    volatile ReferenceQueue<? super T> queue;

    /**
     * Used internally by java.lang.ref.ReferenceQueue.
     * VM requirement: this field <em>must</em> be called "queueNext"
     * and be a java.lang.ref.Reference.
     */
    @SuppressWarnings("unchecked")
    volatile Reference queueNext;


    /**
     * Constructs a new instance of this class.
     */
    Reference() {
    }

    Reference(T r, ReferenceQueue<? super T> q) {
        referent = r;
        queue = q;
    }

    /**
     * Adds an object to its reference queue.
     *
     * @return {@code true} if this call has caused the {@code Reference} to
     * become enqueued, or {@code false} otherwise
     *
     * @hide
     */
    public final synchronized boolean enqueueInternal() {
        if (queue != null && queueNext == null) {
            queue.enqueue(this);
            queue = null;
            return true;
        }
        return false;
    }

    /**
     * Forces the reference object to be enqueued if it has been associated with
     * a queue.
     *
     * @return {@code true} if this call has caused the {@code Reference} to
     * become enqueued, or {@code false} otherwise
     */
    public boolean enqueue() {
        return enqueueInternal();
    }

	...

}
```
这里主要关注Reference的构造方法和equeue方法。看到在Reference中有两个构造方法，其中传入的ReferenceQueue的构造方法将传入的ReferenceQueue保存在其queue这个成员变量中。并且通过enqueue方法调用enqueueInternal将自己添加到queue中。这里大家注意Reference中可能会有一个用于保存自己的queue队列，后面会发现其巧妙的使用方式。ReferenceQueue顾名思义，就是一个引用队列，其内部通过两个Reference类型的成员变量head和tail来构成一个链表结构．并提供了入队出队的相应方法，相关代码如下：
**ReferenceQueue.java**

```
public class ReferenceQueue<T> {
    private static final int NANOS_PER_MILLI = 1000000;

    private Reference<? extends T> head;
    private Reference<? extends T> tail;

    /**
     * Constructs a new instance of this class.
     */
    public ReferenceQueue() {
    }

    /**
     * Returns the next available reference from the queue, removing it in the
     * process. Does not wait for a reference to become available.
     *
     * @return the next available reference, or {@code null} if no reference is
     *         immediately available
     */
    @SuppressWarnings("unchecked")
    public synchronized Reference<? extends T> poll() {
        if (head == null) {
            return null;
        }

        Reference<? extends T> ret = head;

        if (head == tail) {
            tail = null;
            head = null;
        } else {
            head = head.queueNext;
        }

        ret.queueNext = null;
        return ret;
    }

    /**
     * Returns the next available reference from the queue, removing it in the
     * process. Waits indefinitely for a reference to become available.
     *
     * @throws InterruptedException if the blocking call was interrupted
     */
    public Reference<? extends T> remove() throws InterruptedException {
        return remove(0L);
    }

    /**
     * Returns the next available reference from the queue, removing it in the
     * process. Waits for a reference to become available or the given timeout
     * period to elapse, whichever happens first.
     *
     * @param timeoutMillis maximum time to spend waiting for a reference object
     *     to become available. A value of {@code 0} results in the method
     *     waiting indefinitely.
     * @return the next available reference, or {@code null} if no reference
     *     becomes available within the timeout period
     * @throws IllegalArgumentException if {@code timeoutMillis < 0}.
     * @throws InterruptedException if the blocking call was interrupted
     */
    public synchronized Reference<? extends T> remove(long timeoutMillis)
            throws InterruptedException {
        if (timeoutMillis < 0) {
            throw new IllegalArgumentException("timeout < 0: " + timeoutMillis);
        }

        if (head != null) {
            return poll();
        }

        // avoid overflow: if total > 292 years, just wait forever
        if (timeoutMillis == 0 || (timeoutMillis > Long.MAX_VALUE / NANOS_PER_MILLI)) {
            do {
                wait(0);
            } while (head == null);
            return poll();
        }

        // guaranteed to not overflow
        long nanosToWait = timeoutMillis * NANOS_PER_MILLI;
        int timeoutNanos = 0;

        // wait until notified or the timeout has elapsed
        long startTime = System.nanoTime();
        while (true) {
            wait(timeoutMillis, timeoutNanos);
            if (head != null) {
                break;
            }
            long nanosElapsed = System.nanoTime() - startTime;
            long nanosRemaining = nanosToWait - nanosElapsed;
            if (nanosRemaining <= 0) {
                break;
            }
            timeoutMillis = nanosRemaining / NANOS_PER_MILLI;
            timeoutNanos = (int) (nanosRemaining - timeoutMillis * NANOS_PER_MILLI);
        }
        return poll();
    }

    /**
     * Enqueue the reference object on the receiver.
     *
     * @param reference
     *            reference object to be enqueued.
     */
    synchronized void enqueue(Reference<? extends T> reference) {
        if (tail == null) {
            head = reference;
        } else {
            tail.queueNext = reference;
        }

        // The newly enqueued reference becomes the new tail, and always
        // points to itself.
        tail = reference;
        tail.queueNext = reference;
        notify();
    }

	/** @hide */
    public static Reference<?> unenqueued = null;

    static void add(Reference<?> list) {
        synchronized (ReferenceQueue.class) {
            if (unenqueued == null) {
                unenqueued = list;
            } else {
                // Find the last element in unenqueued.
                Reference<?> last = unenqueued;
                while (last.pendingNext != unenqueued) {
                  last = last.pendingNext;
                }
                // Add our list to the end. Update the pendingNext to point back to enqueued.
                last.pendingNext = list;
                last = list;
                while (last.pendingNext != list) {
                    last = last.pendingNext;
                }
                last.pendingNext = unenqueued;
            }
            ReferenceQueue.class.notifyAll();
        }
    }
}
```

通过poll方法弹出队列头部存储的Reference．通过remove方法可以将poll变成block的，即队列为空时remove方法可以试当前线程阻塞住，等到enqueue时通过notify再将block唤醒．大家需要着重注意的是最后@hide起来的那个static成员变量unenqueued和add方法。通过add方法将参数表示的Reference添加到unenqueued描述的一个队列中。并通过ReferenceQueue.class.notifyAll()唤醒某处被阻塞住的线程。这里留两个疑问：1.唤醒的是哪个线程？2.这个add方法又是在哪里被调用的呢？我们先来看一下Daemons.java中的一个守护线程ReferenceQueueDaemon干了什么，真相自会浮出水面。
**Daemons.java**

```
public final class Daemons {
/**
     * This heap management thread moves elements from the garbage collector's
     * pending list to the managed reference queue.
     */
    private static class ReferenceQueueDaemon extends Daemon {
        private static final ReferenceQueueDaemon INSTANCE = new ReferenceQueueDaemon();

        ReferenceQueueDaemon() {
            super("ReferenceQueueDaemon");
        }

        @Override public void run() {
            while (isRunning()) {
                Reference<?> list;
                try {
                    synchronized (ReferenceQueue.class) {
                        while (ReferenceQueue.unenqueued == null) {
                            ReferenceQueue.class.wait();
                        }
                        list = ReferenceQueue.unenqueued;
                        ReferenceQueue.unenqueued = null;
                    }
                } catch (InterruptedException e) {
                    continue;
                }
                enqueue(list);
            }
        }

        private void enqueue(Reference<?> list) {
            Reference<?> start = list;
            do {
                // pendingNext is owned by the GC so no synchronization is required.
                Reference<?> next = list.pendingNext;
                list.pendingNext = null;
                list.enqueueInternal();
                list = next;
            } while (list != start);
        }
    }
```
Daemons.java中定义了4个守护线程（Android_M之前是5个，其中就包括鼎鼎大名的GC线程。但再Android_M中GCDaemon和HeapTrimDaemon合并了）。并且在fork出进程的时候会将Daemons.java中定义的几个守护线程都跑起来。关于Daemons在后续GC的专题讨论中我会具体介绍。这里我们主要看其中的一个守护线程ReferenceQueueDaemon，我们看它的run方法中首先判断ReferenceQueue的静态成员变量unqueue是否为空，空则阻塞住当前线程，这里的ReferenceQueue.class.wait()有点似曾相识，没错，刚刚我们再找ReferenceQueue的add方法唤醒了哪个线程，唤醒就是这个ReferenceQueueDaemon守护线程。如果不为空，则通过enqueue调用unqueue所指向的Reference的enqueueInternal()方法。前面分析Reference的enqueueInternal()方法知道它将自己所表示的Reference添加到自己的queue成员中，这个queue成员就是构造Referene时传进去的ReferenceQueue。现在上面提到的问题1解决了，那问题2.ReferenceQueue的add方法是哪里调用的呢？答案是从虚拟机里面调出来的，在虚拟机内部完成GC时就会通过JNI反调回ReferenceQueue的add方法中。关于虚拟机内部反调回ReferenceQueue的过程再后续的GC专题会详细叙述。
到这里也许会有一点晕，来个小总结：
ReferenceQueueDaemon在应用启动后就开始工作，任务是从ReferenceQueue.unqueue中读出需要处理的Reference。并将读出的Reference放入构造其自身时传入的ReferenceQueue中。
虚拟机在每次GC完成后会调用ReferenceQueue.add方法将这次GC释放的内存的对象所对应的Reference添加到ReferenceQueue.unqueue中
一个典型的生产者消费者模型。
当然，当不使用Reference时，或者构造Reference不传入ReferenceQueue时，这部分处理工作其实是直接跳过的。
所以说到这里，ReferenceQueue的作用也很明显了，它就是起到了一个监控对象生命周期的作用。即当对象被GC回收时，倘若为它创建了带ReferenceQueue的Reference，那么会将这个Reference加入到构造它时传入的ReferenceQueue中。这样我们遍历这个ReferenceQueue就知道被监控的对象是否被GC回收了。前面说的PhantomReference通常用来监控对象的生命周期也就是这个原理。


五． FinalizerReference．
----------------------

FinalizerReference主要是为了协助FinalizerDaemon守护线程完成对象的finalize工作而生的．
其主要代码如下：
**FinalizerReference.java**

```

/**
 * @hide
 */
public final class FinalizerReference<T> extends Reference<T> {
    // This queue contains those objects eligible for finalization.
    public static final ReferenceQueue<Object> queue = new ReferenceQueue<Object>();

    // Guards the list (not the queue).
    private static final Object LIST_LOCK = new Object();

    // This list contains a FinalizerReference for every finalizable object in the heap.
    // Objects in this list may or may not be eligible for finalization yet.
    private static FinalizerReference<?> head = null;

    // The links used to construct the list.
    private FinalizerReference<?> prev;
    private FinalizerReference<?> next;

    // When the GC wants something finalized, it moves it from the 'referent' field to
    // the 'zombie' field instead.
    private T zombie;

    public FinalizerReference(T r, ReferenceQueue<? super T> q) {
        super(r, q);
    }

    @Override public T get() {
        return zombie;
    }

    @Override public void clear() {
        zombie = null;
    }

    public static void add(Object referent) {
        FinalizerReference<?> reference = new FinalizerReference<Object>(referent, queue);
        synchronized (LIST_LOCK) {
            reference.prev = null;
            reference.next = head;
            if (head != null) {
                head.prev = reference;
            }
            head = reference;
        }
    }

    public static void remove(FinalizerReference<?> reference) {
        synchronized (LIST_LOCK) {
            FinalizerReference<?> next = reference.next;
            FinalizerReference<?> prev = reference.prev;
            reference.next = null;
            reference.prev = null;
            if (prev != null) {
                prev.next = next;
            } else {
                head = next;
            }
            if (next != null) {
                next.prev = prev;
            }
        }
    }
}
```

可以看到 FinalizerReference内部定义了一个static的ReferenceQueue对象queue．这个queue在add方法中作为FinalizerReference的构造方法参数构造了一个FinalizerReference对象，并将构造的FinalizerReference对象加入到他自身维护的一个队列中．remove方法从其自身维护的队列中删除指定的Reference。另外看到FinalizerReference的get方法返回的是zombie成员。这个成员是在虚拟机中从referent拷贝过来的（后面介绍GC时会详细说明）。
简单来说，FinalizerReference就是一个派生自Reference的类，内部实现了一个由head,prev,next维护的队列,还有一个自己定义的成员变量queue。它的蹊跷之处就在这个queue成员变量和add方法。
在其add方法中使用这个queue和参数中的一个对象构造了一个FinalizerReference，并将其插入自己维护的队列中。根据前面对ReferenceQueue的说明，当这个被FinalizerReference引用的对象被GC释放其所占用的内存堆空间时，会把这个对象的FinalizerReference引用插入到这个queue中。这个add方法同样是从虚拟机中反调回来的，当一个对象实现了finalize方法，虚拟机中能够检测到，并且反调这个add方法将实现了finalize方法的对象当做参数传出来。即所有实现了finalize方法的对象的生命周期都被FinalizerReference的queue所监控着，当GC发生时queue中就会插入当前正准备释放内存的对象的FinalizerReference引用。到这里能很清晰看出这个也是一个典型的围绕这个queue成员变量的生产者消费者模型，生产者已经找到，接下来看下哪里去消费这个queue呢？我们还是将目光转向Daemons.java
**Daemons.java**

```
public final class Daemons {

	...
		private static class FinalizerDaemon extends Daemon {
        private static final FinalizerDaemon INSTANCE = new FinalizerDaemon();
        private final ReferenceQueue<Object> queue = FinalizerReference.queue;
        private volatile Object finalizingObject;
        private volatile long finalizingStartedNanos;

        FinalizerDaemon() {
            super("FinalizerDaemon");
        }

        @Override public void run() {
            while (isRunning()) {
                // Take a reference, blocking until one is ready or the thread should stop
                try {
                    doFinalize((FinalizerReference<?>) queue.remove());
                } catch (InterruptedException ignored) {
                }
            }
        }

        @FindBugsSuppressWarnings("FI_EXPLICIT_INVOCATION")
        private void doFinalize(FinalizerReference<?> reference) {
            FinalizerReference.remove(reference);
            Object object = reference.get();
            reference.clear();
            try {
                finalizingStartedNanos = System.nanoTime();
                finalizingObject = object;
                synchronized (FinalizerWatchdogDaemon.INSTANCE) {
                    FinalizerWatchdogDaemon.INSTANCE.notify();
                }
                object.finalize();
            } catch (Throwable ex) {
                // The RI silently swallows these, but Android has always logged.
                System.logE("Uncaught exception thrown by finalizer", ex);
            } finally {
                // Done finalizing, stop holding the object as live.
                finalizingObject = null;
            }
        }
    }
    
	...
}
```
FinalizerDaemon是Daemons.java中定义的另一个守护线程，FinalizerReference中定义的queue的消费者就是它。它内部定义了一个ReferenceQueue类型的对象queue，并将其赋值为前面说的FinalizerReference中的定义的那个queue。run方法中通过ReferenceQueue的remove方法把保存在queue中的Reference获取出来并通过doFinalize方法做下一步处理。前面提过ReferenceQueue的remove方法是阻塞的，在队列中没有Reference时将阻塞直到有Reference入队。我们看一下doFinalize方法，通过从队列中获取出来的reference的get方法获取到被引用的真实对象，并在这里调用该对象的finalize方法。但在这之前会通过FinalizerWatchdogDaemon.INSTANCE.notify()唤醒FinalizerWatchdogDaemon守护线程，FinalizerWatchdogDaemon在稍后介绍。
总结起来，FinalizerQueue和FinalizerDaemon组合起来完成了在合适的时机去调用我们实现的finalize方法的工作:虚拟机检测到有对象实现了finalize方法会调用FinalizerQueue的add方法使得在GC的时候能将实现了finalize方法的对象的引用加入到FinalizerQueue的queue成员中。而FinalizerDaemon则从FinalizerQueue的queue中取出跟踪的引用并调用被引用对象的finalize方法。
上面提到的FinalizerWatchdogDaemon同样是定义在Daemons.java中的一个守护线程。它的代码比较简单，感兴趣的朋友可以去看一下。这里主要介绍下它的作用。它主要用来监控finalize方法执行的时长，并在finalize执行超时时会抛出finalize() timed out异常并退出进程。所以我们在实现finalize方法的时候一定不能在finalize方法内做太过负责的事情。另外从这里也看出，如果对象实现了finalize方法，那么它的内存会等到其finalize方法执行完成才真正释放，这从某种程度上说也推迟啦GC回收内存的进度。所以不是万不得已个人是不建议实现finalize方法的。

以上就是我对Android中的Reference的学习过程。希望能对朋友们有所帮助。感谢您能读到这里，有各种意见欢迎指出讨论。
