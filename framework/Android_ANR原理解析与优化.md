# Android 15 ANR 原理、排查与优化指南

## 1. 宏观视角：什么是 ANR
在 Android 操作系统中，应用必须对用户的交互保持敏捷的响应。如果应用程序的主线程（UI 线程）在特定的时间段内未能响应系统或用户的操作，系统就会抛出一个名为 **ANR (Application Not Responding，即应用无响应)** 的错误。

### 为什么系统要设计 ANR 机制？
为了保证绝佳的用户体验。如果一个应用因为死锁、死循环或者高负载的 I/O 操作导致 UI 线程卡死，用户点击屏幕将毫无反应。Android 作为一个现代操作系统，为了防止“一个劣质应用卡死整个手机”，强制引入了 ANR 机制。它像一个系统的看门狗（Watchdog），时刻监控着每个应用的主线程健康状态。

### ANR 的直接表现
当触发 ANR 时，系统通常会：
1. **弹出系统对话框**：提示用户“XXX 应用无响应”，并提供“关闭应用”或“等待”的选项。
2. **冻结或闪退**：如果用户选择关闭，或者在较新的后台静默 ANR 机制中，应用可能会直接被系统杀死（闪退）。
3. **现场保存**：系统会将当前的进程堆栈、CPU 占用、I/O 状态记录下来，存入 DropBox 并在 `/data/anr/` 目录下生成著名的 `traces.txt` 文件，供开发者排查。

---

## 2. 核心成因与场景

ANR 的本质是**主线程在规定时间内未能完成系统分发的任务**。在 Android 15 源码中，系统对不同的组件设置了明确的超时阈值。这些阈值主要定义在 `ActivityManagerService.java` (AMS) 和 `InputConstants` 中。

主要的 ANR 触发场景及超时参数如下：

1. **InputDispatching Timeout (输入事件分发超时)**
   - **阈值**：**5 秒**
   - **场景**：用户触摸屏幕（如点击按钮）或按下实体按键，但主线程由于正在执行耗时操作，超过 5 秒未能处理该输入事件。
   - **源码参考** (`IInputConstants.aidl` / `InputConstants.java` / `InputDispatcher.cpp`)：
     `const int UNMULTIPLIED_DEFAULT_DISPATCHING_TIMEOUT_MILLIS = 5000; // 5 seconds`

2. **Broadcast Timeout (广播处理超时)**
   - **阈值**：**前台广播 10 秒 / 后台广播 60 秒**
   - **场景**：BroadcastReceiver 的 `onReceive()` 方法执行耗时过长。
   - **源码参考** (`ActivityManagerService.java`)：
     ```java
     static final int BROADCAST_FG_TIMEOUT = 10 * 1000 * Build.HW_TIMEOUT_MULTIPLIER;
     static final int BROADCAST_BG_TIMEOUT = 60 * 1000 * Build.HW_TIMEOUT_MULTIPLIER;
     ```

3. **Service Timeout (服务处理超时)**
   - **阈值**：**前台服务 20 秒 / 后台服务 200 秒** (注: 实际 Background 常量为 `SERVICE_TIMEOUT * 10`)
   - **场景**：Service 的各个生命周期函数（如 `onCreate()`, `onStartCommand()`, `onBind()`）在主线程执行耗时过长。
   - **源码参考** (`ActivityManagerConstants.java`)：
     ```java
     private static final long DEFAULT_SERVICE_TIMEOUT = 20 * 1000 * Build.HW_TIMEOUT_MULTIPLIER;
     private static final long DEFAULT_SERVICE_BACKGROUND_TIMEOUT = DEFAULT_SERVICE_TIMEOUT * 10;
     ```

4. **ContentProvider Timeout (ContentProvider 发布超时)**
   - **阈值**：**10 秒**
   - **场景**：应用进程启动时，其 `ContentProvider` 的 `onCreate()` 未能在规定时间内执行完毕（发布到 AMS）。
   - **源码参考** (`ContentResolver.java` / `ActivityManagerService.java`)：
     `public static final int CONTENT_PROVIDER_PUBLISH_TIMEOUT_MILLIS = 10000;`

*(注：部分厂商设备上的 `Build.HW_TIMEOUT_MULTIPLIER` 可能大于 1，从而延长这些阈值)*

---

## 3. 微观视角：AOSP (Android 15) 源码解析

Android 系统是如何精准探测超时，并收集 ANR 现场的呢？这依赖于一套完善的**埋雷与拆雷机制（看门狗机制）**。

### 3.1 超时监控机制 (Watchdog & Handler)

在 AMS (`ActivityManagerService`) 以及底层系统中，超时机制通常这样工作：
1. **埋雷**：当系统（如 AMS）调度一个组件生命周期或下发输入事件时，会向系统 Handler 发送一个延迟消息（延时时间即为超时阈值）。
2. **拆雷**：如果应用主线程在规定时间内完成了任务，系统就会调用相应的完成回调，把之前发送的延迟消息撤销（Remove）。
3. **爆炸**：如果应用主线程卡住，未能及时回调，延迟消息就会在 Handler 中触发。系统判定 ANR 发生。

#### Service 超时探测 (`ActiveServices.java`)
当启动 Service 时，系统调度 `SERVICE_TIMEOUT_MSG` 延迟消息：
```java
Message msg = mAm.mHandler.obtainMessage(
        ActivityManagerService.SERVICE_TIMEOUT_MSG, "SERVICE_TIMEOUT");
mAm.mHandler.sendMessageDelayed(msg,
        proc.mExecState.isCurProcessGroupOomAdj() ? mAm.mConstants.SERVICE_TIMEOUT : mAm.mConstants.SERVICE_BACKGROUND_TIMEOUT);
```
若主线程完成了 Service 生命周期并上报，AMS 会取消这个 MSG。如果没取消，触发超时并调用 ANR 收集流程。

#### Input 事件超时探测 (`InputDispatcher.cpp`)
输入系统的检测略有不同，它涉及到跨进程通信。底层 `InputDispatcher` 通过 Socket 向 App 发送 Input 事件。`InputDispatcher` 中维护了一个队列和定时器，当发送事件后开始倒计时（默认 `DISPATCHING_TIMEOUT` 5 秒）。如果 App 的 UI 线程卡住，未能通过 Socket 回复 `Finish` 信号，`InputDispatcher` 就会报告输入超时，回调给上层的 `InputManagerService`，最终触发 ANR。

### 3.2 ANR 触发与收集流程 (`AnrHelper.java` & `AppErrors.java`)
当确认 ANR 发生后，整个收集现场的过程是极其宏大且复杂的：

1. **进入收集入口**：由 `AppErrors.appNotResponding()` 承接。
2. **异步处理机制**：Android 15 中，为了防止 ANR 收集过程自身卡死 AMS，引入了 `AnrHelper.java`。请求会被封装成一个任务扔进专门的线程处理。
3. **收集系统状态**：记录发生 ANR 瞬间的 CPU 使用率 (`Load average`)，内存状态，I/O 负载等。这些信息对于排查“系统整体高负载导致的背锅型 ANR”非常关键。
4. **Dump 堆栈 (`ProcessRecord.java` 与 ART 交互)**：
   系统向目标应用进程（以及一些关键的系统进程和 HAL 进程）发送特殊的信号（早期是 `SIGQUIT` 即 Signal 3）。
   ART 虚拟机的 Signal Catcher 线程捕获到该信号后，会主动挂起所有线程，遍历所有线程的堆栈信息，并将其写入到 `/data/anr/traces.txt` 文件中。
5. **系统弹窗或直接杀死**：最后，AMS 根据系统设置和是否处于前台，决定是弹出 ANR 对话框，还是在后台默默干掉该进程。

---

## 4. 实战：ANR 排查方式

面对线上的 ANR 崩溃，我们需要一套标准的方法论来剥丝抽茧。

### 第一步：获取和解读 traces.txt / bugreport
**这是最核心的证据！** 拿到 `traces.txt`（或包含在 Bugreport 中的压缩包）后，搜索我们的应用包名，重点关注 `main` 线程的状态：

- `main` 处于 **RUNNABLE**：说明主线程正在执行代码。顺着堆栈往下看，通常是陷入了耗时的死循环，或者在执行极其复杂的运算。
- `main` 处于 **BLOCKED**：说明主线程在等待锁（如 `synchronized` 块）。注意看堆栈中提示的 `waiting to lock <0xXXXX>`，然后在整个 trace 文件中搜索 `<0xXXXX>`，找出究竟是哪个后台线程持有了这把锁，并在干什么耗时操作。
- `main` 处于 **WAITING / TIMED_WAITING**：通常在等待某种条件释放，或者调用了 `Thread.sleep()`, `Object.wait()`, 或者是等待某个 Binder 跨进程调用的返回。

### 第二步：分析系统侧环境 (Logcat & Event Log)
不要只看 App 堆栈，一定要结合 Logcat。
- **系统 CPU 状态**：ANR 日志开头会有 `CPU usage from XXX ms to XXX ms ago` 的记录。如果 `iowait` 极高（例如 50% 以上），说明系统 I/O 拥堵，你的 App 可能只是受害者（背锅）。如果 `user` 或 `kernel` CPU 占用 100%，说明有进程在疯狂计算抢占资源。
- **内存与 GC**：搜索 `GC_FOR_ALLOC` 或 `Alloc concurrent copying GC freed` 等字样。如果发生 ANR 前后有极高频且耗时的 GC 打印，说明是内存抖动导致的 STW（Stop-The-World），直接拖垮了主线程。

### 第三步：经典 ANR 案例分析
- **案例一：主线程 I/O**
  开发者在主线程直接调用了 `SharedPreferences.commit()` 或者进行了巨大的数据库 SQLite 查询。遇到磁盘慢的时候，分分钟卡住 5 秒引发 Input ANR。
- **案例二：主线程死锁**
  主线程需要刷新 UI，调用了被 `synchronized` 修饰的函数；而工作线程正在执行耗时的网络请求，且同样持有了该锁。主线程死等，导致 ANR。
- **案例三：系统整体高负载 (背锅型)**
  `traces.txt` 里主线程状态非常健康（比如正阻塞在正常的 `MessageQueue.next()`），但还是 ANR 了。这通常是因为当时整机 CPU 被其他流氓软件跑满，或者磁盘极度繁忙，导致你的应用分不到时间片。这种 ANR 很难由应用自身解决。

---

## 5. 高阶：ANR 优化与防范方案

解决 ANR 的核心思想就是：**主线程只做 UI 渲染和极轻量级的逻辑控制**，脏活累活全丢给子线程。

### 5.1 架构级防范
- **全面拥抱异步**：合理使用 Kotlin 协程（Coroutines）或线程池。在涉及到网络、文件读写、数据库操作时，强制切换到 `Dispatchers.IO`。
- **避免在生命周期做重操作**：绝不在 `Application.onCreate()` 或 `Activity.onCreate()` 中做大量同步初始化操作。应采用延迟初始化、异步初始化或使用 Jetpack App Startup。
- **合理拆分 Broadcast 和 Service**：由于前台 Broadcast 只有 10 秒时间，不要在 `onReceive()` 里做网络请求。可以通过 `goAsync()` 稍微延长时间，但最佳实践是将耗时任务转交给 `WorkManager` 或 `JobScheduler`。

### 5.2 监控建设
由于线上设备环境千奇百怪，线下很难复现所有的 ANR，因此必须建设监控系统：
- **线上 ANR 监控平台**：接入 Bugly、Firebase Crashlytics 等平台。较新的平台利用 Android 的 `ApplicationExitInfo` API，能非常精准地捕获和上报线上的 ANR 现场。
- **线下卡顿监控 (BlockCanary 思想)**：
  ANR 的前兆是卡顿。可以在主线程的 `Looper` 中设置一个自定义的 `Printer`（通过 `Looper.getMainLooper().setMessageLogging()`）。记录每一个 Message 执行前后的时间差，如果超过一定阈值（比如 200ms），就主动抓取一次堆栈并报警。这样能在演变成彻底的 ANR 之前，把隐患扼杀在摇篮里。
- **Choreographer 监控**：通过 `Choreographer.getInstance().postFrameCallback()` 监听每一帧的渲染耗时，检测掉帧情况。

### 总结
Android 15 中的 ANR 机制依然是捍卫用户体验的最后一道防线。深入理解其底层的埋雷、拆雷原理，熟练掌握 `traces.txt` 的阅读技巧，并在架构设计时坚守“主线程轻量化”的底线，是每一位高级 Android 工程师的必修课。