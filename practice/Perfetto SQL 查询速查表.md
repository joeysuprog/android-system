# Perfetto SQL 查询速查表

> Perfetto Trace Processor SQL 常用查询，按使用场景分类

**使用方式**：在 Perfetto UI (https://ui.perfetto.dev/) 中打开 trace 后，点击 **Query (SQL)** 标签执行。

---

## 掉帧分析

### 1. 查找掉帧帧（doFrame > 16ms）

```sql
-- 查找 Choreographer#doFrame 耗时超过 16.6ms 的帧（60fps 下为掉帧）
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE s.name LIKE 'Choreographer#doFrame%'
  AND s.dur > 16666666  -- 16.6ms in ns
ORDER BY s.dur DESC;
```

### 2. FrameTimeline：查找 Jank 帧（需 Android 12+）

```sql
-- 从 actual_frame_timeline_slice 查找 jank_type 非 None 的帧
SELECT
  ts,
  dur / 1e6 AS dur_ms,
  jank_type,
  on_time_finish,
  present_type,
  layer_name,
  process.name AS process_name
FROM actual_frame_timeline_slice
LEFT JOIN process USING (upid)
WHERE jank_type != 'None' AND jank_type IS NOT NULL
ORDER BY ts;
```

### 3. RenderThread 慢帧

```sql
-- 查找 RenderThread 上 DrawFrame 等耗时较长的 slice
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  s.category,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE t.name = 'RenderThread'
  AND s.dur > 8000000  -- 8ms
ORDER BY s.dur DESC;
```

### 4. SurfaceFlinger 合成慢帧

```sql
-- 查找 SurfaceFlinger 主线程耗时较长的 slice
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE p.name LIKE '%surfaceflinger%'
  AND t.name LIKE '%main%'
  AND s.dur > 8000000
ORDER BY s.dur DESC;
```

### 5. 帧耗时分布 P50/P90/P99

```sql
-- 统计 doFrame 耗时的百分位数
SELECT
  APPROX_PERCENTILE(dur_ms, 50) AS p50_ms,
  APPROX_PERCENTILE(dur_ms, 90) AS p90_ms,
  APPROX_PERCENTILE(dur_ms, 99) AS p99_ms,
  AVG(dur_ms) AS avg_ms,
  COUNT(*) AS frame_count
FROM (
  SELECT s.dur / 1e6 AS dur_ms
  FROM slice s
  JOIN thread_track tt ON s.track_id = tt.id
  JOIN thread t ON tt.utid = t.utid
  WHERE s.name LIKE 'Choreographer#doFrame%'
);
```

---

## 启动分析

### 6. 启动关键 slice（按包名过滤）

```sql
-- 查看目标应用启动期间的 key slice
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  s.category,
  t.name AS thread_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE p.name = 'com.example.yourapp'  -- 替换为目标包名
  AND s.ts BETWEEN (
    SELECT MIN(ts) FROM slice
    JOIN thread_track tt ON slice.track_id = tt.id
    JOIN thread t ON tt.utid = t.utid
    JOIN process p ON t.upid = p.upid
    WHERE p.name = 'com.example.yourapp'
  ) AND (
    SELECT MIN(ts) FROM slice
    JOIN thread_track tt ON slice.track_id = tt.id
    JOIN thread t ON tt.utid = t.utid
    JOIN process p ON t.upid = p.upid
    WHERE p.name = 'com.example.yourapp'
  ) + 5e9  -- 启动后 5 秒内
ORDER BY s.ts;
```

### 7. 启动期间 Class Loading

```sql
-- 查找 ClassLoader 加载类的耗时
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE (s.name LIKE '%ClassLoader%' OR s.name LIKE '%loadClass%')
  AND s.dur > 100000  -- > 0.1ms
ORDER BY s.dur DESC;
```

### 8. 启动期间 GC

```sql
-- 查找 GC 相关 slice（会阻塞主线程）
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE (s.name LIKE '%GC%' OR s.name LIKE '%gc%' OR s.name LIKE '%CollectGarbage%')
ORDER BY s.ts;
```

---

## Binder 分析

### 9. 耗时 Binder 调用

```sql
-- 查找 binder  transaction 耗时超过阈值的调用
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE s.name = 'binder transaction'
  AND s.dur > 5000000  -- 5ms
ORDER BY s.dur DESC;
```

### 10. 主线程 Binder 阻塞

```sql
-- 查找主线程上的 binder 调用（主线程等待 Binder 返回会阻塞 UI）
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE s.name = 'binder transaction'
  AND t.name = 'main'
ORDER BY s.dur DESC;
```

### 11. Binder 线程池使用

```sql
-- 统计各进程 Binder 线程的活跃情况
SELECT
  p.name AS process_name,
  t.name AS thread_name,
  COUNT(*) AS slice_count,
  SUM(s.dur) / 1e6 AS total_dur_ms
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE t.name LIKE 'Binder:%'
  AND s.name = 'binder transaction'
GROUP BY p.name, t.name
ORDER BY total_dur_ms DESC;
```

---

## CPU 调度分析

### 12. 目标线程 CPU 使用

```sql
-- 统计某线程的 CPU 运行时间（需 sched 数据）
SELECT
  t.name AS thread_name,
  p.name AS process_name,
  SUM(s.dur) / 1e6 AS cpu_time_ms,
  COUNT(*) AS switch_count
FROM sched s
JOIN thread t ON s.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE p.name = 'com.example.yourapp'  -- 替换为目标包名
GROUP BY t.name, p.name
ORDER BY cpu_time_ms DESC;
```

### 13. CPU 频率变化

```sql
-- 查看各核 CPU 频率随时间变化（需 power/cpu_frequency）
SELECT
  ts,
  cpu,
  value AS freq_khz
FROM counter
WHERE track_id IN (
  SELECT id FROM counter_track
  WHERE name LIKE '%cpu_freq%' OR name LIKE '%frequency%'
)
ORDER BY ts
LIMIT 500;
```

### 14. 线程调度延迟（被阻塞的线程状态）

```sql
-- 查找线程 blocked 状态及阻塞时长（state 含 D/S 等）
SELECT
  ts.ts,
  ts.dur / 1e6 AS dur_ms,
  ts.state,
  ts.blocked_function,
  t.name AS thread_name,
  p.name AS process_name
FROM thread_state ts
JOIN thread t ON ts.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE ts.state IN ('D', 'S', 'R+')  -- D=uninterruptible, S=interruptible sleep
  AND ts.dur > 1000000  -- > 1ms
ORDER BY ts.dur DESC
LIMIT 100;
```

---

## 内存分析

### 15. 进程内存使用（Counter）

```sql
-- 查看进程内存 counter 随时间变化（需 memory 数据源）
SELECT
  c.ts,
  c.value / 1024 AS value_kb,
  ct.name AS counter_name
FROM counter c
JOIN counter_track ct ON c.track_id = ct.id
WHERE ct.name LIKE '%mem%' OR ct.name LIKE '%heap%' OR ct.name LIKE '%rss%'
ORDER BY c.ts
LIMIT 500;
```

### 16. 内存 Counter 变化趋势

```sql
-- 按时间窗口聚合内存使用
SELECT
  (ts / 1e9) AS time_sec,
  AVG(value) / 1024 AS avg_kb
FROM counter c
JOIN counter_track ct ON c.track_id = ct.id
WHERE ct.name LIKE '%mem%'
GROUP BY time_sec
ORDER BY time_sec;
```

### 17. Heap 分配热点（需 heapprofd）

```sql
-- 分配最多的 callsite，按 frame 聚合（需 heap_profile_allocation 表）
SELECT
  COUNT(*) AS alloc_count,
  SUM(CASE WHEN h.size > 0 THEN h.size ELSE 0 END) AS total_alloc_bytes,
  f.name AS frame_name
FROM heap_profile_allocation h
JOIN stack_profile_callsite c ON h.callsite_id = c.id
JOIN stack_profile_frame f ON c.frame_id = f.id
GROUP BY f.name
ORDER BY total_alloc_bytes DESC
LIMIT 20;
```

---

## 通用查询

### 18. 查看某进程所有线程

```sql
-- 列出目标进程的所有线程
SELECT
  t.utid,
  t.name AS thread_name,
  t.tid,
  p.name AS process_name,
  p.pid
FROM thread t
JOIN process p ON t.upid = p.upid
WHERE p.name = 'com.example.yourapp'
ORDER BY t.name;
```

### 19. 按耗时排序所有 Slice

```sql
-- 全局耗时最长的 slice Top 50
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  s.category,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE s.dur > 0
ORDER BY s.dur DESC
LIMIT 50;
```

### 20. 查看某时间窗口内的所有事件

```sql
-- 指定时间范围（ns）内的 slice
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  t.name AS thread_name,
  p.name AS process_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE s.ts BETWEEN 60000000000 AND 65000000000  -- 替换为实际时间范围
ORDER BY s.ts;
```

### 21. performTraversals 及其子阶段耗时

```sql
-- 分析单帧的 measure/layout/draw 耗时分布
SELECT
  s.ts,
  s.dur / 1e6 AS dur_ms,
  s.name,
  s.depth,
  t.name AS thread_name
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
WHERE (s.name = 'performTraversals' OR s.name LIKE 'performMeasure%'
       OR s.name LIKE 'performLayout%' OR s.name LIKE 'performDraw%')
  AND s.dur > 0
ORDER BY s.ts, s.depth;
```

### 22. 主线程阻塞区间（thread_state）

```sql
-- 查找主线程非 Running 状态（如 D、S）的区间
SELECT
  ts,
  dur / 1e6 AS dur_ms,
  state,
  blocked_function,
  t.name AS thread_name,
  p.name AS process_name
FROM thread_state ts
JOIN thread t ON ts.utid = t.utid
JOIN process p ON t.upid = p.upid
WHERE t.name = 'main'
  AND state != 'R' AND state != 'Running'
  AND dur > 1000000  -- > 1ms
ORDER BY dur DESC;
```

### 23. Expected vs Actual Frame 对比（Android 12+）

```sql
-- 对比 expected 与 actual frame 时长
SELECT
  e.ts,
  e.dur / 1e6 AS expected_ms,
  a.dur / 1e6 AS actual_ms,
  (a.dur - e.dur) / 1e6 AS overflow_ms,
  a.jank_type,
  process.name
FROM expected_frame_timeline_slice e
JOIN actual_frame_timeline_slice a
  ON e.upid = a.upid AND e.surface_frame_token = a.surface_frame_token
LEFT JOIN process ON e.upid = process.upid
WHERE a.dur > e.dur
ORDER BY overflow_ms DESC;
```

---

## 使用说明

| 表名 | 说明 | 数据源要求 |
|------|------|------------|
| `slice` | atrace 产生的 slice | atrace、tracepoint |
| `thread_state` | 线程调度状态 | sched/sched_switch 等 |
| `sched` | CPU 调度切片 | sched/sched_switch |
| `actual_frame_timeline_slice` | 实际帧时间线 | android.surfaceflinger.frametimeline |
| `expected_frame_timeline_slice` | 预期帧时间线 | 同上 |
| `process`, `thread` | 进程/线程元数据 | 自动 |
| `counter` | 计数器（CPU 频率、内存等） | power、memory 等 |
| `heap_profile_allocation` | 堆分配 | heapprofd |

**抓取完整 trace 的 adb 命令示例**：

```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace.pb -t 10s \
  sched freq idle am wm gfx view binder_driver
```

若需 FrameTimeline（Android 12+），在 Perfetto UI 录制时勾选 **Frame Timeline**。
