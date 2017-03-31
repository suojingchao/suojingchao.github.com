> 本文对最近实现Service时遇到的一个小问题做一个记录。
> 主要以一个简单的demo讨论在bindService时，client和Service分处不同的进程，bindService传入的Flag分别对client进程和service进程的oom_adj值有什么影响。


我写了两个应用，**A应用（com.test.serviceadjdemo_client）**作为client存在，就一个按钮，点击按钮后调用bindService绑定B应用的一个服务。**B应用（com.test.serviceadjdemo_service）**也很简单，作为service存在，实现了被绑定的服务。下面看看当bindService传入不同的Flag时两个应用的oom_adj值有什么不同：

## BIND_AUTO_CREATE
通过**adb shell dumpsys meminfo**查看进程的oom_adj：
**启动A**

```
    20094 kB: Foreground
               20094 kB: com.test.serviceadjdemo_client (pid 13687 / activities)
```
这里看到A应用的oom_adj理所应当是FOREGROUND_APP_ADJ。

**点击按钮后通过bindService拉起B** 

```
    19430 kB: Foreground
               19430 kB: com.test.serviceadjdemo_client (pid 13687 / activities)
               
   331549 kB: Visible
				...
                5284 kB: com.test.serviceadjdemo_service (pid 14357)
                ...
```

这时候看到B应用被bindService拉起来了，并且B的oom_adj为VISIBLE_APP_ADJ，仅次于FOREGROUND_APP_ADJ。

## BIND_AUTO_CREATE | BIND_ABOVE_CLIENT
**启动A**

```
        20331 kB: Foreground
               20331 kB: com.test.serviceadjdemo_client (pid 14631 / activities)
```
这里看到A应用的oom_adj理所应当是FOREGROUND_APP_ADJ。

**点击按钮后通过bindService拉起B** 

```
     5324 kB: Foreground
                5324 kB: com.test.serviceadjdemo_service (pid 15078)

   332218 kB: Visible
			   ...
               19479 kB: com.test.serviceadjdemo_client (pid 14631 / activities)
               ...
```
这里看到神奇的事情就开始发生了，被拉起的B应用的oom_adj变成了FOREGROUND_APP_ADJ，在前台显示的A进程的oom_adj反而变成了VISIBLE_APP_ADJ。
## BIND_AUTO_CREATE | BIND_IMPORTANT

**启动A**

```
            20255 kB: Foreground
               20255 kB: com.test.serviceadjdemo_client (pid 15849 / activities)
```
这里看到A应用的oom_adj理所应当是FOREGROUND_APP_ADJ。

**点击按钮后通过bindService拉起B** 

```
    25441 kB: Foreground
               20096 kB: com.test.serviceadjdemo_client (pid 15849 / activities)
                5345 kB: com.test.serviceadjdemo_service (pid 15884)
```
这里看到神奇的事情又发生了，被拉起的B应用的oom_adj此时和A应用一样，都是FOREGROUND_APP_ADJ。
## BIND_AUTO_CREATE | BIND_IMPORTANT | BIND_WAIVE_PRIORITY
**启动A**

```
    20226 kB: Foreground
               20226 kB: com.test.serviceadjdemo_client (pid 16372 / activities)
```
这里看到A应用的oom_adj理所应当是FOREGROUND_APP_ADJ。

**点击按钮后通过bindService拉起B** 

```
    20023 kB: Foreground
               20023 kB: com.test.serviceadjdemo_client (pid 16372 / activities)

   332744 kB: Cached
                ...
                5400 kB: com.test.serviceadjdemo_service (pid 16403)
                ...
```
这里看到加入了BIND_WAIVE_PRIORITY后，BIND_IMPORTANT就失效了，新拉起来的B应用进程的oom_adj>=CACHED_APP_MIN_ADJ将被当做background进程去管理。

当组合为**BIND_AUTO_CREATE | BIND_ABOVE_CLIENT | BIND_WAIVE_PRIORITY**时其实效果跟上面一组一样，系统一样会把拉起来的B进程的oom_adj设置为>=CACHED_APP_MIN_ADJ，并且将B当做一个background进程去管理。

## 总结
- 可以通过BIND_ABOVE_CLIENT或者BIND_IMPORTANT来提升Service所在进程的优先级。
- 可以通过BIND_WAIVE_PRIORITY来放弃对Service所在进程的优先级的提升，当设置该Flag后，Service所在进程的管理完全交由系统当做background进程管理。

本文刚好作为一个引子，引出后面的oom_adj系列的文章。在后面oom_adj系列的记录文章中，我将从：
* 如何计算oom_adj
* 如何使用oom_adj
* lmk实现
三个角度去详细介绍oom_adj在android中扮演的角色。


谢谢大侠们赏脸看完，有理解偏差的地方多多指正。


