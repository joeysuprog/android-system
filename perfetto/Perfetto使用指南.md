# Perfetto 使用指南

> 系统掌握 Perfetto 抓取 Trace、UI 分析、SQL 查询及性能分析方法论

---

## 目录

1. [Perfetto 概述](#1-perfetto-概述)
2. [抓取 Trace 的三种方法](#2-抓取-trace-的三种方法)
3. [Perfetto UI 界面操作](#3-perfetto-ui-界面操作)
4. [关键 Trace 切片（Slice）解读](#4-关键-trace-切片slice解读)
5. [Perfetto SQL 常用查询](#5-perfetto-sql-常用查询)
6. [性能分析方法论](#6-性能分析方法论)
7. [AI 交互建议](#7-ai-交互建议)

---

## 1. Perfetto 概述

### 1.1 什么是 Perfetto

**Perfetto** 是 Android 和 Chrome 团队共同维护的**下一代追踪框架**，用于替代传统的 systrace。它可以：

- 在 **内核层** 和 **用户空间** 统一采集 trace 数据
- 支持 CPU、GPU、内存、电源、Binder、Input 等多种数据源
- 通过 **Web UI** 可视化分析，无需额外安装原生应用
- 支持 **SQL** 查询，便于批量统计和深度分析

### 1.2 与 systrace 的对比


| 特性    | systrace    | Perfetto                 |
| ----- | ----------- | ------------------------ |
| 数据源   | 以 atrace 为主 | 内核 ftrace + atrace + 自定义 |
| 配置方式  | 命令行参数       | 灵活的 .pbtx 配置文件           |
| 分析 UI | HTML 本地打开   | 云端/本地 Web UI             |
| 扩展性   | 有限          | 可扩展数据源与解析器               |


### 1.3 Web UI 入口

- **官方线上环境**：[https://ui.perfetto.dev/](https://ui.perfetto.dev/)
- 支持拖拽 `.perfetto-trace` 或 `.pftrace` 文件进行分析
- 支持连接设备进行 **录制（Record）**

---

## 2. 抓取 Trace 的三种方法

### 方法 1：adb 命令行（Perfetto CLI）

在设备上预置 `perfetto` 可执行文件（Android 9+ 通常已有），通过 adb 直接抓取：

```bash
# 抓取 10 秒，默认包含 sched、ftrace、atrace 等
adb shell perfetto -o /data/misc/perfetto-traces/trace.pb -t 10s

# 拉取到本地
adb pull /data/misc/perfetto-traces/trace.pb ./trace.pb
```

**指定数据源**（通过 `-c` 传入配置）：

```bash
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.pb <<EOF
duration_ms: 10000
buffers: {
  size_kb: 63488
  fill_policy: RING_BUFFER
}
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "power/cpu_frequency"
      atrace_apps: "*"
    }
  }
}
EOF
```

### 方法 2：Perfetto UI 网页录制

1. 打开 [https://ui.perfetto.dev/](https://ui.perfetto.dev/)
2. 点击 **Record new trace**
3. 选择 **Android**，按提示连接设备（需开启 USB 调试）
4. 勾选需要的 trace 类型（e.g. **Scheduling details**, **CPU frequency**, **Atrace**）
5. 可选：勾选 **Record for** 设置时长，或手动停止
6. 点击 **Record** 开始，操作设备复现问题后停止
7. 自动跳转到分析视图

### 方法 3：自定义 Trace 配置文件（.pbtx）

创建配置文件 `config.pbtx`：

```protobuf
buffers: {
  size_kb: 63488
  fill_policy: RING_BUFFER
}

duration_ms: 15000

data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "power/cpu_frequency"
      ftrace_events: "ftrace/print"
      atrace_apps: "com.android.systemui"
      atrace_apps: "com.example.yourapp"
    }
  }
}

data_sources: {
  config {
    name: "traced_perf"
    perf_event_config {
      callstack_sampling {
        scope {
          process_cmdline: "com.example.yourapp"
        }
        sample_frequency: 1000
      }
    }
  }
}
```

执行：

```bash
adb push config.pbtx /data/local/tmp/config.pbtx
adb shell perfetto -c /data/local/tmp/config.pbtx -o /data/misc/perfetto-traces/trace.pb
adb pull /data/misc/perfetto-traces/trace.pb trace.pb
```

---

## 3. Perfetto UI 界面操作

### 3.1 时间轴导航


| 操作        | 说明            |
| --------- | ------------- |
| **W**     | 放大（Zoom in）   |
| **S**     | 缩小（Zoom out）  |
| **A**     | 左移（Pan left）  |
| **D**     | 右移（Pan right） |
| **鼠标滚轮**  | 缩放时间轴         |
| **拖拽时间轴** | 平移视图          |


### 3.2 固定轨道（Pin Tracks）

- 点击轨道左侧 **图钉** 图标，可固定重要进程/线程
- 固定后滚动时保持可见，便于对比分析

### 3.3 Slice 详情

- **点击** 任意 slice 可查看：
  - Duration（时长）
  - Timestamp（时间戳）
  - Arguments（参数，如 Binder transaction 的 code）
- **右键** slice 可复制、筛选、跳转等

### 3.4 Flow 事件

- 部分 slice 带 **流向箭头**，表示因果/调用关系
- 如：Binder transaction 从 Client 到 Server 的连线
- 用于追踪跨进程、跨线程的调用链

### 3.5 SQL 查询标签页

- 左侧 **Query (SQL)** 标签可执行 SQL
- 支持 `slice`、`thread_track`、`process` 等表
- 结果支持导出为 CSV

---

## 4. 关键 Trace 切片（Slice）解读


| Slice 名称                  | 含义                                         | 所属线程/进程                     | 性能关注点                        |
| ------------------------- | ------------------------------------------ | --------------------------- | ---------------------------- |
| **Choreographer#doFrame** | 一帧内主线程的全部工作（INPUT → ANIMATION → TRAVERSAL） | App 主线程                     | 若 > 16ms 则为 UI jank          |
| **DrawFrame**             | RenderThread 执行一帧渲染（DisplayList 回放 + GPU）  | RenderThread                | 若 > 16ms 则为 render jank      |
| **queueBuffer**           | Buffer 提交到 BufferQueue                     | App 进程                      | 若频繁阻塞，可能 buffer stuffing     |
| **onMessageRefresh**      | SurfaceFlinger 执行一帧合成                      | surfaceflinger 主线程          | 若 > 16ms 则为 composition jank |
| **acquireBuffer**         | SurfaceFlinger 从 BufferQueue 取 Buffer      | surfaceflinger              | 若耗时过长，Producer 可能被阻塞         |
| **HWC::present**          | 硬件 Composer 将合成结果提交给 Display               | surfaceflinger / hwcomposer | 显示最后一步                       |
| **deliverInputEvent**     | 输入事件投递到目标 View                             | App 主线程                     | 若延迟大，触摸反馈慢                   |
| **measure**               | View 树 measure 阶段                          | App 主线程                     | 复杂 layout 导致超时               |
| **layout**                | View 树 layout 阶段                           | App 主线程                     | 同上                           |
| **draw**                  | View 树 draw 阶段（含 DisplayList 录制）           | App 主线程                     | 复杂绘制导致超时                     |
| **binder transaction**    | Binder 跨进程调用                               | 任意涉及 Binder 的线程             | 若 > 5ms 需关注是否阻塞主线程           |


---

## 5. Perfetto SQL 常用查询

### 5.1 掉帧分析（doFrame > 16ms）

```sql
-- 查找 doFrame 超过 16ms 的帧（纳秒：16000000）
SELECT ts, dur/1000000.0 AS ms, name
FROM slice
WHERE name LIKE '%doFrame%' AND dur > 16000000
ORDER BY dur DESC
LIMIT 20;
```

### 5.2 RenderThread DrawFrame 耗时

```sql
SELECT ts, dur/1000000.0 AS ms
FROM slice
WHERE name = 'DrawFrame'
ORDER BY dur DESC
LIMIT 20;
```

### 5.3 SurfaceFlinger 合成耗时

```sql
SELECT ts, dur/1000000.0 AS ms
FROM slice
WHERE name = 'onMessageRefresh'
ORDER BY dur DESC
LIMIT 20;
```

### 5.4 Binder 调用耗时（> 5ms）

```sql
SELECT ts, dur/1000000.0 AS ms, name
FROM slice
WHERE name LIKE 'binder transaction%' AND dur > 5000000
ORDER BY dur DESC
LIMIT 20;
```

### 5.5 某进程的所有线程活动（按耗时排序）

```sql
SELECT t.name AS thread_name, s.ts, s.dur/1000000.0 AS ms, s.name AS slice_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE p.name LIKE '%com.example%'
ORDER BY s.dur DESC
LIMIT 50;
```

### 5.6 CPU 频率变化

```sql
SELECT ts, dur/1000000.0 AS ms, value
FROM counter c
JOIN counter_track ct ON c.track_id = ct.id
WHERE ct.name LIKE '%cpu_freq%'
ORDER BY ts
LIMIT 500;
```

### 5.7 调度与 runnable 延迟（sched 表）

```sql
-- sched 表记录线程在 CPU 上的运行片段；dur 为运行时长
-- 可配合 thread_state 表查看 runnable 等待时间
SELECT ts, dur/1000.0 AS us, utid
FROM sched
WHERE utid IN (SELECT utid FROM thread WHERE name LIKE '%main%')
ORDER BY ts DESC
LIMIT 100;
```

### 5.8 BufferQueue 状态（若有 buffer 相关 trace）

```sql
-- 查找与 buffer 相关的 slice（名称因实现可能不同）
SELECT ts, dur/1000000.0 AS ms, name
FROM slice
WHERE name LIKE '%queueBuffer%' OR name LIKE '%acquireBuffer%'
ORDER BY ts
LIMIT 100;
```

### 5.9 帧时间线（doFrame 与 DrawFrame 对齐）

```sql
SELECT
  s.name,
  s.ts,
  s.dur/1000000.0 AS ms,
  t.name AS thread
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
WHERE s.name IN ('Choreographer#doFrame', 'DrawFrame', 'queueBuffer')
ORDER BY s.ts
LIMIT 200;
```

### 5.10 内存使用（若 trace 包含 memory 数据源）

```sql
-- 进程内存 RSS（需启用 memory 数据源）
SELECT ts, value, track_id
FROM counter c
JOIN counter_track ct ON c.track_id = ct.id
WHERE ct.name LIKE '%rss%'
ORDER BY ts DESC
LIMIT 100;
```

### 5.11 doFrame 的 P50 / P90 / P99 耗时统计

```sql
-- 先构建带排名的帧耗时
WITH frame_durations AS (
  SELECT dur/1000000.0 AS ms
  FROM slice
  WHERE name LIKE '%doFrame%' AND dur > 0
),
ranked AS (
  SELECT ms, ROW_NUMBER() OVER (ORDER BY ms) - 1 AS rn,
         (SELECT COUNT(*) FROM frame_durations) AS cnt
  FROM frame_durations
)
SELECT
  (SELECT MIN(ms) FROM frame_durations) AS min_ms,
  (SELECT MAX(ms) FROM frame_durations) AS max_ms,
  (SELECT AVG(ms) FROM frame_durations) AS avg_ms,
  (SELECT ms FROM ranked WHERE rn = cnt/2 LIMIT 1) AS p50_ms,
  (SELECT ms FROM ranked WHERE rn = CAST(cnt * 0.9 AS INT) LIMIT 1) AS p90_ms,
  (SELECT ms FROM ranked WHERE rn = CAST(cnt * 0.99 AS INT) LIMIT 1) AS p99_ms;
```

若 Perfetto 版本不支持 `ROW_NUMBER()`，可用简化版：

```sql
SELECT
  COUNT(*) AS frame_count,
  MIN(dur/1000000.0) AS min_ms,
  MAX(dur/1000000.0) AS max_ms,
  AVG(dur/1000000.0) AS avg_ms
FROM slice
WHERE name LIKE '%doFrame%' AND dur > 0;
```

---

## 6. 性能分析方法论

### 6.1 五步分析法


| 步骤         | 操作                                       | 目的                                        |
| ---------- | ---------------------------------------- | ----------------------------------------- |
| **Step 1** | 复现问题时录制 trace                            | 确保 trace 中包含卡顿片段                          |
| **Step 2** | 定位卡顿帧（doFrame > 16ms 或 DrawFrame > 16ms） | 找到问题发生的时间点                                |
| **Step 3** | 深入慢帧，看是哪一阶段慢                             | 区分 measure / layout / draw / RenderThread |
| **Step 4** | 检查 Binder 阻塞、CPU 调度、锁竞争                  | 排除系统/跨进程因素                                |
| **Step 5** | 形成假设并用更细 trace 验证                        | 针对某进程/某场景二次抓取                             |


### 6.2 典型问题与对应 slice


| 现象      | 优先查看的 slice                      | 可能原因             |
| ------- | -------------------------------- | ---------------- |
| 滑动卡顿    | doFrame, measure, layout         | 复杂 layout、主线程阻塞  |
| 动画掉帧    | Choreographer#doFrame, DrawFrame | 动画计算或绘制过重        |
| 点击延迟    | deliverInputEvent                | 主线程排队、WMS 焦点慢    |
| 多窗口/分屏卡 | onMessageRefresh, acquireBuffer  | SF 合成压力、Layer 过多 |
| 启动慢     | binder transaction, doFrame      | Binder 阻塞、初始化重   |


### 6.3 抓取建议

- **复现路径**：先确定操作步骤，再开始录制，避免无效 trace
- **时长**：一般 10–30 秒即可，过长文件大、分析慢
- **数据源**：至少包含 `sched`、`atrace`；需要时加 `cpu_frequency`、`memory`
- **应用过滤**：用 `atrace_apps` 只 trace 目标应用，减小噪音

---

## 7. AI 交互建议

在与 AI 协作分析 Perfetto trace 时，可尝试：

1. **整体分析**
  > 「帮我分析这段 Perfetto trace 数据，找出掉帧原因。」
2. **统计类**
  > 「写一个 Perfetto SQL 查询来统计所有 doFrame 的 P50/P90/P99 耗时。」
3. **定位类**
  > 「这段 trace 里主线程在 t=5s 附近阻塞了约 100ms，请帮我找可能原因。」
4. **对比类**
  > 「对比滑动列表和静态界面两种场景的 doFrame 分布，写 SQL 实现。」
5. **配置类**
  > 「我想只 trace 某应用的 measure/layout/draw，请给我一个 perfetto 配置示例。」

---

**相关文档**：  

- [渲染全链路](../framework/渲染全链路.md)  
- [VSync与帧调度](../framework/VSync与帧调度.md)

