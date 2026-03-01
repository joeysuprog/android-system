本文档库为 Android 系统知识体系的学习索引，按模块和推荐的学习顺序进行了梳理：

## 目录

### 1. AOSP环境与架构概览 (`aosp-env`)
*主要涵盖环境搭建及 AOSP 项目工程结构，是深入系统源码的第一步。*
- [环境与工具准备](./aosp-env/环境与工具准备.md)
- [AOSP工程架构全景](./aosp-env/AOSP工程架构全景.md)

### 2. 系统核心框架与底层原理 (`framework`)
*从 Linux 内核到 Android 系统启动层，以及核心 IPC、管理服务、运行环境等内容。*
- [Android_Linux内核原理](./framework/Android_Linux内核原理.md)
- [Android系统架构全景](./framework/Android系统架构全景.md)
- [Android系统启动流程](./framework/Android系统启动流程.md)
- [App启动流程](./framework/App启动流程.md)
- [Binder与跨进程通信](./framework/Binder与跨进程通信.md)
- [AMS与WMS核心服务](./framework/AMS与WMS核心服务.md)
- [JNI桥接机制](./framework/JNI桥接机制.md)
- [ART与Native协同机制](./framework/ART与Native协同机制.md)

### 3. UI与渲染框架原理 (`framework`)
*深入了解 Android 中 View 从绘制、HWUI 管线到最终交由 SurfaceFlinger 合成上屏的全链路原理。*
- [View绘制体系](./framework/View绘制体系.md)
- [HWUI渲染管线](./framework/HWUI渲染管线.md)
- [SurfaceFlinger合成](./framework/SurfaceFlinger合成.md)
- [VSync与帧调度](./framework/VSync与帧调度.md)
- [渲染全链路](./framework/渲染全链路.md)
- [Android动画框架原理](./framework/Android动画框架原理.md)

### 4. 系统UI与桌面架构 (`system-ui-app`)
*关于 Android 系统默认 UI 组件以及桌面的架构实现。*
- [SystemUI架构](./system-ui-app/SystemUI架构.md)
- [Launcher3架构](./system-ui-app/Launcher3架构.md)

### 5. 应用层组件与架构进阶 (`app`)
*基于系统框架，深入到应用开发者最常接触的组件底层原理。*
- [Activity与Fragment](./app/Activity与Fragment.md)
- [Service组件](./app/Service组件.md)
- [BroadcastReceiver](./app/BroadcastReceiver.md)
- [ContentProvider](./app/ContentProvider.md)
- [RecyclerView原理](./app/RecyclerView原理.md)
- [Kotlin协程与Flow](./app/Kotlin协程与Flow.md)

### 6. 跨平台UI与组件对比 (`multi-platform`)
*将 Android 原生的渲染体系与其他跨平台框架（如 Compose，Flutter）在相同场景下的实现进行横向对比分析。*
- [Compose渲染框架](./multi-platform/Compose渲染框架.md)
- [Flutter渲染框架](./multi-platform/Flutter渲染框架.md)
- [列表组件跨框架对比](./multi-platform/列表组件跨框架对比.md)
- [事件分发跨框架对比](./multi-platform/事件分发跨框架对比.md)
- [嵌套滚动跨框架对比:RecyclerView-Compose-Flutter](./multi-platform/嵌套滚动跨框架对比:RecyclerView-Compose-Flutter.md)

### 7. 音视频原理 (`av`)
*多媒体处理、音视频编解码与显示渲染的基础原理。*
- [音频编码原理](./av/音频编码原理.md)
- [视频编码原理](./av/视频编码原理.md)
- [SurfaceView视频显示原理](./av/SurfaceView视频显示原理.md)

### 8. 性能分析实战 (`perfetto`)
*系统性能优化的理论工具和真实场景分析实战。*
- [Perfetto使用指南](./perfetto/Perfetto使用指南.md)
- [AndroidStudioProfiler性能优化](./perfetto/AndroidStudioProfiler性能优化.md)
- [启动分析流程](./perfetto/启动分析流程.md)
- [掉帧分析方法](./perfetto/掉帧分析方法.md)