# ANR 分析

## 目录

1. [场景描述](#场景描述)
2. [第一步：ANR 机制回顾](#第一步anr-机制回顾)
3. [第二步：收集 ANR 信息](#第二步收集-anr-信息)
4. [第三步：traces.txt 解读](#第三步tracestxt-解读)
5. [第四步：Perfetto 中定位 ANR](#第四步perfetto-中定位-anr)
6. [第五步：优化方向](#第五步优化方向)
7. [一、ANR 相关](#一anr-相关)
8. [二、内存与 LMK](#二内存与-lmk)
9. [三、SurfaceFlinger 与刷新率](#三surfaceflinger-与刷新率)
10. [四、Perfetto 通用采集](#四perfetto-通用采集)
11. [五、启动与帧率](#五启动与帧率)
12. [六、常用 SQL（Perfetto Query）](#六常用-sqlperfetto-query)

---

## 场景描述

> **线上反馈**：后台播放音乐时，前台 App 偶发 ANR。traces.txt 显示主线程阻塞在 `Binder.transact()`，对端为 `system_server` 的 `AudioService`。

这是典型的 **Binder 阻塞导致的 ANR**。主线程在执行跨进程调用时被阻塞，无法响应输入事件，最终触发 Input dispatching timeout。

---

## 第一步：ANR 机制回顾

### 1.1 各类 ANR 超时时间


| 类型                    | 超时时间             | 触发条件                       |
| --------------------- | ---------------- | -------------------------- |
| **Input dispatching** | 5s               | 触摸/按键事件未在 5s 内响应           |
| **BroadcastReceiver** | 前台 10s / 后台 60s  | onReceive 执行超时             |
| **Service**           | 前台 20s / 后台 200s | onCreate/onStartCommand 超时 |
| **ContentProvider**   | publish 超时       | getContentProvider 超时      |


### 1.2 源码路径

- **ANR 调度**：`frameworks/base/services/core/java/com/android/server/am/AnrHelper.java`
- **Input 超时**：AMS 监控 InputDispatcher 的调度状态

### 1.3 ANR 发生时的系统行为

1. 系统检测到超时
2. 收集各进程的 Java/Kernel stack，写入 `/data/anr/traces.txt`
3. 弹出 ANR 弹窗（若为前台 App）
4. 可配合 `bugreport` 获取更完整信息

---

## 第二步：收集 ANR 信息

### 2.1 基础命令

```bash
# 拉取 ANR 时的堆栈
adb pull /data/anr/traces.txt ./

# 完整 bugreport（含 logcat、系统状态）
adb bugreport

# 查看 ANR 相关进程状态
adb shell dumpsys activity processes | grep -A5 "ANR"
```

### 2.2 traces.txt 结构

```
----- pid 12345 at 2025-02-23 10:00:00 -----
Cmd line: com.example.app
...
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12c4a0 self=0x7a12345600
  | held mutexes=
  at com.example.SomeClass.someMethod(SomeClass.java:42)
  at android.os.Binder.transact(Native method)
  ...
```

重点找到 **main** 线程的 `Blocked` 或 `Waiting` 状态及其 stack trace。

---

## 第三步：traces.txt 解读

### 3.1 常见主线程阻塞原因


| Stack 特征                                | 含义                                 | 优化方向             |
| --------------------------------------- | ---------------------------------- | ---------------- |
| `Binder.transact()`                     | 主线程发起 Binder 调用，对端处理慢或 Binder 线程池满 | 异步化、避免主线程 Binder |
| `Object.wait()`                         | 等待锁，可能死锁                           | 检查锁顺序、减少锁粒度      |
| `Thread.sleep()`                        | 主线程不应 sleep                        | 移除或移至子线程         |
| `SQLiteDatabase.rawQuery()`             | 主线程数据库查询                           | 移至子线程或使用 Room 异步 |
| `SharedPreferences$EditorImpl.commit()` | 同步写 SP                             | 改用 `apply()`     |
| `FileInputStream.read()`                | 主线程 IO                             | 移至子线程            |


### 3.2 Binder 阻塞的典型栈

```
"main" Blocked
  at android.os.BinderProxy.transactNative(Native method)
  at android.os.BinderProxy.transact(BinderProxy.java:xxx)
  at com.android.internal.app.IAppOpsService.checkOperation(...)
  ...
```

说明：主线程调用 `checkOperation`（或其他 Binder 接口），对端 `system_server` 的 Binder 线程繁忙或处理慢，导致主线程阻塞。

---

## 第四步：Perfetto 中定位 ANR

若在 ANR 发生前后有 Perfetto trace，可精确定位阻塞时段。

### 4.1 主线程长时间阻塞 Slice

```sql
-- 查找主线程 > 1 秒的 slice
SELECT ts, dur/1000000.0 as ms, name FROM slice
WHERE track_id IN (
  SELECT id FROM thread_track tt
  JOIN thread t ON tt.utid = t.utid
  WHERE t.name = 'main'
)
AND dur > 1000000000
ORDER BY dur DESC;
```

### 4.2 Binder 线程池使用情况

```sql
-- Binder 线程调用统计（按总耗时排序）
SELECT t.name, count(*) as binder_calls, sum(s.dur)/1000000.0 as total_ms
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
WHERE t.name LIKE 'Binder:%'
GROUP BY t.name
ORDER BY total_ms DESC;
```

### 4.3 与 ANR 时间对齐

在 Perfetto 时间轴上找到 ANR 弹窗出现的时刻，回溯主线程在当时的 slice，即可定位阻塞点。

---

## 第五步：优化方向


| 方向                 | 具体措施                                              |
| ------------------ | ------------------------------------------------- |
| **主线程 Binder 异步化** | 将检查类 Binder 调用移至子线程，结果回调主线程                       |
| **Binder 线程池**     | 系统默认 15 个 Binder 线程，App 无法直接修改；减少主线程 Binder 调用可缓解 |
| **StrictMode**     | 开启检测主线程 IO/网络，提前暴露问题                              |
| **WorkManager**    | 耗时操作使用 WorkManager 替代前台直接执行                       |
| **避免死锁**           | 统一锁顺序，避免多锁交叉等待                                    |


---

# AI 交互建议（适用于所有场景）

在实践过程中，可向 AI 提出以下问题，辅助深入分析：


| 场景   | 示例问题                                                               |
| ---- | ------------------------------------------------------------------ |
| 转场动画 | 「Perfetto 中 SurfaceFlinger 的 onMessageRefresh 有一帧耗时 25ms，帮我分析可能原因」 |
| 转场动画 | 「转场动画期间 Layer 数量从 10 增加到 25，这正常吗？如何减少？」                            |
| ANR  | 「帮我分析这个 ANR traces.txt，主线程 stack trace 显示阻塞在 Binder.transact」      |
| LMK  | 「低内存设备上如何用 Perfetto 追踪 lmk 杀进程的时机？」                                |
| 刷新率  | 「如何在 Perfetto 中查看刷新率切换事件？」                                         |


---

# 真机实操速查表

## 一、ANR 相关


| 用途           | 命令                                    |
| ------------ | ------------------------------------- |
| 拉取 ANR 堆栈    | `adb pull /data/anr/traces.txt ./`    |
| 完整 bugreport | `adb bugreport`                       |
| 查看 ANR 进程    | `adb shell dumpsys activity processes |
| 查看主线程状态      | `adb shell dumpsys activity top`      |


## 二、内存与 LMK


| 用途          | 命令                                                             |
| ----------- | -------------------------------------------------------------- |
| 进程内存        | `adb shell dumpsys meminfo <package>`                          |
| 系统内存        | `adb shell cat /proc/meminfo`                                  |
| 进程 ADJ      | `adb shell dumpsys activity processes                          |
| LMK minfree | `adb shell cat /sys/module/lowmemorykiller/parameters/minfree` |
| LMK adj     | `adb shell cat /sys/module/lowmemorykiller/parameters/adj`     |
| dump 堆      | `adb shell am dumpheap <pid> /data/local/tmp/heap.hprof`       |


## 三、SurfaceFlinger 与刷新率


| 用途           | 命令                                 |
| ------------ | ---------------------------------- |
| 刷新率          | `adb shell dumpsys SurfaceFlinger  |
| 刷新率策略        | `adb shell dumpsys SurfaceFlinger  |
| Layer 信息     | `adb shell dumpsys SurfaceFlinger` |
| 禁用 HW 合成（调试） | `adb shell setprop debug.sf.hw 0`  |


## 四、Perfetto 通用采集


| 场景       | 命令                                                                                                                                       |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 转场动画     | `adb shell perfetto -o /data/misc/perfetto-traces/trace_transition.pb -t 10s sched freq idle am wm gfx view binder_driver hal input res` |
| ANR / 卡顿 | `adb shell perfetto -o /data/misc/perfetto-traces/trace_anr.pb -t 20s sched am wm gfx view binder_driver memory`                         |
| 内存压力     | `adb shell perfetto -o /data/misc/perfetto-traces/trace_mem.pb -t 30s sched memory am`                                                   |
| 刷新率      | `adb shell perfetto -o /data/misc/perfetto-traces/trace_refresh.pb -t 15s sched gfx hal`                                                 |
| 拉取 trace | `adb pull /data/misc/perfetto-traces/trace_xxx.pb ./`                                                                                    |


## 五、启动与帧率


| 用途      | 命令                                                                   |
| ------- | -------------------------------------------------------------------- |
| 冷启动测量   | `adb shell am force-stop <pkg> && adb shell am start -W <pkg>/<act>` |
| 帧统计     | `adb shell dumpsys gfxinfo <package>`                                |
| 帧统计（详细） | `adb shell dumpsys gfxinfo <package> framestats`                     |


## 六、常用 SQL（Perfetto Query）


| 用途         | SQL 片段                                                                                                                        |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------- |
| SF 合成耗时    | `WHERE name = 'onMessageRefresh' AND dur > 8000000`                                                                           |
| 主线程长耗时     | `WHERE track_id IN (SELECT id FROM thread_track tt JOIN thread t ON tt.utid=t.utid WHERE t.name='main') AND dur > 1000000000` |
| Binder 统计  | `WHERE t.name LIKE 'Binder:%'` 配合 `GROUP BY t.name`                                                                           |
| 转场 Layer 数 | `WHERE name LIKE '%numLayers%'`                                                                                               |


