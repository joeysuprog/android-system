# ADB 命令速查表

> Android Debug Bridge (ADB) 常用命令，按场景分类整理

---

## 系统信息

```bash
# 系统版本
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk

# 设备型号与 build 信息
adb shell getprop ro.product.model
adb shell getprop ro.build.display.id
adb shell getprop ro.build.fingerprint

# 内核版本
adb shell cat /proc/version
adb uname -a

# 屏幕分辨率与密度
adb shell wm size
adb shell wm density

# 电池状态
adb shell dumpsys battery
```

---

## 进程与服务

```bash
# 列出所有进程（含系统）
adb shell ps -A
adb shell ps -A | grep <keyword>

# 按包名查进程
adb shell pidof <package>

# 系统服务列表
adb shell service list

# dumpsys 可用服务列表
adb shell dumpsys -l

# Activity 相关
adb shell dumpsys activity
adb shell dumpsys activity activities    # 当前 Activity 栈
adb shell dumpsys activity processes     # 进程及其 Activity
adb shell dumpsys activity intents       # 待处理 Intent
adb shell dumpsys activity recents       # 最近任务

# 指定服务 dump
adb shell dumpsys <service_name>
```

---

## 窗口与显示

```bash
# 当前窗口列表
adb shell dumpsys window windows

# 显示设备信息
adb shell dumpsys window displays

# 当前焦点窗口
adb shell dumpsys window | grep -i "mCurrentFocus"

# SurfaceFlinger 相关信息
adb shell dumpsys SurfaceFlinger
adb shell dumpsys SurfaceFlinger --list   # Layer 列表
adb shell dumpsys SurfaceFlinger --latency <window>  # 帧延迟（需指定 window 名）

# 禁用/启用状态栏
adb shell cmd statusbar expand-notifications
adb shell cmd statusbar collapse
```

---

## 渲染调试

```bash
# 过度绘制可视化（需重启 App）
adb shell setprop debug.hwui.overdraw show
# 关闭：adb shell setprop debug.hwui.overdraw false

# 布局边界可视化
adb shell setprop debug.layout true
# 关闭：adb shell setprop debug.layout false

# GPU 渲染模式分析（需在开发者选项中开启）
adb shell dumpsys gfxinfo <package>
adb shell dumpsys gfxinfo <package> framestats   # 详细帧统计
adb shell dumpsys gfxinfo <package> reset         # 重置统计

# 当前 HWUI 渲染器
adb shell getprop debug.hwui.renderer
# 切换软件渲染（调试用）
adb shell setprop debug.hwui.renderer skiagl
```

---

## 性能分析

```bash
# Perfetto 抓 10 秒 trace
adb shell perfetto -o /data/misc/perfetto-traces/trace.pb -t 10s sched freq idle am wm gfx view binder_driver

# 拉取 trace 到本地
adb pull /data/misc/perfetto-traces/trace.pb ./

# 启动耗时测量（-W 等待启动完成）
adb shell am start -W <package>/<activity>
# 示例：adb shell am start -W com.example.app/.MainActivity

# 启动并带 extras
adb shell am start -n <package>/<activity> -a <action> -d <data>

# CPU 使用率（需先进入 shell）
adb shell top -n 1
adb shell top -p <pid>
```

---

## 内存分析

```bash
# 进程内存详情
adb shell dumpsys meminfo <package>
adb shell dumpsys meminfo <pid>

# 按内存排序（-s）
adb shell dumpsys meminfo -s

# 系统总内存
adb shell cat /proc/meminfo

# 抓取 Heap dump（需 debuggable 或 root）
adb shell am dumpheap <pid> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof ./

# 进程内存占用排行
adb shell procrank
```

---

## ANR 与调试

```bash
# 拉取 ANR traces
adb pull /data/anr/traces.txt ./

# 完整 bugreport（含 logs、traces、dumpsys 等）
adb bugreport
# 或指定输出：adb bugreport bugreport.zip

# 查看 ANR 相关进程
adb shell dumpsys activity processes | grep -i ANR

# 主动触发 ANR（测试用，慎用）
adb shell am hang
```

---

## SystemUI 与通知

```bash
# 状态栏相关
adb shell dumpsys statusbar

# 通知列表
adb shell dumpsys notification

# 展开/收起通知栏
adb shell cmd statusbar expand-notifications
adb shell cmd statusbar collapse

# 展开/收起快捷设置
adb shell cmd statusbar expand-settings

# 清除通知
adb shell cmd notification dismiss_all
```

---

## 应用控制

```bash
# 启动 Activity
adb shell am start -n <package>/<activity>
# 示例：adb shell am start -n com.android.settings/.Settings

# 启动 Service
adb shell am startservice -n <package>/<service>

# 发送 Broadcast
adb shell am broadcast -a <action> [-n <component>]

# 强制停止
adb shell am force-stop <package>

# 清除应用数据
adb shell pm clear <package>

# 包列表
adb shell pm list packages
adb shell pm list packages -3          # 仅第三方
adb shell pm list packages | grep <keyword>

# 包信息
adb shell pm dump <package>

# 安装/卸载
adb install -r <apk_path>
adb uninstall <package>
```

---

## 输入模拟

```bash
# 点击
adb shell input tap <x> <y>

# 滑动
adb shell input swipe <x1> <y1> <x2> <y2> [duration_ms]

# 文本输入
adb shell input text "hello"

# 按键
adb shell input keyevent <keycode>
# 常用：HOME=3, BACK=4, MENU=82, ENTER=66

# 长按
adb shell input swipe <x> <y> <x> <y> 500
```

---

## 日志

```bash
# 实时日志
adb logcat

# 按 tag 过滤
adb logcat -s <tag>

# 按级别过滤
adb logcat *:E    # 仅 Error
adb logcat *:W    # Warning 及以上

# 清空日志缓冲
adb logcat -c

# 输出到文件
adb logcat -d > logcat.txt
```

---

## 文件操作

```bash
# 推送到设备
adb push <local_path> <remote_path>

# 从设备拉取
adb pull <remote_path> [local_path]

# 列出文件
adb shell ls <path>
```

---

## 多设备

```bash
# 设备列表
adb devices

# 指定设备执行
adb -s <device_serial> shell ...
```
