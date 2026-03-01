# Android Studio Profiler 性能优化指南

> 系统掌握 Android Studio Profiler 各模块的使用方法，并建立与 Perfetto 联合分析的完整方法论

---

## 目录

1. [Profiler 概述与 Perfetto 对比](#1-profiler-概述与-perfetto-对比)
2. [CPU Profiler](#2-cpu-profiler)
3. [Memory Profiler](#3-memory-profiler)
4. [Network Profiler](#4-network-profiler)
5. [Energy Profiler](#5-energy-profiler)
6. [Layout Inspector](#6-layout-inspector)
7. [Profiler + Perfetto 联合分析方法论](#7-profiler-perfetto-联合分析方法论)
8. [性能优化 Checklist](#8-性能优化-checklist)

---

## 1. Profiler 概述与 Perfetto 对比

### 1.1 定位差异


|      | **Android Studio Profiler** | **Perfetto**                |
| ---- | --------------------------- | --------------------------- |
| 视角   | **单个 App 进程**内部             | **整个系统**所有进程                |
| 粒度   | 方法级调用栈、对象级内存                | 系统级 atrace 切片、内核调度          |
| 使用方式 | Android Studio 内置 GUI       | 命令行 + 浏览器 (ui.perfetto.dev) |
| 数据关联 | **直接跳转到源码行号**               | 只能看到 trace 切片名              |
| 适用阶段 | 开发/调试阶段                     | 开发/线上问题排查                   |


### 1.2 适用场景


| 场景                     | 推荐工具                 | 原因                    |
| ---------------------- | -------------------- | --------------------- |
| 某个方法执行慢                | **Profiler CPU**     | 能精确到方法名和行号            |
| Activity/Fragment 内存泄漏 | **Profiler Memory**  | Heap Dump 可按类查找泄漏对象   |
| 网络请求慢或重复请求             | **Profiler Network** | 可视化请求时间线+调用栈          |
| 滑动掉帧（不确定原因）            | **Perfetto** 先定位层级   | 需要看跨进程全貌              |
| SurfaceFlinger 合成慢     | **Perfetto**         | Profiler 看不到 SF 进程    |
| Binder 阻塞导致 ANR        | **Perfetto**         | 需要看跨进程 Binder 事务配对    |
| 主线程被调度到小核              | **Perfetto**         | 需要看内核 CPU 调度信息        |
| Compose 过度重组           | **Layout Inspector** | 可看每个 Composable 的重组次数 |


### 1.3 联合使用原则

```
性能问题
  │
  ├─ 已知是 App 代码问题 ──→ 直接用 Profiler
  │
  ├─ 不确定问题在哪层 ──→ Perfetto 先定位
  │     │
  │     └─ 定位到 App 层 ──→ Profiler 深入代码
  │
  └─ 系统/跨进程问题 ──→ Perfetto + dumpsys
```

---

## 2. CPU Profiler

CPU Profiler 用于分析 App 的 CPU 使用情况和方法调用耗时。

### 2.1 启动方式

1. **Android Studio** → **View** → **Tool Windows** → **Profiler**
2. 选择目标设备和进程
3. 点击 **CPU** 时间线进入 CPU Profiler

### 2.2 三种录制模式


| 模式                             | 原理                        | 开销    | 适用场景              |
| ------------------------------ | ------------------------- | ----- | ----------------- |
| **Sample Java/Kotlin Methods** | 定时采样调用栈（默认 1ms）           | 低     | 日常性能分析，找热点方法      |
| **Trace Java/Kotlin Methods**  | 插桩每个方法入口/出口               | **高** | 精确测量方法耗时，短时间使用    |
| **Sample C/C++ Functions**     | 采样 Native 调用栈（simpleperf） | 低     | 分析 JNI/Native 层性能 |


**选择建议**：

- 大多数情况用 **Sample** 模式（开销低，结果足够准确）
- 需要精确到每次方法调用耗时时用 **Trace** 模式（注意会显著拖慢 App）
- 怀疑 Native 层有问题时用 **C/C++ Sample** 模式

### 2.3 四种分析视图

录制完成后，有四种视图展示数据：

**Flame Chart（火焰图）**

```
宽度 = 方法占用的 CPU 时间
从下到上 = 调用栈（底部是调用者，顶部是被调用者）
颜色 = 区分不同包/模块

典型阅读方式：
  找到最宽的"平顶"方法 → 就是 CPU 热点
```


| 视图              | 排列方式         | 适用场景          |
| --------------- | ------------ | ------------- |
| **Flame Chart** | 按时间顺序展示调用栈   | 直观看热点分布       |
| **Top Down**    | 从调用者展开到被调用者  | 分析某个入口方法的耗时分布 |
| **Bottom Up**   | 从被调用者反向展开    | 找到最耗时的叶子方法    |
| **Events**      | 按时间线展示每次方法调用 | 分析调用时序和并发     |


### 2.4 关键指标


| 指标                  | 含义                    | 关注点          |
| ------------------- | --------------------- | ------------ |
| **Wall Clock Time** | 方法从开始到结束的真实时间（含等待）    | 反映用户感知的耗时    |
| **Thread Time**     | 方法实际占用 CPU 的时间        | 反映 CPU 计算量   |
| **Wall >> Thread**  | 说明方法在等待（I/O、锁、Binder） | 需要排查阻塞原因     |
| **Thread ≈ Wall**   | 说明方法在持续计算             | 需要优化算法或减少计算量 |


### 2.5 实战：定位主线程耗时方法

场景：Perfetto 显示某帧 `Choreographer#doFrame` 耗时 45ms，需要找到具体慢在哪。

```
Step 1: Perfetto 定位慢帧时间点（如 t=3.2s）
Step 2: 在同一时间段用 CPU Profiler 录制
Step 3: 切到 Flame Chart，找到主线程
Step 4: 定位 doFrame 对应的调用栈
Step 5: 展开看 measure → layout → draw 各阶段
Step 6: 找到最宽的"平顶"方法
Step 7: 双击直接跳转到源码行号
```

---

## 3. Memory Profiler

Memory Profiler 用于分析 App 的内存使用和泄漏。

### 3.1 实时内存图表

打开 Memory Profiler 后，顶部显示实时内存时间线：


| 内存分区          | 含义                      | 关注点                        |
| ------------- | ----------------------- | -------------------------- |
| **Java Heap** | Java/Kotlin 对象占用        | 最常分析的区域                    |
| **Native**    | C/C++ 分配的内存（malloc）     | Bitmap 在 Android 8.0+ 算在这里 |
| **Graphics**  | GPU 纹理、Surface Buffer 等 | 重度 UI/游戏场景关注               |
| **Stack**     | 线程栈                     | 线程过多时增长                    |
| **Code**      | DEX/OAT/SO 文件映射         | 通常稳定                       |
| **Others**    | 其他                      | —                          |


### 3.2 Heap Dump 分析

**操作**：点击 **Dump Java Heap** 按钮（或 Capture heap dump）

**内存泄漏检测三步法**：

```
Step 1: 反复进入/退出某个 Activity 5 次
Step 2: 手动触发 GC（点击垃圾桶图标）
Step 3: Dump Heap
Step 4: 按包名过滤，搜索目标 Activity 类名
Step 5: 如果已退出的 Activity 仍有实例 → 泄漏
Step 6: 查看 "References" 面板，找到 GC Root 引用链
```

**常见泄漏模式**：


| 泄漏类型        | 典型原因                                  | Heap Dump 表现                    |
| ----------- | ------------------------------------- | ------------------------------- |
| Activity 泄漏 | 匿名内部类持有 Activity 引用                   | 已 finish 的 Activity 实例仍存在       |
| Fragment 泄漏 | Fragment 被 ViewModel 或静态变量引用          | 已 detach 的 Fragment 实例仍存在       |
| Bitmap 泄漏   | 大图未回收、缓存无上限                           | Native 内存持续增长                   |
| Handler 泄漏  | MessageQueue 持有 Handler → 持有 Activity | Handler 的 mCallback 指向 Activity |
| Listener 泄漏 | 注册后未反注册                               | 系统服务持有 App 对象引用                 |


### 3.3 Allocation Tracking

**操作**：在 Memory 时间线上拖选一段时间区间

**分析内容**：

- 该时间段内所有 Java 对象分配
- 按类名分组，显示分配次数和大小
- 点击可查看分配时的调用栈

**典型应用**：

```
场景：滑动列表时内存抖动（GC 频繁）

Step 1: 开始录制，快速滑动列表 5 秒
Step 2: 停止录制
Step 3: 按 "Allocations" 降序排列
Step 4: 查看短时间内大量创建的对象
Step 5: 常见问题：
  - String 拼接产生大量临时对象
  - onDraw() 中创建 Paint/Rect 对象
  - Adapter 中未复用 ViewHolder
```

### 3.4 与 Perfetto 内存数据对比


| 维度          | Profiler Memory | Perfetto Memory  |
| ----------- | --------------- | ---------------- |
| Java 对象详情   | 有（类名、引用链）       | 无                |
| Native 分配详情 | 有限              | 有（需启用 heapprofd） |
| 系统级 RSS/PSS | 无               | 有                |
| 跨进程内存压力     | 无               | 有（可看 LMK 事件）     |


---

## 4. Network Profiler

### 4.1 功能概述


| 功能      | 说明                 |
| ------- | ------------------ |
| 请求时间线   | 按时间展示所有网络请求        |
| 请求/响应详情 | Headers、Body、大小、耗时 |
| 调用栈     | 发起请求的代码位置          |
| 连接类型    | Wi-Fi / 移动网络 / 无网络 |


### 4.2 关注点


| 问题      | 表现             | 优化方向     |
| ------- | -------------- | -------- |
| 重复请求    | 同一 URL 短时间多次请求 | 加缓存或去重   |
| 大响应体    | 单次响应 > 1MB     | 分页、压缩、裁剪 |
| 串行请求    | 多个请求依次发出       | 并行化或合并   |
| 启动时请求过多 | 冷启动阶段大量网络请求    | 延迟非关键请求  |


### 4.3 使用限制

- 需要 App 使用 `HttpURLConnection` 或 OkHttp（需添加 Profiler 拦截器）
- **debuggable=true** 才能使用
- 无法分析其他进程的网络请求（系统级网络分析需用 `tcpdump` 或 Perfetto）

---

## 5. Energy Profiler

### 5.1 功能概述

Energy Profiler 估算 App 的能耗来源，帮助优化后台耗电。


| 数据源       | 说明                 | 高耗电表现           |
| --------- | ------------------ | --------------- |
| CPU       | App 的 CPU 使用率      | 后台持续占用 CPU      |
| Network   | 网络请求频率和大小          | 频繁唤醒网络          |
| Location  | GPS/网络定位请求         | 高精度持续定位         |
| Wake Lock | 阻止设备休眠的锁           | 长时间持有 Wake Lock |
| Alarms    | 定时任务（AlarmManager） | 频繁唤醒            |
| Jobs      | JobScheduler 任务    | 后台频繁执行          |


### 5.2 典型优化场景

```
场景：用户投诉 App 后台耗电

Step 1: 打开 Energy Profiler
Step 2: 将 App 切到后台
Step 3: 等待 2-3 分钟
Step 4: 检查时间线上的能耗事件
Step 5: 重点关注：
  - Wake Lock 是否及时释放
  - 后台是否有持续的网络请求
  - GPS 定位是否在后台仍在运行
  - AlarmManager 是否设置了过短的间隔
```

---

## 6. Layout Inspector

### 6.1 View 层级检查


| 功能        | 说明                   |
| --------- | -------------------- |
| 实时 View 树 | 以树形结构展示当前界面的所有 View  |
| 属性面板      | 选中 View 查看其所有属性值     |
| 3D 视图     | 将 View 层级以 3D 爆炸视图展示 |
| 层级深度      | 显示 View 的嵌套层数        |


**关注的性能信号**：


| 信号               | 问题                         | 优化方向                   |
| ---------------- | -------------------------- | ---------------------- |
| View 层级 > 10 层   | 布局嵌套过深，measure/layout 耗时增加 | 用 ConstraintLayout 扁平化 |
| 大量 `gone` 的 View | 仍占用内存和 measure 时间          | 用 ViewStub 延迟加载        |
| 过多 overdraw      | GPU 重复绘制同一像素区域             | 移除不必要的背景色              |


### 6.2 Compose 重组监控

Android Studio（Hedgehog+）支持 Compose 专属功能：


| 功能                      | 说明                  |
| ----------------------- | ------------------- |
| **Recomposition Count** | 每个 Composable 的重组次数 |
| **Skip Count**          | 被跳过重组的次数            |


**重组过多的典型原因**：


| 原因            | 代码模式                         | 修复                                      |
| ------------- | ---------------------------- | --------------------------------------- |
| 不稳定参数         | 传入 `List` 或 lambda（每次都是新实例）  | 用 `@Immutable` / `@Stable` / `remember` |
| 读取频繁变化的 State | 在高层级 Composable 读取滚动位置等      | 将 State 读取下推到最小 Composable              |
| 未使用 key       | `LazyColumn` 中 items 未设置 key | 加 `key = { item.id }`                   |


**对应 Perfetto**：当 Perfetto 显示 `measure` 或 `layout` 阶段耗时异常长时，用 Layout Inspector 检查 View 层级深度或 Compose 重组次数，定位根因。

---

## 7. Profiler + Perfetto 联合分析方法论

### 7.1 完整工作流

```
                         性能问题
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
           Perfetto                  Profiler
       (系统级定位)              (App级深入)
                 │                     │
      ┌──────────┼──────────┐          │
      ▼          ▼          ▼          │
   App层慢   SF合成慢   CPU调度差      │
      │                                │
      └───────────── 切换到 ───────────┘
                            │
                   CPU/Memory Profiler
                   精确到代码行号
```

### 7.2 四大场景联合分析

**场景 1：列表滑动掉帧**（对应 [掉帧分析方法](./掉帧分析方法.md)）


| 步骤  | 工具               | 操作                                        |
| --- | ---------------- | ----------------------------------------- |
| 1   | Perfetto         | 找到 doFrame > 16ms 的帧                      |
| 2   | Perfetto         | 确认瓶颈在 measure/layout/draw 还是 RenderThread |
| 3   | Profiler CPU     | Sample 模式录制滑动过程                           |
| 4   | Profiler CPU     | Flame Chart 找到主线程热点方法                     |
| 5   | Layout Inspector | 检查 View 层级深度 / Compose 重组次数               |


**场景 2：冷启动慢**（对应 [启动分析流程](./启动分析流程.md)）


| 步骤  | 工具               | 操作                                               |
| --- | ---------------- | ------------------------------------------------ |
| 1   | Perfetto         | 拆分启动各阶段耗时（bindApplication → activityResume → 首帧） |
| 2   | Profiler CPU     | Trace 模式录制启动过程（精确每个方法耗时）                         |
| 3   | Profiler CPU     | Top Down 视图展开 Application.onCreate()             |
| 4   | Profiler CPU     | 找到初始化耗时最大的 SDK/模块                                |
| 5   | Profiler Network | 检查启动阶段是否有阻塞性网络请求                                 |


**场景 3：内存泄漏导致 OOM**


| 步骤  | 工具              | 操作                          |
| --- | --------------- | --------------------------- |
| 1   | Profiler Memory | 观察内存曲线是否持续上升                |
| 2   | Profiler Memory | 反复操作后 Dump Heap             |
| 3   | Profiler Memory | 按类名搜索已销毁的 Activity/Fragment |
| 4   | Profiler Memory | 查看 GC Root 引用链找到泄漏源         |
| 5   | Perfetto        | 检查是否触发了 LMK（低内存杀进程）         |


**场景 4：后台 ANR**（对应 [ANR分析实战](../practice/ANR分析实战.md)）


| 步骤  | 工具              | 操作                     |
| --- | --------------- | ---------------------- |
| 1   | Perfetto        | 查看 ANR 时间点的主线程状态       |
| 2   | Perfetto        | 检查是否有长 Binder 事务阻塞主线程  |
| 3   | Profiler CPU    | 如能复现，录制 ANR 前后的 CPU 活动 |
| 4   | Profiler CPU    | Bottom Up 视图找到阻塞主线程的方法 |
| 5   | `adb bugreport` | 查看 traces.txt 中的线程堆栈   |


---

## 8. 性能优化 Checklist

按 JD 要求的方向整理，每项注明推荐工具：

### 8.1 渲染优化


| 检查项                                     | 工具               | 标准               |
| --------------------------------------- | ---------------- | ---------------- |
| 每帧 doFrame < 16ms（60fps）或 < 8ms（120fps） | Perfetto         | 掉帧率 < 1%         |
| View 层级深度 < 10 层                        | Layout Inspector | 越浅越好             |
| 无 overdraw（开发者选项 → 调试 GPU 过度绘制）         | 设备设置             | 红色区域尽量少          |
| Compose 无过度重组                           | Layout Inspector | 稳定状态下重组次数不增长     |
| RenderThread 无阻塞                        | Perfetto         | DrawFrame < 8ms  |
| 自定义 View.onDraw() 无对象分配                 | Profiler Memory  | onDraw 中无 new 操作 |


### 8.2 启动优化


| 检查项                            | 工具                      | 标准                            |
| ------------------------------ | ----------------------- | ----------------------------- |
| 冷启动 < 1.5s                     | `adb shell am start -W` | 首帧上屏时间                        |
| Application.onCreate() < 300ms | Profiler CPU (Trace)    | —                             |
| 首屏 Activity.onCreate() < 200ms | Profiler CPU (Trace)    | —                             |
| 启动阶段无主线程 Binder 阻塞             | Perfetto                | 无 > 10ms 的 binder transaction |
| ContentProvider 数量合理           | 代码审查                    | 延迟非必要 Provider                |
| 启动阶段无同步网络/IO 请求                | Profiler Network + CPU  | —                             |


### 8.3 内存优化


| 检查项                      | 工具                          | 标准                  |
| ------------------------ | --------------------------- | ------------------- |
| 无 Activity/Fragment 内存泄漏 | Profiler Memory (Heap Dump) | 退出后实例数 = 0          |
| 无 Bitmap 泄漏              | Profiler Memory             | Native 内存不持续增长      |
| GC 不频繁（不影响帧率）            | Perfetto                    | 无 > 5ms 的 GC pause  |
| 大图按需加载、缩放                | 代码审查                        | 不加载超过 View 尺寸的图片    |
| 缓存有上限                    | 代码审查                        | LruCache 设置 maxSize |


### 8.4 Binder / 跨进程优化


| 检查项                    | 工具                   | 标准                           |
| ---------------------- | -------------------- | ---------------------------- |
| 主线程无耗时 Binder 调用       | Perfetto             | 无 > 5ms 的 binder transaction |
| Binder 线程池未耗尽          | Perfetto + `dumpsys` | 默认上限 15 个线程                  |
| ContentProvider 查询在子线程 | Profiler CPU         | 主线程无 query() 调用              |
| 批量 Binder 调用合并         | 代码审查                 | 避免循环中逐条调 system service      |


---

**相关文档**：

- [Perfetto使用指南](./Perfetto使用指南.md) — 系统级追踪工具
- [掉帧分析方法](./掉帧分析方法.md) — 列表滑动掉帧排查
- [启动分析流程](./启动分析流程.md) — 冷启动优化
- [转场动画卡顿实战](../practice/转场动画卡顿实战.md) — 转场动画卡顿排查
- [ANR分析实战](../practice/ANR分析实战.md) — Binder 阻塞导致的 ANR
- [低内存频繁杀后台实战](../practice/低内存频繁杀后台实战.md) — LMK 机制与内存排查
- [刷新率切换动画异常实战](../practice/刷新率切换动画异常实战.md) — 刷新率切换导致动画异常排查

