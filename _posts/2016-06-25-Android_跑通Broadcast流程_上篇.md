> 写在前面：
Broadcast即广播，是Android四大组件之一，Android提供了方便的接口让开发者使用广播机制，本文不细说如何使用，主要依据源码和大家一起一窥广播相关框架的整个实现流程，顺便对自己的的学习做个记录。该系列文章计划分两篇介绍，上篇介绍广播的注册流程，下篇介绍广播的发送流程。

-------------------
## 注册方式
**静态注册** 在应用AndroidManifest.xml文件中配置receiver

**动态注册** 在代码中通过Context提供的registerReceiver方法注册

下面就分别对两种注册方式的实现做详细介绍。


## 静态注册

一个apk在被安装时，PMS会通过解析apk中的AndroidManifest.xml文件将各个标签组装成对应的数据结构保存在PMS内部。
**PackageManagerService.java**

```
public class PackageManagerService extends IPackageManager.Stub {
	
	...
	
    // this lock held have the prefix "LP".
    @GuardedBy("mPackages")
    final ArrayMap<String, PackageParser.Package> mPackages =
            new ArrayMap<String, PackageParser.Package>();
	 
    // All available activities, for your resolving pleasure.
    final ActivityIntentResolver mActivities =
            new ActivityIntentResolver();

    // All available receivers, for your resolving pleasure.
    final ActivityIntentResolver mReceivers =
            new ActivityIntentResolver();

    // All available services, for your resolving pleasure.
    final ServiceIntentResolver mServices = new ServiceIntentResolver();

    // All available providers, for your resolving pleasure.
    final ProviderIntentResolver mProviders = new ProviderIntentResolver();

...
}
```
PMS中有很多用于保存各个应用的四大组件的容器，并且还有一个mPackages容器保存了每个已安装的应用的信息结构。PMS中的这个数据结构不是很复杂，但是关系比较纷繁，感兴趣的小伙伴可以自己研究PMS的代码，这里就不过多的叙述了，我们这里主要注意上面代码中列出的mReceivers容器，在AndroidManifest.xml里面静态注册的广播会在安装应用解析时保存到这个容器中。


## 动态注册

动态注册主要通过Context提供的registerReceiver一系列的方法完成，它们最终都是通过registerReceiverInternal方法来实现的，该方法的实现在ContextImpl中。一起来看下代码：
**ContextImpl.java**

```
class ContextImpl extends Context {

...
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }

...

}

```

这个方法主要完成两件事情：
1.构造一个IIntentReceiver类型的对象rd。
2.将该对象传到AMS中，开始广播注册的AMS之旅。
在注册一个BroadcastReceiver时会先构造一个ReceiverDispatcher对象，一个ReceiverDispatcher内部持有应用自己定义的BroadcastReceiver,同时还持有一个IIntentReceiver类型的成员变量，可以通过getIIntentReceiver将其内部的IIntentReceiver获取出来，这个BroadcastDispatcher中的BroadcastReceiver和IIntentReceiver其实都对应同一个广播接收器，只不过前者用于在应用层表示，后者用于在AMS系统层表示。并且AMS和应用进程间的通信也可以由IIntentReceiver来完成。也就是说，一个BroadcastReceiver对应一个BroadcastDispatcher对应一个IIntentReceiver，当应用注册一个广播接收器，这个广播接收器在应用端和系统端会有两种不同的表现形式，这两种不同的表现形式通过一个BroadcastDispatcher来串联起来。
当系统发送一个广播时，在系统进程中会找到对该广播感兴趣的IIntentReceiver并通过它的接口跨进程与对应的应用进程通信，这时候在代码逻辑就会跑到应用进程中的BroadcastDispatcher中的IIntentReceiver来，在这里通过BroadcastDispatcher将广播最终分发给应用层实现的BroadcastReceiver。广播的发送过程在下篇中会着重介绍，下面我们就进到AMS中去，看看在AMS中是如何处理广播注册的。

这里有兄弟可能会疑惑ActivityManagerNative.getDefault()得到的是什么对象，它又是怎么跑到AMS里面去的，这里主要是通过binder实现的跨进程通信，Android中特别源码中铺天盖地的类似玩意儿，有意或的兄弟可以先补一下binder相关的知识，这里我就直接跳过中间过程，一步到位直接进入AMS：

**ActivityManagerService.java**

```
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

...

    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        
        //1.安全检查

        enforceNotIsolatedCaller("registerReceiver");
        ArrayList<Intent> stickyIntents = null;
        ProcessRecord callerApp = null;
        int callingUid;
        int callingPid;
        synchronized(this) {
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when registering receiver " + receiver);
                }
                if (callerApp.info.uid != Process.SYSTEM_UID &&
                        !callerApp.pkgList.containsKey(callerPackage) &&
                        !"android".equals(callerPackage)) {
                    throw new SecurityException("Given caller package " + callerPackage
                            + " is not running in process " + callerApp);
                }
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            userId = handleIncomingUser(callingPid, callingUid, userId,
                    true, ALLOW_FULL_ONLY, "registerReceiver", callerPackage);

            //2.遍历系统中保存的已发送的sticky广播。

            Iterator<String> actions = filter.actionsIterator();
            if (actions == null) {
                ArrayList<String> noAction = new ArrayList<String>(1);
                noAction.add(null);
                actions = noAction.iterator();
            }

            // Collect stickies of users
            int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
            while (actions.hasNext()) {
                String action = actions.next();
                for (int id : userIds) {
                    ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                    if (stickies != null) {
                        ArrayList<Intent> intents = stickies.get(action);
                        if (intents != null) {
                            if (stickyIntents == null) {
                                stickyIntents = new ArrayList<Intent>();
                            }
                            stickyIntents.addAll(intents);
                        }
                    }
                }
            }
        }

        ArrayList<Intent> allSticky = null;
        if (stickyIntents != null) {
            final ContentResolver resolver = mContext.getContentResolver();
            // Look for any matching sticky broadcasts...
            for (int i = 0, N = stickyIntents.size(); i < N; i++) {
                Intent intent = stickyIntents.get(i);
                // If intent has scheme "content", it will need to acccess
                // provider that needs to lock mProviderMap in ActivityThread
                // and also it may need to wait application response, so we
                // cannot lock ActivityManagerService here.
                if (filter.match(resolver, intent, true, TAG) >= 0) {
                    if (allSticky == null) {
                        allSticky = new ArrayList<Intent>();
                    }
                    allSticky.add(intent);
                }
            }
        }

        // The first sticky in the list is returned directly back to the client.
        Intent sticky = allSticky != null ? allSticky.get(0) : null;
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Register receiver " + filter + ": " + sticky);
        if (receiver == null) {
            return sticky;
        }

        //3.将广播接收器注册到系统中

        synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) {
                // Original caller already died
                return null;
            }
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid != callingUid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for uid " + callingUid
                        + " was previously registered for uid " + rl.uid);
            } else if (rl.pid != callingPid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for pid " + callingPid
                        + " was previously registered for pid " + rl.pid);
            } else if (rl.userId != userId) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for user " + userId
                        + " was previously registered for user " + rl.userId);
            }
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadcast");
            }
            mReceiverResolver.addFilter(bf);

            //4.处理匹配的sticky广播

            // Enqueue broadcasts for all existing stickies that match
            // this filter.
            if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                final int stickyCount = allSticky.size();
                for (int i = 0; i < stickyCount; i++) {
                    Intent intent = allSticky.get(i);
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                            null, 0, null, null, false, true, true, -1);
                    queue.enqueueParallelBroadcastLocked(r);
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }

...

}

```

这段代码比较长，我在代码里面已经加了分段注释，我们分段来阅读：
1.安全检查。
这部分代码主要完成检查注册广播接收器的进程的合法性逻辑。若检查不通过会抛出异常。
2.遍历系统中保存的已发送的sticky广播。
sticky广播，即粘性广播。这种广播发出后会一直滞留（等待），以便有人注册这则广播后能尽快的收到这条广播。其他功能与sendBroadcast相同。但是使用sendStickyBroadcast 发送广播需要获得BROADCAST_STICKY permission，如果没有这个permission则会抛出异常，这在下篇介绍广播发送时会详细介绍。AMS中的mStickyBroadcasts中保存了系统中滞留（等待）的广播，所以这里遍历这个容器，并找到与当前注册的广播接收器的filter的action相匹配的广播Intent。最终将与当前正在注册的广播接收器的filter完全匹配的sticky广播放入到allSticky列表中。准备完allSticky列表之后，会判断如果当前注册的广播接收器为null，则直接将当前allSticky中的第一个滞留广播返回给客户端调用者。
3.将广播接收器注册到系统中。
到这里，才开始真正的广播接收器的注册工作，其实说白了注册就是把要注册的广播接收器保存在AMS相应的数据结构列表中，方便系统发送一个广播时能够在注册表中找到需要接收该广播的接收器。在AMS中与广播接收器的注册相关的容器有两个，一个是mReceiverResolver，它内部定义了很多以不同维度来保存当前注册的广播的filter的数据结构，还有一个是mRegisteredReceivers。mRegisteredReceivers是个映射表结构，以IIntentReceiver.aasBinder()为key，一个ReceiverList为value。前面说了一个IIntentReceiver和一个BroadcastReceiver是对应的，所以也就是说mRegisteredReceivers中的key其实就是注册的广播接收器。mReceiverResolver是一个IntentResolver类型的对象，在往mRegisteredReceivers中添加完当前正在注册的广播接收器后，会构造一个BroadcastFilter对象，这个对象描述了一个广播接收器能够接收的广播范围，之后通过调用mReceiverResolver.addFilter方法并传入这个BroadcastFilter对象来实现在mReceiverResolver中以不同的维度保存了当前注册的广播接收器的filter，当发送一个广播时，正是通过在mReceiverResolver中查找匹配的intent来完成对动态注册广播的查找的。
4.处理匹配的sticky广播。
在第二步中已经得到了与当前注册的广播接收器的filter匹配的sticky广播，并保存在allSticky列表中，在这一部要做的就是将这些广播发送出去，因为allSticky中的广播是与当前注册的广播接收器匹配的sticky广播，所以当前广播接收器注册完毕后需要将这些广播及时的发送出去以便当前广播接收器能够接收到这些sticky广播。广播的发送时通过构造一个BroadcastQueue并调用其scheduleBroadcastsLocked();方法触发的，这里就不多涉及广播发送的流程，在下篇的我将详细介绍。

到这里在AMS中广播接收器的注册工作也完成了，这其中我只是将其流程大致的串讲一边，但是其中的数据结构还是比较复杂的，还涉及Intent的解析与匹配，建议如果感兴趣的兄弟还是一定要自己通读一遍相关的源代码。源代码才是最好的老师。另外如果有什么说的不对的地方欢迎大家指正，多谢~
