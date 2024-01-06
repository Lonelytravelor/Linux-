# 应用启动

> https://juejin.cn/post/7261599630827765816

## 应用启动时间

### 初始显示时间（TTID）

初始显示时间 （TTID，The Time to Initial Display） ，它是从系统接收到启动意图到应用程序显示第一帧界面的时间，也就是用户看到应用程序界面的时间。

### 测量TTID

1. **方法一:通过logcat命令进行查看**

当应用程序完成上面提到的所有工作时，可以在logcat中看到以下的日志输出。

```bash
/system_process I/ActivityTaskManager: Displayed xxxx/.MainActivity: +401ms
```

在所有资源完全加载并显示之前，Logcat输出中的Displayed时间指标，省去了布局文件中未引用的资源或应用作为对象初始化一部分创建资源的时间。

有的时候logcat输出中的日志中会包含一个附加字段total。如下所示：

```bash
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms (total +1m22s643ms)
```

在这种情况下，第一个时间测量值仅针对第一个绘制的 activity。`total` 时间测量值是从应用进程启动时开始计算，并且可以包含首次启动但未在屏幕上显示任何内容的另一个 activity。`total` 时间测量值仅在单个activity的时间和总启动时间之间存在差异时才会显示。

一些博客中介绍的使用`am start -S -W`命令来测得的时间，实际上就是初始显示时间。

2. **方法二:通过Perfetto进行查看**

通过查看主线程最后一个ActivityResume后面的渲染帧;

### 完全显示的时间（TTFD）

大多数情况下，初始显示时间并不能代表应用真正的启动时间，例如，应用启动时需要从网络上同步最新的数据，当所有的数据加载完毕后的耗时才是真正的启动耗时，这个就是接下来要介绍的TTFD - **完全显示时间**。

完全显示时间 （TTFD，The Time to Full Display） 。它是从系统接收到启动意图到应用程序加载完成所有资源和视图层次结构的时间，也就是用户可以真正使用应用程序的时间。

### 测量TTFD

1. **方法一：调用`reportFullyDrawn()`**

`reportFullyDrawn()`方法可以在应用程序加载完成所有资源和视图层次结构后调用，让系统知道应用程序已经完全显示，从而计算出完全显示时间。如果不调用这个方法，系统只能计算出TTID而无法得知TTFD。

```bash
system_process I/ActivityTaskManager: Fully drawn xxxx/.MainActivity: +1s54ms
```

2. **方法二：拆帧**

拆帧法是目前计算车载应用启动耗时时最普遍的做法，拆帧法有许多不同的录制、拆帧方式。

常见的有，使用支持60fps的摄像机（支持60fps摄像的手机也可以）拍摄应用的启动视频，再使用`Potplay`视频播放器查看从**桌面点击**到**应用画面完全显示出来**的帧数差值，然后除以60就可以得到应用的启动耗时。

3. **方法三:FFmpeg拆帧**

以上方法适合测试人员使用，这里介绍另一种更适合开发人员操作的方式：FFmpeg拆帧。

> FFmpeg 的下载地址：[ffmpeg.org/download.ht…](https://link.juejin.cn?target=http%3A%2F%2Fffmpeg.org%2Fdownload.html%3Faemtn%3Dtg-on)

首先使用adb连接Android设备，使用录屏指令录制应用启动时的视频。

```bash
adb shell screenrecord /sdcard/launch.mp4
```

使用FFmpeg查看视频的帧数,具体请看官方文档或者上述引用.

## 应用启动时间实战分析

### 启动时触发GC

**现象**：**slow_start_reason**中出现"GC Activity"，表示在启动阶段GC活动拖慢了应用程序的启动。

**分析**：点击【show timeline】返回到Perfetto时间轴界面。在启动时间轴中，可以看到有一个线程名为**HeapTaskDaemon**，他就是应用程序的GC线程，在启动阶段活跃了约100ms左右，导致activityResume的时间轴也被拉长100ms。为防止是偶发现现象，进行了多次测量，发现该应用启动时必定会触发GC活动。如图所示：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9472ff1f19c4e47a3902e0f8797e113~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**原因**：根据时间轴向前分析，发现该应用在启动阶段会加载一个特殊字体，该字体约13MB，经过与应用的开发沟通，确认该字体已经移动到系统层，应用层不必加载该字体。移除字体后，应用启动时不再100%触发GC。

### 主线程耗时操作

**现象**：**slow_start_reason**出现 "Main Thread - Time spent in Running state(状态)"，表示启动阶段，主线程中执行较多的耗时操作。

**原因**：这种情况在应用开发时很常见，一些跨进程的获取数据的操作，应用开发人员会很自然的将其放在主线程Activity的OnCreate或onStart方法下执行，这些IPC方法虽然不至于触发ANR，但是会拖慢应用的启动，应该放置到线程池或协程中执行。

### OpenDexFilesFromOat耗时

**现象**：**slow_start_reason**出现"Main Thread - Time spent in OpenDexFilesFromOat*"，表示启动阶段，花费了较多时间在读取dex文件上。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bafa33f4dd8b4ef3bdd3dec8677286ff~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**原因**：这种情况在车载Android系统中较为常见。这可能是因为系统为了加快启动速度，修改了系统中dex2oat的流程，导致此现象，耗时不多的话可以忽略。

> 这里只是说是“可能”，是因为如今的车载OS为了快速启动，对原生Android的修改非常多，我们需要结合自身的实际情况再做详细分析。

### 连续多帧绘制超时

**现象：** 某个应用在Perfetto中的启动时间并不长，大约在1.3s左右，但是使用拆帧法后，发现该应用启动后会有些许卡顿，导致实际启动时间拉长到2.1s。表现在Perfetto上如下所示，在第一帧绘制完毕后，后续2、4、5帧的绘制时间都超过了150ms。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e11b18d004d14aad9c0d6932a819fa3e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**分析**：Perfetto给出的帧绘制时间轴显示大部分时间花费在View的Layout上，这说明在首帧绘制完毕后，又触发了多次页面重绘。

**原因**：通过代码结合应用的日志，发现该应用在启动时会使用空数据刷新一次页面，然后会从IPC接口中再获取一次数据更新页面，而且由于代码缺陷，数据刷新会连续执行4次，导致了该情况。修改缺陷代码，首帧之后就不会再发生连续绘制超时的情况了。