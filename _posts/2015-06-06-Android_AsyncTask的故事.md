> 写在前面：
AsyncTask不用多介绍，今天不说怎样使用，我带大家看看AsyncTask的进化史，希望大家能从中有所收获。顺便问一句：你认为你应用中实例化多个AsyncTask去execute，这些AsyncTask都在高效的并发运行吗？

在很久很久以前（2.3以前）

一群可爱的程序猿发现了一个叫做AsyncTask的东西，觉得它很好用，比起Thread来方便多了。于是AsyncTask一夜间红遍五大洲四大洋。可是用着用着，一个细心的程序猿（比如说我）发现了一个问题。在应用中使用了5个AsyncTask，并都调用了execute，当我再构造一个AsyncTask并execute时，第六个AsyncTask并没有执行（至少没有马上执行）。why? Read The Fucking Source Code.

让我们看看2.3时AsyncTask的成员变量有些什么：

**AsyncTask.java**
```
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";

    private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final BlockingQueue<Runnable> sWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,
            MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;
    private static final int MESSAGE_POST_CANCEL = 0x3;

    private static final InternalHandler sHandler = new InternalHandler();

    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;

    private volatile Status mStatus = Status.PENDING;

    /**
     * Indicates the current status of the task. Each status will be set only once
     * during the lifetime of a task.
     */
    public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         */
        PENDING,
        /**
         * Indicates that the task is running.
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         */
        FINISHED,
    }
}
```

非常简单，我们主要关注一下AsyncTask中真正执行任务的线程池。

```
private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,
            MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);

/*参数解释：
CORE_POOL_SIZE 表示线程池中的核心线程数量，可理解为线程池中的最少线程数或者线程池中的不死线程（当然也可以配置保活时间，这里我们不考虑）。
MAXIMUM_POOL_SIZE 表示线程池中最多能容纳的线程数量。
KEEP_ALIVE 保活时间，指的是线程池中的超过CORE_POOL_SIZE定义的数量的线程在执行完任务处于空闲时保活的时间，超过这个时间，线程将回收。
TimeUnit.SECONDS 保活时间的单位。
sWorkQueue 线程池中的任务队列。（此处是一个阻塞队列）
sThreadFactory 用于线程池生成线程的线程工厂。
ThreadPoolExecutor的构造方法还有一个重载形式是多了一个超过线程池负载时处理策略的参数的。这里默认的处理测略是抛出java.util.concurrent.RejectedExecutionException异常。

顺便介绍一下线程池的工作流程，当调用线程池的execute方法执行一个任务时，假设线程池中已经存在的线程数量为Tn,线程池的任务队列sWorkQueue中保存的任务数量为Wn,sWorkQueue的size为Wsize,有以下几种情况：
1.当Tn < CORE_POOL_SIZE时，不管此时线程池中已存在的线程是否空闲，都会新建一个线程去执行任务。因为线程池要保证它拥有的线程至少有CORE_POOL_SIZE个。
2.当CORE_POOL_SIZE < Tn < MAXIMUM_POOL_SIZE且Wn < Wsize时，线程池只会把要执行的任务放到sWorkQueue中。并不会新建一个线程去执行当前的任务。
3.当CORE_POOL_SIZE < Tn < MAXIMUM_POOL_SIZE且Wn = Wsize时，线程池才会新建一个线程去执行任务。
4.当Tn = MAXIMUM_POOL_SIZE且Wn = Wsize时，通过上面说的构造线程池时传入的表示超过线程池负载时处理策略的参数去完成处理，默认是抛出异常java.util.concurrent.RejectedExecutionException。

最重要的是这里的线程池的static的。那意味着一个进程空间中所有的AsyncTask类的实例都将共用这一个线程池！！！
*/
```

我把关于线程池参数的解释和线程池分配任务的策略在上面的注释里写的已经很清楚了，一定要仔细看完注释，看完后再分析AsyncTask时就豁然开朗了。

了解了线程池的相关知识后我们先看一下这个版本的AsyncTask的构造方法：

**AsyncTask.java**
```
public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                return doInBackground(mParams);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                Message message;
                Result result = null;

                try {
                    result = get();
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occured while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    message = sHandler.obtainMessage(MESSAGE_POST_CANCEL,
                            new AsyncTaskResult<Result>(AsyncTask.this, (Result[]) null));
                    message.sendToTarget();
                    return;
                } catch (Throwable t) {
                    throw new RuntimeException("An error occured while executing "
                            + "doInBackground()", t);
                }

                message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                        new AsyncTaskResult<Result>(AsyncTask.this, result));
                message.sendToTarget();
            }
        };
}
```

构造方法比较简单，主要是生成了一个待执行的任务。这里的任务是通过WorkerRunnable封装的，WorkerRunnable实现了Callable接口。Callable是什么？你可以大致把理解为Callable + FutureTask = Runnable，同Runnable一样它也是描述一个要执行的任务，Runnable跑run方法，而Callable跑call方法。只是它提供了获取执行结果的回调处理，这一部分就是通过FutureTask来完成的，所以你只要粗略理解Callable + FutureTask = Runnable就可以了。在call方法中去跑一下doInBackground()（哇，多么熟悉的名字）。

任务生成了，让一切跑起来，看看execute方法做了什么：

**AsyncTask.java**
```
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        sExecutor.execute(mFuture);

        return this;
}
```

首先check一下状态，接着跑一下onPreExecute()回调，把参数赋值给我们构造方法中创建的任务后，一切就交由线程池去搞定了。

好了，那么问题来了。在2.3的AsyncTask中CORE_POOL_SIZE为5，MAXIMUM_POOL_SIZE为128，sWorkQueue长度为10，前面说已经跑了5个AsyncTask，他们都是共用线程池的，那么这5个任务如果都很耗时，因为5 < 6 < 128并且此时线程池任务队列是空的，第6个任务必然被阻塞，阻塞是因为根本没有线程去执行它，它只会被独自扔在线程池的任务队列中，一个人慢慢老去...

还有一个隐患，如果线程某个时刻业务量很大，线程池中的线程个数已满，达到MAXIMUM_POOL_SIZE，并且任务队列也满了，此时再往线程池扔任务，必然只能通过超出负载处理策略来处理，而这里默认抛出java.util.concurrent.RejectedExecutionException，这时候如果我们没有捕获该异常。那就等着用户卸载吧...

当然，这是发生在很久很久以前的故事。优化是永不停歇的话题。所以

终于有一天（4.0准确说应该是3.0吧）

可爱的程序猿们发现AsyncTask变了：

**AsyncTask.java**
```
public abstract class AsyncTask<Params, Progress, Result> {
    private static final String LOG_TAG = "AsyncTask";

    private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    private static final InternalHandler sHandler = new InternalHandler();

    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;

    private volatile Status mStatus = Status.PENDING;

    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

}
```

主要的变化在于：

1.多了一个SerialExecutor,并且将该执行器作为默认的AsyncTask的执行器。

2.开放了一个executeOnExecutor接口，我们其实可以通过这个接口定制适用于自己应用的线程池来完成默认线程池的工作。

3.将2.3中负责执行任务的线程池public了。

先来看SerialExecutor，它定义了一个mTasks存放需要执行的任务，它其实就是一个跑在调用execute的线程的任务分发器，串行的将任务加入到mTasks中，不过其最终还是使用老的THREAD_POOL_EXECUTOR来执行任务，但此时THREAD_POOL_EXECUTOR其实只有一个线程在工作，类似一个生产者消费者模型，SerialExecutor此时充当一个生产者，将任务往mTasks里面塞，THREAD_POOL_EXECUTOR类似一个消费者，往mTasks里面拿任务并执行。但只有当第一次执行任务或任务执行完时才会去mTasks中取下一个任务，即此时虽然是一个线程池在工作，但同时运行的只有一个任务。

上面说的SerialExecutor是AsyncTask默认的Executor。比起2.3时代的直接用线程池跑任务。这种策略不会把线程池塞爆，不会出现任务在线程池的任务队列中等待终老的情况，但是此时任务串行执行，即同一个进程中的AsyncTask都是串行执行的，下一个AsyncTask只能在上一个AsyncTask执行完后才会开始执行。也许这里本来就是个嘈点，于是大神们干脆给了executeOnExecutor接口，言外之意即是“默认的不满意你们爱咋整自己整去吧”。看下4.0中AsyncTask的execute方法你就明白了：
**AsyncTask.java**
```
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
}
```

execute通过executeOnExecutor来完成工作的，传入一个执行器作为真正完成工作的Executor。此时传入的就是SERIAL_EXECUTOR。

这下真相大白了，如果你觉得默认的SERIAL_EXECUTOR嘈点太多。完全可以根据自身应用的业务逻辑自己定义符合应用策略的Executor。将其作为参数在执行AsyncTask时通过executeOnExecutor方法来执行任务就可以了。

将原有的负责执行任务的线程池public了，即THREAD_POOL_EXECUTOR可以在应用的任何地方通过AsyncTask类名来获取。所以这里依旧存在一个隐患，如果大家在自己应用的代码中非常非常的滥用AsyncTask.THREAD_POOL_EXECUTOR这个线程池，那么依然会由于前面分析的线程池的执行策略分配任务的逻辑出现任务不执行或者java.util.concurrent.RejectedExecutionException崩溃等问题。

如果你能看到这里，希望没有让你感觉浪费了几分钟。阅读源码是很好的学习方式，不知你从一个小小的AsyncTask的变迁过程有没有感觉到跟Google大牛有神交的感觉。

总结一下其实只要大致理解线程池的分配任务的逻辑。AsyncTask的缺点就暴露无疑了（当然暇不掩瑜）。

开头的提问现在可以给个明确的答复了，你认为你应用中实例化多个AsyncTask去execute，这些AsyncTask都在高效的并发运行吗？

在2.3前时代，有两种情况下任务是在并发执行的：

1.当线程池中的线程数量少于CORE_POOL_SIZE。

2.当线程池中的线程数量大于CORE_POOL_SIZE且小于MAXIMUM_POOL_SIZE，并且线程池任务队列已满时。

在2.3后时代，如果你使用默认的Executor，任务永远是串行执行的。不过可以自己定义Executor。

本文可能有地方描述不准确，欢迎大家提出指正，共同学习，感激不尽～