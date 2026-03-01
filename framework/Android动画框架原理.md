# Android 动画框架原理深度解析

> 本文基于 Android 14 最新架构，系统性介绍 Android 动画框架的核心原理与关键组件。
>
> 建议先了解 Android 渲染机制。

---

## 目录

1. [概述与背景](#1-概述与背景)
2. [动画的底层驱动 —— Choreographer 与 Vsync](#2-动画的底层驱动--choreographer-与-vsync)
3. [View 动画（Animation）](#3-view-动画animation)
4. [属性动画（Property Animation）](#4-属性动画property-animation)
5. [TimeInterpolator —— 时间插值器](#5-timeinterpolator--时间插值器)
6. [Transition 框架 —— 转场动画](#6-transition-框架--转场动画)
7. [物理动画（Physics-based Animation）](#7-物理动画physics-based-animation)
8. [Compose 动画系统](#8-compose-动画系统)
9. [动画性能优化](#9-动画性能优化)
10. [总结](#10-总结)

---

## 1. 概述与背景

### 1.1 动画的本质

动画的本质是：**在时间轴上连续改变视觉属性，利用人眼的视觉暂留效应，制造出运动的错觉**。

就像电影胶片一样，每一帧都是一张静止的画面，但当它们以足够快的速度连续播放时，大脑就会将其感知为连续的运动。在 Android 中，动画框架的核心任务就是：**在每个 Vsync 信号到达时，计算出当前时刻的属性值，交给渲染管线绘制到屏幕上**。

```
┌─────────────────────────────────────────────────────────────────┐
│                    动画的本质                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  时间轴 →                                                       │
│                                                                  │
│  帧 1        帧 2        帧 3        帧 4        帧 5          │
│  ┌──┐       ┌──┐       ┌──┐       ┌──┐       ┌──┐            │
│  │  │       │  │       │  │       │  │       │  │            │
│  │  │       │  │→      │    │→    │      │→  │        │→    │
│  │  │       │  │       │  │       │  │       │  │            │
│  └──┘       └──┘       └──┘       └──┘       └──┘            │
│  x=0        x=25       x=50       x=75       x=100            │
│                                                                  │
│  每帧改变属性值，人眼感知为连续运动                            │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  动画 = f(时间)                                                 │
│                                                                  │
│  当前值 = evaluate(interpolate(elapsed / duration),             │
│                    startValue,                                   │
│                    endValue)                                     │
│                                                                  │
│  三要素：                                                       │
│  1. 时间进度 (fraction)     ─ 当前时刻在总时长中的比例         │
│  2. 插值函数 (interpolator) ─ 控制动画的节奏感                 │
│  3. 值计算器 (evaluator)    ─ 根据进度计算实际属性值           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 为什么需要动画框架


| 需求          | 说明                                    |
| ----------- | ------------------------------------- |
| **视觉连续性**   | 人眼需要至少 24fps 感知连续运动，60fps/120fps 才够流畅 |
| **物理世界隐喻**  | 现实世界中物体不会瞬移，动画让界面变化符合物理直觉             |
| **用户注意力引导** | 通过运动引导用户关注重要的 UI 变化                   |
| **状态转换可感知** | 帮助用户理解界面前后状态的关系（从哪来、到哪去）              |
| **统一帧调度**   | 所有动画需要与 Vsync 同步，避免各自为政导致的撕裂和卡顿       |


### 1.3 Android 动画框架演进历史

```
┌─────────────────────────────────────────────────────────────────┐
│                 Android 动画框架演进历史                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Android 1.0 (2008)     View Animation (补间动画)               │
│  ──────────────────     ─────────────────────────               │
│                         • 4种变换：平移/缩放/旋转/透明度        │
│                         • 只改变绘制效果，不改变 View 属性       │
│                         • AnimationDrawable (帧动画)             │
│                                                                  │
│          │                                                       │
│          ↓                                                       │
│                                                                  │
│  Android 3.0 (2011)     Property Animation (属性动画)           │
│  ──────────────────     ─────────────────────────               │
│                         • ValueAnimator / ObjectAnimator         │
│                         • 真正改变对象属性                       │
│                         • TypeEvaluator / TimeInterpolator       │
│                         • AnimatorSet 动画编排                   │
│                                                                  │
│          │                                                       │
│          ↓                                                       │
│                                                                  │
│  Android 4.4 (2013)     Transition Framework (转场框架)         │
│  ──────────────────     ─────────────────────────               │
│                         • Scene + Transition                     │
│                         • 自动捕获前后状态差异                   │
│                         • Activity/Fragment 转场动画             │
│                         • SharedElement 共享元素                  │
│                                                                  │
│          │                                                       │
│          ↓                                                       │
│                                                                  │
│  AndroidX (2018)        Physics-based Animation (物理动画)      │
│  ──────────────────     ─────────────────────────               │
│                         • SpringAnimation (弹簧动画)             │
│                         • FlingAnimation (惯性滑动)              │
│                         • 基于力学模型，可中断、速度连续         │
│                                                                  │
│          │                                                       │
│          ↓                                                       │
│                                                                  │
│  Jetpack Compose (2021) Compose Animation (声明式动画)          │
│  ──────────────────     ─────────────────────────               │
│                         • 状态驱动，声明式描述                   │
│                         • animate*AsState / AnimatedVisibility   │
│                         • Transition API 多值协调                │
│                         • 底层仍由 Choreographer 驱动            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 动画框架全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                 Android 动画框架全景图                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    应用层 (API)                             │ │
│  │                                                            │ │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────────┐│ │
│  │  │View 动画   │ │Transition  │ │  Compose Animation     ││ │
│  │  │            │ │Framework   │ │                        ││ │
│  │  │ Translate  │ │            │ │ AnimatedVisibility     ││ │
│  │  │ Scale      │ │ Fade       │ │ animate*AsState        ││ │
│  │  │ Rotate     │ │ ChangeBnds │ │ Transition API         ││ │
│  │  │ Alpha      │ │ Slide      │ │ Animatable             ││ │
│  │  │ AnimSet    │ │ SharedElem │ │                        ││ │
│  │  └────────────┘ └─────┬──────┘ └───────────┬────────────┘│ │
│  │        │               │                    │             │ │
│  └────────│───────────────│────────────────────│─────────────┘ │
│           │               │                    │               │
│  ┌────────│───────────────│────────────────────│─────────────┐ │
│  │        ↓               ↓                    ↓              │ │
│  │                  核心引擎层                                │ │
│  │                                                            │ │
│  │  ┌─────────────────────────────────────────────────────┐  │ │
│  │  │              Property Animation                      │  │ │
│  │  │                                                      │  │ │
│  │  │  ValueAnimator ← TimeInterpolator + TypeEvaluator   │  │ │
│  │  │  ObjectAnimator ← PropertyValuesHolder              │  │ │
│  │  │  AnimatorSet ← AnimationEvent 有向图                │  │ │
│  │  └─────────────────────────────────────────────────────┘  │ │
│  │                                                            │ │
│  │  ┌──────────────────┐    ┌──────────────────────────┐    │ │
│  │  │  Physics Engine  │    │  ViewPropertyAnimator    │    │ │
│  │  │                  │    │                          │    │ │
│  │  │  SpringForce     │    │  批量属性 + 自动硬件层  │    │ │
│  │  │  FlingAnimation  │    │  直接操作 RenderNode     │    │ │
│  │  └──────────────────┘    └──────────────────────────┘    │ │
│  │                                                            │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                 │
│  ┌────────────────────────────│───────────────────────────────┐ │
│  │                            ↓                                │ │
│  │                  帧调度层                                   │ │
│  │                                                             │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │              AnimationHandler                        │   │ │
│  │  │     (管理所有活跃的 ValueAnimator 实例)              │   │ │
│  │  └──────────────────────┬──────────────────────────────┘   │ │
│  │                         │ postFrameCallback                 │ │
│  │                         ↓                                   │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │              Choreographer                           │   │ │
│  │  │     (CALLBACK_ANIMATION / postFrameCallback)        │   │ │
│  │  └──────────────────────┬──────────────────────────────┘   │ │
│  │                         │ scheduleVsync                     │ │
│  │                         ↓                                   │ │
│  │  ┌─────────────────────────────────────────────────────┐   │ │
│  │  │          Vsync 信号 (来自 SurfaceFlinger)            │   │ │
│  │  └─────────────────────────────────────────────────────┘   │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 动画的底层驱动 —— Choreographer 与 Vsync

### 2.1 回顾：Choreographer 的角色

在 [Android渲染原理分享](Android渲染原理分享.md) 中，我们了解到 Choreographer 管理五种回调，按固定顺序执行：

```
Vsync 到达 → doFrame()

  1. CALLBACK_INPUT          ← 输入事件
  2. CALLBACK_ANIMATION      ← ★ 动画更新（本文核心）
  3. CALLBACK_INSETS_ANIMATION ← 窗口边界动画
  4. CALLBACK_TRAVERSAL      ← Measure / Layout / Draw
  5. CALLBACK_COMMIT         ← 提交完成
```

动画回调排在输入之后、视图遍历之前，这个顺序至关重要：**先更新动画值，再让 View 树基于新值进行 Measure/Layout/Draw**。

### 2.2 AnimationHandler：动画的中央调度器

`AnimationHandler` 是连接 `ValueAnimator` 和 `Choreographer` 的桥梁。每个线程有且仅有一个 `AnimationHandler` 实例（ThreadLocal），负责管理该线程上所有活跃的动画。

```
┌─────────────────────────────────────────────────────────────────┐
│              AnimationHandler 核心调度流程                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ValueAnimator.start()                                          │
│        │                                                         │
│        │ addAnimationFrameCallback(this)                        │
│        ↓                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  AnimationHandler                         │   │
│  │                                                           │   │
│  │  mAnimationCallbacks: ArrayList<AnimationFrameCallback>  │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │   │
│  │  │Animator 1│ │Animator 2│ │Animator 3│  ...             │   │
│  │  └──────────┘ └──────────┘ └──────────┘                 │   │
│  │                                                           │   │
│  │  当列表非空时，注册 FrameCallback 到 Choreographer       │   │
│  │                                                           │   │
│  └────────────────────────┬──────────────────────────────────┘   │
│                           │                                      │
│                           │ Choreographer.postFrameCallback()    │
│                           ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Choreographer                            │   │
│  │                                                           │   │
│  │  收到 FrameCallback → 申请 Vsync                        │   │
│  │                                                           │   │
│  └────────────────────────┬──────────────────────────────────┘   │
│                           │                                      │
│                           │ Vsync 到达                           │
│                           ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  AnimationHandler.mFrameCallback.doFrame(frameTimeNanos) │   │
│  │                                                           │   │
│  │  遍历 mAnimationCallbacks:                               │   │
│  │    for (callback in mAnimationCallbacks) {                │   │
│  │        callback.doAnimationFrame(frameTime)  ← 各 Animator│   │
│  │    }                                                      │   │
│  │                                                           │   │
│  │  如果还有活跃动画 → 再次 postFrameCallback → 下一个 Vsync │   │
│  │  如果全部结束 → 停止请求                                 │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 关键源码解析

```java
// AnimationHandler 核心逻辑（精简）
class AnimationHandler {

    // 每个线程一个实例
    private static final ThreadLocal<AnimationHandler> sAnimatorHandler = new ThreadLocal<>();

    // 所有活跃动画的回调列表
    private final ArrayList<AnimationFrameCallback> mAnimationCallbacks = new ArrayList<>();

    // 注册到 Choreographer 的帧回调
    private final Choreographer.FrameCallback mFrameCallback = frameTimeNanos -> {
        doAnimationFrame(getProvider().getFrameTime());
        // 如果还有活跃动画，继续请求下一帧
        if (mAnimationCallbacks.size() > 0) {
            getProvider().postFrameCallback(mFrameCallback);
        }
    };

    // 添加动画回调
    public void addAnimationFrameCallback(AnimationFrameCallback callback) {
        if (mAnimationCallbacks.size() == 0) {
            // 第一个动画注册时，开始请求 Vsync
            getProvider().postFrameCallback(mFrameCallback);
        }
        mAnimationCallbacks.add(callback);
    }

    // Vsync 到达时执行
    private void doAnimationFrame(long frameTime) {
        for (int i = 0; i < mAnimationCallbacks.size(); i++) {
            AnimationFrameCallback callback = mAnimationCallbacks.get(i);
            callback.doAnimationFrame(frameTime);
        }
        // 清理已结束的动画
        cleanUpList();
    }
}
```

### 2.4 动画在渲染管线中的时序

```
┌─────────────────────────────────────────────────────────────────┐
│              动画在一帧渲染中的位置                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Vsync 信号到达                                                 │
│  │                                                               │
│  │  ① CALLBACK_INPUT: 处理触摸事件                              │
│  │     │                                                         │
│  │     ↓                                                         │
│  │  ② CALLBACK_ANIMATION: ★ 动画帧更新                         │
│  │     │                                                         │
│  │     │  AnimationHandler.doAnimationFrame()                   │
│  │     │    ├─ Animator1.doAnimationFrame(frameTime)            │
│  │     │    │    ├─ 计算 fraction = elapsed / duration          │
│  │     │    │    ├─ 应用 interpolator                           │
│  │     │    │    ├─ 计算新值 = evaluator.evaluate(fraction)     │
│  │     │    │    └─ 通知 UpdateListener → 可能触发 invalidate  │
│  │     │    ├─ Animator2.doAnimationFrame(frameTime)            │
│  │     │    └─ ...                                              │
│  │     │                                                         │
│  │     ↓                                                         │
│  │  ③ CALLBACK_INSETS_ANIMATION: 键盘/导航栏动画               │
│  │     │                                                         │
│  │     ↓                                                         │
│  │  ④ CALLBACK_TRAVERSAL: 视图遍历                              │
│  │     │                                                         │
│  │     │  performMeasure()  ← 基于动画更新后的属性值            │
│  │     │  performLayout()                                       │
│  │     │  performDraw()     → 构建 DisplayList                  │
│  │     │                                                         │
│  │     ↓                                                         │
│  │  ⑤ sync → RenderThread → GPU → Display                      │
│  │                                                               │
│  ↓                                                               │
│  下一个 Vsync（如果动画未结束，继续循环）                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 动画帧率与显示刷新率

动画的帧率受 Vsync 频率约束：


| 刷新率   | Vsync 周期 | 动画帧间隔   | 说明               |
| ----- | -------- | ------- | ---------------- |
| 60Hz  | 16.67ms  | 16.67ms | 传统设备，每帧预算约 16ms  |
| 90Hz  | 11.11ms  | 11.11ms | 部分中端设备           |
| 120Hz | 8.33ms   | 8.33ms  | 高端设备，动画更流畅但帧预算更紧 |


动画框架本身不控制帧率——它只是在每个 Vsync 到达时被回调，计算当前时刻对应的属性值。帧率由硬件和系统决定。

---

## 3. View 动画（Animation）

### 3.1 概述

View 动画是 Android 最早的动画系统（API 1），也被称为**补间动画（Tween Animation）**。它的核心思想是：**不改变 View 的实际属性，只在绘制时对 Canvas 施加矩阵变换**。

### 3.2 四种基本变换

```
┌─────────────────────────────────────────────────────────────────┐
│                 View 动画四种基本变换                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────┐                 │
│  │ TranslateAnimation│     │  ScaleAnimation  │                 │
│  │                  │     │                  │                 │
│  │  ┌──┐    ┌──┐   │     │  ┌──┐    ┌────┐ │                 │
│  │  │  │ →  │  │   │     │  │  │ →  │    │ │                 │
│  │  └──┘    └──┘   │     │  └──┘    │    │ │                 │
│  │  (dx, dy)       │     │         └────┘ │                 │
│  └──────────────────┘     │  (sx, sy)      │                 │
│                           └──────────────────┘                 │
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────┐                 │
│  │ RotateAnimation  │     │  AlphaAnimation  │                 │
│  │                  │     │                  │                 │
│  │  ┌──┐    ╱╲     │     │  ┌──┐    ┌──┐   │                 │
│  │  │  │ →  ╲╱     │     │  │██│ →  │░░│   │                 │
│  │  └──┘           │     │  └──┘    └──┘   │                 │
│  │  (pivot, angle) │     │  (alpha 0→1)    │                 │
│  └──────────────────┘     └──────────────────┘                 │
│                                                                  │
│  AnimationSet: 可以组合多种变换同时执行或顺序执行              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 核心原理：矩阵变换

View 动画的核心是 `Transformation` 类，内部包含一个 `Matrix`（变换矩阵）和一个 `alpha` 值：

```java
// View 动画的核心计算链
class Animation {
    // 每帧被调用，根据当前时间计算变换
    boolean getTransformation(long currentTime, Transformation outTransformation) {
        // 1. 计算时间进度
        float normalizedTime = (currentTime - mStartTime) / mDuration;
        normalizedTime = Math.max(Math.min(normalizedTime, 1.0f), 0.0f);

        // 2. 应用插值器
        float interpolatedTime = mInterpolator.getInterpolation(normalizedTime);

        // 3. 子类实现具体变换（填充 Matrix 和 alpha）
        applyTransformation(interpolatedTime, outTransformation);
        return true;
    }
}

// TranslateAnimation 的实现
class TranslateAnimation extends Animation {
    void applyTransformation(float interpolatedTime, Transformation t) {
        float dx = mFromXDelta + (mToXDelta - mFromXDelta) * interpolatedTime;
        float dy = mFromYDelta + (mToYDelta - mFromYDelta) * interpolatedTime;
        // 直接设置矩阵的平移分量
        t.getMatrix().setTranslate(dx, dy);
    }
}
```

绘制时，`ViewGroup.drawChild()` 会获取子 View 的动画变换，应用到 Canvas 上：

```java
// ViewGroup 绘制子 View 时应用动画
boolean drawChild(Canvas canvas, View child, long drawingTime) {
    Animation animation = child.getAnimation();
    if (animation != null) {
        Transformation transformToApply = new Transformation();
        boolean more = animation.getTransformation(drawingTime, transformToApply);

        // ★ 对 Canvas 施加矩阵变换，而非改变 View 属性
        canvas.concat(transformToApply.getMatrix());
        canvas.setAlpha(transformToApply.getAlpha());

        child.draw(canvas);

        if (more) {
            invalidate(); // 动画未结束，请求下一帧
        }
    }
}
```

### 3.4 View 动画的局限性

```
┌─────────────────────────────────────────────────────────────────┐
│                 View 动画的致命缺陷                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  问题：点击区域不随动画移动                                     │
│  ────────────────────────                                       │
│                                                                  │
│  动画前：                    动画后（TranslateAnimation）：     │
│                                                                  │
│  ┌────────┐                  ┌────────┐                         │
│  │ Button │ ← 可点击         │(空白)  │ ← 可点击（实际属性位置）│
│  └────────┘                  └────────┘                         │
│                                         ┌────────┐             │
│                                         │ Button │ ← 不可点击  │
│                                         └────────┘  （仅视觉） │
│                                                                  │
│  原因：View 的 left/top/right/bottom 没有改变                  │
│       只是绘制时对 Canvas 施加了矩阵变换                       │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  其他局限：                                                     │
│  • 只能作用于 View 对象，不能给任意属性（如颜色）做动画        │
│  • 只支持 4 种固定变换，无法自定义属性                          │
│  • 无法获取动画过程中的中间值                                   │
│                                                                  │
│  ★ 这些局限催生了 Android 3.0 的属性动画框架                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 帧动画（AnimationDrawable）

帧动画是另一种完全不同的机制——逐帧切换 Drawable 资源，类似 GIF 动画：

```xml
<!-- drawable/frame_animation.xml -->
<animation-list android:oneshot="false">
    <item android:drawable="@drawable/frame_01" android:duration="100" />
    <item android:drawable="@drawable/frame_02" android:duration="100" />
    <item android:drawable="@drawable/frame_03" android:duration="100" />
</animation-list>
```

帧动画通过 `Handler.postDelayed()` 定时切换当前帧，与 Vsync 无直接关联。优点是实现简单，缺点是内存消耗大（每帧一张位图），适用于简单的小型动画。

---

## 4. 属性动画（Property Animation）

### 4.1 架构概览

属性动画是 Android 3.0 引入的全新动画框架，它真正改变对象的属性值，是目前最核心、使用最广泛的动画系统。

```
┌─────────────────────────────────────────────────────────────────┐
│                 属性动画核心架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Animator (抽象基类)                    │   │
│  │                                                          │   │
│  │  • start() / cancel() / end()                           │   │
│  │  • setDuration() / setStartDelay()                      │   │
│  │  • addListener(AnimatorListener)                        │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                        │                                        │
│           ┌────────────┴────────────┐                          │
│           ↓                         ↓                          │
│  ┌─────────────────┐     ┌──────────────────┐                 │
│  │  ValueAnimator  │     │   AnimatorSet    │                 │
│  │                 │     │                  │                 │
│  │ 值计算引擎      │     │ 动画编排容器     │                 │
│  │                 │     │                  │                 │
│  │ TimeInterpolator│     │ playTogether()  │                 │
│  │ TypeEvaluator   │     │ playSequentially│                 │
│  │ PropertyValues  │     │ before() after()│                 │
│  │  Holder         │     │                  │                 │
│  └────────┬────────┘     └──────────────────┘                 │
│           │                                                    │
│           ↓                                                    │
│  ┌─────────────────┐                                          │
│  │ ObjectAnimator  │                                          │
│  │                 │                                          │
│  │ 自动应用属性值  │                                          │
│  │ 反射/setter调用 │                                          │
│  └─────────────────┘                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 ValueAnimator：值计算引擎

`ValueAnimator` 是属性动画的核心，它的职责是：**在指定时间内，根据插值器和计算器，生成从起始值到结束值的连续过渡值**。

#### 4.2.1 启动流程

```java
// ValueAnimator.start() 精简流程
void start() {
    // 1. 初始化属性值持有者
    if (mValues == null) {
        setValues(PropertyValuesHolder.ofFloat("", mFloatValues));
    }

    // 2. 记录动画开始时间
    mStartTime = AnimationUtils.currentAnimationTimeMillis();

    // 3. 注册到 AnimationHandler（核心！）
    AnimationHandler.getInstance().addAnimationFrameCallback(this);
    // → 这会触发 Choreographer.postFrameCallback
    // → 下一个 Vsync 到达时，doAnimationFrame 被调用

    // 4. 通知监听器
    notifyStartListeners();
}
```

#### 4.2.2 帧更新流程

```
┌─────────────────────────────────────────────────────────────────┐
│            ValueAnimator 单帧计算流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Vsync 到达 → AnimationHandler.doAnimationFrame(frameTime)     │
│                    │                                             │
│                    ↓                                             │
│  ValueAnimator.doAnimationFrame(frameTime)                      │
│                    │                                             │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  animateBasedOnTime(currentTime)                          │  │
│  │                                                           │  │
│  │  ① 计算原始时间比例                                      │  │
│  │     fraction = (currentTime - startTime) / duration       │  │
│  │     例如：300ms / 1000ms = 0.3                            │  │
│  │                                                           │  │
│  │  ② 处理 RepeatMode                                       │  │
│  │     RESTART: fraction 直接使用                            │  │
│  │     REVERSE: 偶数轮正向，奇数轮取 1 - fraction           │  │
│  │                                                           │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │                                       │
│                         ↓                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  animateValue(fraction)                                   │  │
│  │                                                           │  │
│  │  ③ 应用插值器                                            │  │
│  │     fraction = interpolator.getInterpolation(fraction)    │  │
│  │     例如：AccelerateDecelerate(0.3) → 0.191              │  │
│  │                                                           │  │
│  │  ④ 通过 PropertyValuesHolder 计算属性值                  │  │
│  │     for (PropertyValuesHolder pvh : mValues) {            │  │
│  │         pvh.calculateValue(fraction);                     │  │
│  │         // 内部调用 TypeEvaluator.evaluate()              │  │
│  │         // 例如：0 + (100 - 0) * 0.191 = 19.1            │  │
│  │     }                                                     │  │
│  │                                                           │  │
│  │  ⑤ 通知 UpdateListener                                   │  │
│  │     for (listener : mUpdateListeners) {                   │  │
│  │         listener.onAnimationUpdate(this);                 │  │
│  │     }                                                     │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  计算链总结：                                                   │
│  时间 → fraction → interpolator → evaluator → 最终值          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 ObjectAnimator：自动属性应用

`ObjectAnimator` 继承自 `ValueAnimator`，增加了**自动将计算值应用到目标对象属性**的能力：

```java
// ObjectAnimator 的核心扩展
class ObjectAnimator extends ValueAnimator {
    private Object mTarget;        // 目标对象
    private String mPropertyName;  // 属性名

    @Override
    void animateValue(float fraction) {
        // 1. 调用父类计算值（插值 + 求值）
        super.animateValue(fraction);

        // 2. ★ 将值应用到目标对象
        for (PropertyValuesHolder pvh : mValues) {
            pvh.setAnimatedValue(mTarget);
        }
    }
}

// PropertyValuesHolder 通过反射设置属性
class PropertyValuesHolder {
    void setAnimatedValue(Object target) {
        if (mProperty != null) {
            // 优先使用 Property 对象（避免反射）
            mProperty.set(target, getAnimatedValue());
        } else {
            // 通过反射调用 setter
            mSetter.invoke(target, getAnimatedValue());
        }
    }

    // setter 方法查找规则
    void setupSetter(Class targetClass) {
        // 属性名 "alpha" → 查找 setAlpha(float) 方法
        String methodName = "set" + capitalize(mPropertyName);
        mSetter = targetClass.getMethod(methodName, mValueType);
    }
}
```

#### 使用示例与反射调用链

```java
// 使用方式
ObjectAnimator.ofFloat(view, "translationX", 0f, 200f).start();

// 内部调用链：
// 1. animateValue(fraction)
// 2. PropertyValuesHolder.calculateValue(fraction)
//    → FloatEvaluator.evaluate(fraction, 0f, 200f) → 当前值
// 3. PropertyValuesHolder.setAnimatedValue(view)
//    → View.setTranslationX(当前值)  ← 反射调用
//    → View.invalidate()             ← setter 内部触发重绘
```

### 4.4 AnimatorSet：动画编排

`AnimatorSet` 允许将多个动画组合成复杂的编排序列：

```
┌─────────────────────────────────────────────────────────────────┐
│                 AnimatorSet 编排模型                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  方式一：同时执行                                               │
│  ────────────────                                               │
│  animSet.playTogether(animA, animB, animC)                      │
│                                                                  │
│  时间 →                                                         │
│  animA: [████████████████████]                                  │
│  animB: [████████████████████]                                  │
│  animC: [████████████████████]                                  │
│                                                                  │
│  方式二：顺序执行                                               │
│  ────────────────                                               │
│  animSet.playSequentially(animA, animB, animC)                  │
│                                                                  │
│  时间 →                                                         │
│  animA: [██████]                                                │
│  animB:        [██████]                                         │
│  animC:               [██████]                                  │
│                                                                  │
│  方式三：依赖关系 (有向图)                                      │
│  ────────────────                                               │
│  animSet.play(animB).after(animA)                               │
│  animSet.play(animC).after(animA)                               │
│  animSet.play(animD).after(animB).after(animC)                  │
│                                                                  │
│         ┌──────┐                                                │
│         │animA │                                                │
│         └──┬───┘                                                │
│        ┌───┴───┐                                                │
│        ↓       ↓                                                │
│   ┌──────┐ ┌──────┐                                            │
│   │animB │ │animC │                                            │
│   └──┬───┘ └───┬──┘                                            │
│      └────┬────┘                                                │
│           ↓                                                      │
│      ┌──────┐                                                   │
│      │animD │                                                   │
│      └──────┘                                                   │
│                                                                  │
│  内部实现：构建 AnimationEvent 有向图                           │
│  每帧检查依赖关系，满足条件的节点才启动                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 TypeEvaluator：值的插值计算

`TypeEvaluator` 决定了如何根据进度计算中间值：

```java
// 内置 Evaluator
public class FloatEvaluator implements TypeEvaluator<Float> {
    public Float evaluate(float fraction, Float startValue, Float endValue) {
        return startValue + fraction * (endValue - startValue);
    }
}

public class IntEvaluator implements TypeEvaluator<Integer> {
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        return (int)(startValue + fraction * (endValue - startValue));
    }
}

// 颜色插值：在 HSL 空间中对各通道分别插值
public class ArgbEvaluator implements TypeEvaluator<Integer> {
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startA = (startValue >> 24) & 0xff;
        int startR = (startValue >> 16) & 0xff;
        int startG = (startValue >>  8) & 0xff;
        int startB =  startValue        & 0xff;

        int endA = (endValue >> 24) & 0xff;
        // ... 各通道分别线性插值
        return (startA + (int)(fraction * (endA - startA))) << 24
             | (startR + (int)(fraction * (endR - startR))) << 16
             | (startG + (int)(fraction * (endG - startG))) << 8
             | (startB + (int)(fraction * (endB - startB)));
    }
}

// 自定义 Evaluator 示例：PointF 插值
public class PointFEvaluator implements TypeEvaluator<PointF> {
    public PointF evaluate(float fraction, PointF start, PointF end) {
        return new PointF(
            start.x + fraction * (end.x - start.x),
            start.y + fraction * (end.y - start.y)
        );
    }
}
```

### 4.6 完整计算链总结

```
┌─────────────────────────────────────────────────────────────────┐
│            属性动画完整计算链                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Vsync 信号                                                     │
│     │                                                            │
│     ↓                                                            │
│  frameTime (当前帧时间戳)                                       │
│     │                                                            │
│     ↓                                                            │
│  fraction = (frameTime - startTime) / duration                  │
│     │        例：300ms / 1000ms = 0.3                            │
│     ↓                                                            │
│  interpolatedFraction = interpolator.getInterpolation(0.3)      │
│     │        例：AccelerateDecelerate → 0.191                   │
│     ↓                                                            │
│  value = evaluator.evaluate(0.191, startValue, endValue)        │
│     │        例：FloatEvaluator(0.191, 0, 100) → 19.1          │
│     ↓                                                            │
│  ObjectAnimator: target.setProperty(19.1)                       │
│     │        例：view.setTranslationX(19.1f)                    │
│     ↓                                                            │
│  View.invalidate() → 等待本帧 CALLBACK_TRAVERSAL 重绘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. TimeInterpolator —— 时间插值器

### 5.1 插值器的数学本质

插值器是一个数学函数 `f: [0,1] → R`，它将**均匀流逝的时间进度**映射为**非均匀的动画进度**，从而创造出加速、减速、弹跳等丰富的运动效果。

```
┌─────────────────────────────────────────────────────────────────┐
│                 插值器的本质                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  输入：时间进度 t ∈ [0, 1]（均匀递增）                         │
│  输出：动画进度 f(t)（可以非均匀，甚至超出 [0,1]）            │
│                                                                  │
│  均匀时间：  0.0  0.1  0.2  0.3  0.4  0.5  0.6  0.7  0.8  0.9 │
│             ──── ──── ──── ──── ──── ──── ──── ──── ──── ────  │
│                                                                  │
│  线性：     0.0  0.1  0.2  0.3  0.4  0.5  0.6  0.7  0.8  0.9  │
│  加速：     0.01 0.04 0.09 0.16 0.25 0.36 0.49 0.64 0.81 →快  │
│  减速：     0.19 0.36 0.51 0.64 0.75 0.84 0.91 0.96 0.99 →慢  │
│  先加后减：  0.02 0.10 0.19 0.35 0.50 0.65 0.81 0.90 0.98 ─╮  │
│                                                       回正→ ╯  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 常用插值器详解

#### LinearInterpolator（线性）

```
f(t) = t

进度 │          ╱
1.0  │        ╱
     │      ╱
     │    ╱
     │  ╱
0.0  │╱─────────────
     0.0          1.0  → 时间

匀速运动，无加速无减速
```

#### AccelerateInterpolator（加速）

```
f(t) = t^factor  (默认 factor = 1.0 → t²)

进度 │              ╱
1.0  │            ╱
     │          ╱
     │        ╱
     │     ╱╱
0.0  │╱╱──────────────
     0.0          1.0  → 时间

开始慢、结束快，模拟物体从静止开始下落
```

#### DecelerateInterpolator（减速）

```
f(t) = 1 - (1 - t)^factor  (默认 factor = 1.0)

进度 │      ────────
1.0  │   ╱╱
     │  ╱
     │ ╱
     │╱
0.0  │─────────────
     0.0          1.0  → 时间

开始快、结束慢，模拟物体被抛出后的减速
```

#### AccelerateDecelerateInterpolator（先加速后减速）

```
f(t) = (cos((t + 1) * π) / 2.0) + 0.5

进度 │        ╱──╲
1.0  │      ╱╱    ╲
     │    ╱╱       ╲
     │  ╱╱          ╲
     │╱╱             ╲
0.0  │────────────────
     0.0          1.0  → 时间

S 曲线，最自然的运动效果（Android 默认插值器）
```

#### OvershootInterpolator（超越回弹）

```
f(t) = (t - 1)² * ((tension + 1) * (t - 1) + tension) + 1
默认 tension = 2.0

进度 │     ╱╲
1.1  │    ╱  ╲         ← 超过终点
1.0  │───╱────╲────────
     │  ╱      ╲╱
     │ ╱
0.0  │╱────────────────
     0.0          1.0  → 时间

超过目标值后回弹到终点，模拟弹性效果
```

#### BounceInterpolator（弹跳）

```
模拟物体落地反弹

进度 │     ╱╲
1.0  │    ╱  ╲    ╱╲   ╱╲ ╱
     │   ╱    ╲  ╱  ╲ ╱  ╲╱
     │  ╱      ╲╱    ╲╱
     │ ╱
0.0  │╱──────────────────────
     0.0                  1.0  → 时间

模拟弹球效果，多次反弹衰减
```

#### PathInterpolator（路径/贝塞尔曲线）

```java
// 三次贝塞尔曲线插值器
PathInterpolator(float controlX1, float controlY1,
                 float controlX2, float controlY2)

// Material Design 标准曲线
PathInterpolator(0.4f, 0.0f, 0.2f, 1.0f)  // Standard easing
PathInterpolator(0.0f, 0.0f, 0.2f, 1.0f)  // Decelerate
PathInterpolator(0.4f, 0.0f, 1.0f, 1.0f)  // Accelerate
```

### 5.3 插值器在计算链中的位置

```
┌─────────────────────────────────────────────────────────────────┐
│           插值器在完整计算链中的位置                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  elapsed    fraction   interpolator       evaluator    最终值   │
│  ────────   ────────   ────────────       ─────────    ──────   │
│                                                                  │
│  300ms   →  0.30    →  AccDecel(0.30)  →  Float(      → 19.1  │
│  1000ms     /1000ms     = 0.191          0.191,0,100)          │
│                                                                  │
│  500ms   →  0.50    →  AccDecel(0.50)  →  Float(      → 50.0  │
│  1000ms     /1000ms     = 0.500          0.500,0,100)          │
│                                                                  │
│  700ms   →  0.70    →  AccDecel(0.70)  →  Float(      → 80.9  │
│  1000ms     /1000ms     = 0.809          0.809,0,100)          │
│                                                                  │
│  分工：                                                         │
│  Interpolator: 控制"节奏"（何时快、何时慢）                   │
│  Evaluator:    控制"值域"（如何从 A 变到 B）                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Transition 框架 —— 转场动画

### 6.1 设计理念

Transition 框架（Android 4.4 引入）的核心理念是：**开发者只需描述布局的前后两个状态（Scene），框架自动计算差异并生成属性动画**。

```
┌─────────────────────────────────────────────────────────────────┐
│                Transition 框架的核心思想                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  传统方式（命令式）：                                           │
│  ─────────────────                                              │
│  // 开发者需要手动创建每个动画                                  │
│  ObjectAnimator.ofFloat(view, "x", oldX, newX).start()          │
│  ObjectAnimator.ofFloat(view, "alpha", 0f, 1f).start()          │
│  // ... 手动管理每个属性的变化                                  │
│                                                                  │
│  Transition 方式（声明式）：                                    │
│  ─────────────────────                                          │
│  // 开发者只描述期望的最终状态                                  │
│  TransitionManager.beginDelayedTransition(container, AutoTransition())│
│  view.visibility = View.VISIBLE                                 │
│  view.layoutParams.width = newWidth                             │
│  // 框架自动检测变化，生成动画！                                │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  本质：状态差异 → 自动生成属性动画                             │
│                                                                  │
│  Scene A (起始状态)          Scene B (结束状态)                  │
│  ┌──────────────┐            ┌──────────────┐                  │
│  │ ┌──┐         │            │ ┌──────────┐ │                  │
│  │ │  │ 小按钮  │  ────→    │ │          │ │                  │
│  │ └──┘         │  Transition│ │  大按钮  │ │                  │
│  │              │            │ │          │ │                  │
│  └──────────────┘            │ └──────────┘ │                  │
│                              └──────────────┘                  │
│                                                                  │
│  框架自动生成：                                                 │
│  • ChangeBounds 动画（位置+尺寸变化）                          │
│  • 无需手动计算差值                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 核心工作流程

```
┌─────────────────────────────────────────────────────────────────┐
│          TransitionManager.beginDelayedTransition() 流程        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① beginDelayedTransition(sceneRoot, transition)                │
│     │                                                            │
│     ↓                                                            │
│  ② captureStartValues()                                        │
│     │  遍历 sceneRoot 下的所有 View                             │
│     │  记录每个 View 的当前属性：                               │
│     │  • bounds (left, top, right, bottom)                      │
│     │  • visibility                                             │
│     │  • alpha, rotation, scale                                 │
│     │  存入 TransitionValues(view → Map<String, Object>)       │
│     ↓                                                            │
│  ③ 注册 OnPreDrawListener（关键！）                            │
│     │  在下一次绘制前拦截                                       │
│     ↓                                                            │
│  ④ 开发者代码执行布局变更                                      │
│     │  view.visibility = GONE                                   │
│     │  params.width = newWidth                                  │
│     │  requestLayout()                                          │
│     ↓                                                            │
│  ⑤ Layout 完成后，OnPreDrawListener 被触发                     │
│     │                                                            │
│     ↓                                                            │
│  ⑥ captureEndValues()                                          │
│     │  再次遍历所有 View，记录布局后的新属性值                  │
│     ↓                                                            │
│  ⑦ 对比起始/结束状态，createAnimator()                         │
│     │  对每个有差异的 View：                                    │
│     │  • Fade: visibility 变化 → alpha 动画                    │
│     │  • ChangeBounds: bounds 变化 → 位置/尺寸动画             │
│     │  • ChangeTransform: transform 变化 → 变换动画            │
│     │                                                            │
│     ↓                                                            │
│  ⑧ 先将 View 恢复到起始状态，然后播放属性动画到结束状态       │
│     │  （用户看到的是平滑过渡）                                 │
│     ↓                                                            │
│  ⑨ 动画结束，View 最终处于结束状态                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 常用 Transition 类型


| Transition          | 检测属性               | 生成动画                | 典型场景     |
| ------------------- | ------------------ | ------------------- | -------- |
| **Fade**            | visibility         | alpha 渐变            | 元素的显示/隐藏 |
| **ChangeBounds**    | bounds             | 位置+尺寸变化             | 布局重排     |
| **ChangeTransform** | scaleX/Y, rotation | 变换属性                | 缩放/旋转过渡  |
| **Slide**           | visibility         | 滑入/滑出               | 页面元素进出   |
| **AutoTransition**  | 组合                 | Fade + ChangeBounds | 默认通用转场   |
| **TransitionSet**   | 组合                 | 自定义组合               | 复杂转场     |


### 6.4 SharedElement 共享元素转场

共享元素转场是 Transition 框架最强大的能力之一——让两个页面中的"同一个"元素平滑过渡：

```
┌─────────────────────────────────────────────────────────────────┐
│              SharedElement 转场原理                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Activity A (列表页)              Activity B (详情页)           │
│  ┌──────────────┐                ┌──────────────┐              │
│  │ ┌────┐       │                │              │              │
│  │ │ 🖼 │ Title │ ──────────→  │  ┌────────┐  │              │
│  │ └────┘       │  共享元素     │  │   🖼   │  │              │
│  │ ┌────┐       │  转场动画     │  │        │  │              │
│  │ │    │       │                │  └────────┘  │              │
│  │ └────┘       │                │  Title       │              │
│  └──────────────┘                │  Content...  │              │
│                                  └──────────────┘              │
│                                                                  │
│  实现步骤：                                                     │
│  ─────────                                                      │
│  1. 两个 Activity 中给"同一元素"设置相同的 transitionName     │
│  2. 启动 Activity B 时传入共享元素映射                         │
│  3. 框架自动：                                                  │
│     a. 捕获 A 中元素的位置/尺寸/外观                           │
│     b. 捕获 B 中元素的位置/尺寸/外观                           │
│     c. 将 B 的元素先放到 A 的位置                              │
│     d. 播放动画过渡到 B 的目标位置                             │
│                                                                  │
│  内部机制：                                                     │
│  ─────────                                                      │
│  • 共享元素被提取到 DecorView 的 Overlay 层                    │
│  • 独立于两个 Activity 的 View 层级                            │
│  • 通过 ChangeBounds + ChangeTransform + ChangeImageTransform  │
│    组合实现平滑过渡                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.5 Transition 源码关键路径

```java
// Transition 基类的核心方法
abstract class Transition {

    // 捕获起始状态
    abstract void captureStartValues(TransitionValues transitionValues);
    // 捕获结束状态
    abstract void captureEndValues(TransitionValues transitionValues);

    // 根据前后差异创建 Animator
    abstract Animator createAnimator(ViewGroup sceneRoot,
        TransitionValues startValues, TransitionValues endValues);
}

// ChangeBounds 示例
class ChangeBounds extends Transition {

    void captureStartValues(TransitionValues values) {
        View view = values.view;
        // 记录 View 的边界
        values.values.put("bounds", new Rect(
            view.getLeft(), view.getTop(),
            view.getRight(), view.getBottom()));
    }

    Animator createAnimator(...) {
        Rect startBounds = startValues.values.get("bounds");
        Rect endBounds = endValues.values.get("bounds");

        if (!startBounds.equals(endBounds)) {
            // 位置/尺寸有变化，创建属性动画
            return ObjectAnimator.ofObject(view, "bounds",
                new RectEvaluator(), startBounds, endBounds);
        }
        return null;
    }
}
```

---

## 7. 物理动画（Physics-based Animation）

### 7.1 为什么需要物理动画

传统属性动画使用**固定时长 + 插值器**的模式，存在一个根本问题：**动画被中断或反向时，行为不自然**。

```
┌─────────────────────────────────────────────────────────────────┐
│            传统动画 vs 物理动画的中断行为                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  场景：用户拖拽卡片释放后，卡片正在回弹，但用户再次触摸       │
│                                                                  │
│  传统属性动画（固定时长）：                                     │
│  ────────────────────────                                       │
│  位置 │     ╱╲                                                  │
│       │    ╱  ╲      ╱                                         │
│       │   ╱    ╲    ╱   ← 中断后从头开始                       │
│       │  ╱      ╲  ╱      速度突变！不自然                     │
│       │ ╱        ╲╱                                             │
│  ─────│╱──────────────────                                      │
│       │                                                          │
│  物理动画（基于速度）：                                         │
│  ────────────────────                                           │
│  位置 │     ╱╲                                                  │
│       │    ╱  ╲                                                 │
│       │   ╱    ╲   ← 中断时保留当前速度                        │
│       │  ╱      ╲     自然过渡到新动画                         │
│       │ ╱        ╲────→                                        │
│  ─────│╱──────────────────                                      │
│       │                                                          │
│                                                                  │
│  核心差异：                                                     │
│  • 属性动画：时间驱动，duration 固定                           │
│  • 物理动画：力驱动，没有固定 duration，自然收敛               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 SpringAnimation（弹簧动画）

弹簧动画模拟真实弹簧的物理行为，由**弹簧力**和**阻尼力**共同决定运动轨迹。

#### 7.2.1 弹簧力学模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    弹簧力学模型                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  胡克定律 + 阻尼：                                              │
│                                                                  │
│  F = -k·x - c·v                                                │
│                                                                  │
│  其中：                                                         │
│  k = stiffness  (刚度，弹簧有多"硬")                          │
│  x = 当前位置 - 平衡位置 (偏移量)                              │
│  c = damping    (阻尼系数)                                      │
│  v = 当前速度                                                   │
│                                                                  │
│  -k·x  → 弹簧回复力（越偏离平衡点，力越大）                   │
│  -c·v  → 阻尼力（速度越快，阻力越大）                         │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  阻尼比 (dampingRatio) = c / (2 * √(k * m))                    │
│                                                                  │
│  不同阻尼比的效果：                                             │
│                                                                  │
│  dampingRatio < 1.0 (欠阻尼)：                                 │
│  ────────────────────────────                                   │
│  位置 │  ╱╲                    振荡后收敛                       │
│       │ ╱  ╲    ╱╲                                             │
│  ─────│╱────╲──╱──╲─╱──── ← 平衡位置                         │
│       │      ╲╱    ╲╱                                          │
│                                                                  │
│  dampingRatio = 1.0 (临界阻尼)：                               │
│  ────────────────────────────                                   │
│  位置 │  ╱                     最快到达平衡，无振荡             │
│       │ ╱                                                       │
│  ─────│╱────────────────── ← 平衡位置                         │
│       │                                                          │
│                                                                  │
│  dampingRatio > 1.0 (过阻尼)：                                 │
│  ────────────────────────────                                   │
│  位置 │  ╱                     缓慢到达平衡，无振荡             │
│       │╱                                                        │
│  ─────│───────────────────── ← 平衡位置                       │
│       │                                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 7.2.2 预定义参数


| 常量                              | 值     | 效果         |
| ------------------------------- | ----- | ---------- |
| **STIFFNESS_HIGH**              | 10000 | 硬弹簧，快速到达目标 |
| **STIFFNESS_MEDIUM**            | 1500  | 中等刚度（默认）   |
| **STIFFNESS_LOW**               | 200   | 软弹簧，缓慢到达   |
| **DAMPING_RATIO_HIGH_BOUNCY**   | 0.2   | 多次弹跳       |
| **DAMPING_RATIO_MEDIUM_BOUNCY** | 0.5   | 少量弹跳       |
| **DAMPING_RATIO_LOW_BOUNCY**    | 0.75  | 微弱弹跳       |
| **DAMPING_RATIO_NO_BOUNCY**     | 1.0   | 无弹跳（临界阻尼）  |


#### 7.2.3 帧更新逻辑

```java
// SpringForce 核心计算（每帧调用）
class SpringForce {
    float[] updateValues(float lastDisplacement, float lastVelocity, long deltaT) {
        float deltaT_sec = deltaT / 1000f;

        // 欠阻尼情况 (dampingRatio < 1.0)
        if (mDampingRatio < 1) {
            float cosCoeff = lastDisplacement;
            float sinCoeff = (lastVelocity + mDampedFreq * lastDisplacement)
                             / mDampedFreq;

            // 解析解：指数衰减 × 正弦振荡
            float displacement = Math.exp(-mDampedFreq * dampingRatio * deltaT_sec)
                * (cosCoeff * cos(mDampedFreq * deltaT_sec)
                 + sinCoeff * sin(mDampedFreq * deltaT_sec));

            float velocity = displacement * (-mNaturalFreq * mDampingRatio)
                + Math.exp(-mNaturalFreq * mDampingRatio * deltaT_sec)
                * (-mDampedFreq * cosCoeff * sin(mDampedFreq * deltaT_sec)
                 +  mDampedFreq * sinCoeff * cos(mDampedFreq * deltaT_sec));

            return new float[]{displacement, velocity};
        }
        // ... 临界阻尼和过阻尼的解析解
    }
}
```

### 7.3 FlingAnimation（惯性滑动动画）

FlingAnimation 模拟手指释放后的**惯性滑动**，速度因摩擦力而指数衰减：

```
┌─────────────────────────────────────────────────────────────────┐
│                 FlingAnimation 力学模型                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  运动方程：                                                     │
│  v(t) = v₀ × e^(-friction × t)                                 │
│  x(t) = v₀ / friction × (1 - e^(-friction × t))               │
│                                                                  │
│  速度随时间指数衰减：                                           │
│                                                                  │
│  速度 │╲                                                        │
│  v₀   │ ╲                                                      │
│       │  ╲                                                      │
│       │   ╲                                                     │
│       │    ╲╲                                                   │
│       │      ╲╲──────                                          │
│  0    │──────────────── → 时间                                  │
│                                                                  │
│  终止条件：|v(t)| < threshold (默认 1 px/s)                    │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  使用场景：                                                     │
│  • RecyclerView/ScrollView 惯性滚动                            │
│  • ViewPager 翻页惯性                                           │
│  • 自定义手势释放后的惯性效果                                   │
│                                                                  │
│  与 Scroller/OverScroller 的关系：                              │
│  Scroller 使用类似的摩擦模型，但 FlingAnimation 基于            │
│  AndroidX DynamicAnimation，支持实时修改参数和边界             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 物理动画的优势总结

```
┌─────────────────────────────────────────────────────────────────┐
│            物理动画 vs 传统属性动画                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  特性            │  属性动画           │  物理动画              │
│  ────────────────│─────────────────────│──────────────────────  │
│  驱动方式        │  时间驱动           │  力驱动                │
│  时长            │  固定 duration      │  自然收敛，无固定时长  │
│  可中断性        │  中断后需 cancel    │  随时可中断，速度连续  │
│                  │  再起新动画         │                        │
│  速度连续性      │  中断后速度可能突变 │  始终保持速度连续     │
│  参数            │  duration,          │  stiffness,           │
│                  │  interpolator       │  dampingRatio,        │
│                  │                     │  friction             │
│  适用场景        │  确定始终状态的动画 │  手势交互、弹性效果  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Compose 动画系统

### 8.1 设计哲学

Compose 动画系统的核心思想是：**动画是状态变化的可视化过程**。开发者只需声明"当状态改变时，用动画过渡"，框架自动处理帧更新。

```
┌─────────────────────────────────────────────────────────────────┐
│              Compose 动画的声明式理念                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  View 体系（命令式）：                                          │
│  ─────────────────────                                          │
│  // 创建动画                                                    │
│  val animator = ObjectAnimator.ofFloat(view, "alpha", 0f, 1f)  │
│  animator.duration = 300                                        │
│  animator.start()                                               │
│  // 还需要在合适时机 cancel                                    │
│  // 还需要处理生命周期                                          │
│                                                                  │
│  Compose（声明式）：                                            │
│  ─────────────────                                              │
│  // 只描述状态和过渡方式                                        │
│  val alpha by animateFloatAsState(                              │
│      targetValue = if (visible) 1f else 0f,                     │
│      animationSpec = tween(300)                                 │
│  )                                                              │
│  Box(Modifier.alpha(alpha)) { ... }                             │
│  // 状态变化自动触发动画                                        │
│  // 生命周期自动管理                                            │
│  // 中断时自动从当前值过渡到新目标                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 动画 API 层次

Compose 的动画 API 分为四层，从高层的便捷 API 到底层的帧控制：

```
┌─────────────────────────────────────────────────────────────────┐
│              Compose 动画 API 层次                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  高层 API（开箱即用，覆盖 80% 场景）                    │   │
│  │                                                          │   │
│  │  AnimatedVisibility     ← 元素显示/隐藏动画             │   │
│  │  AnimatedContent        ← 内容切换动画                  │   │
│  │  Crossfade              ← 交叉淡入淡出                  │   │
│  │  animateContentSize     ← 尺寸变化动画                  │   │
│  │                                                          │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │ 内部使用                       │
│                               ↓                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  中层 API（状态驱动动画值）                              │   │
│  │                                                          │   │
│  │  animateFloatAsState    ← Float 属性动画                │   │
│  │  animateDpAsState       ← Dp 属性动画                   │   │
│  │  animateColorAsState    ← 颜色属性动画                  │   │
│  │  animateIntAsState      ← Int 属性动画                  │   │
│  │  ...                                                     │   │
│  │                                                          │   │
│  │  updateTransition       ← 多值协调动画                  │   │
│  │    transition.animateFloat { state -> ... }              │   │
│  │    transition.animateColor { state -> ... }              │   │
│  │                                                          │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │ 内部使用                       │
│                               ↓                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  底层 API（可控制动画生命周期）                          │   │
│  │                                                          │   │
│  │  Animatable              ← 动画值持有者                 │   │
│  │    .animateTo(target, animationSpec)                     │   │
│  │    .snapTo(value)                                        │   │
│  │                                                          │   │
│  │  Animation               ← 动画定义                     │   │
│  │    TargetBasedAnimation  ← 目标值动画                   │   │
│  │    DecayAnimation        ← 衰减动画                     │   │
│  │                                                          │   │
│  └────────────────────────────┬────────────────────────────┘   │
│                               │ 内部使用                       │
│                               ↓                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  帧级 API（直接与 Choreographer 交互）                   │   │
│  │                                                          │   │
│  │  withFrameNanos { frameTimeNanos ->                     │   │
│  │      // 每帧回调，计算动画值                            │   │
│  │  }                                                       │   │
│  │                                                          │   │
│  │  最终调用 Choreographer.postFrameCallback()             │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 AnimationSpec：动画规格

Compose 使用 `AnimationSpec` 替代传统的 Interpolator + Duration 组合：

```
┌─────────────────────────────────────────────────────────────────┐
│              AnimationSpec 体系                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  AnimationSpec<T>                         │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                        │                                        │
│       ┌────────────────┼────────────────┐                      │
│       ↓                ↓                ↓                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐              │
│  │  tween   │   │  spring  │   │  keyframes   │              │
│  │          │   │          │   │              │              │
│  │ 时间驱动 │   │ 力驱动   │   │ 关键帧驱动  │              │
│  │          │   │          │   │              │              │
│  │ duration │   │ damping  │   │ 0ms: val=0  │              │
│  │ easing   │   │ Ratio    │   │ 150: val=1.2│              │
│  │          │   │ stiffness│   │ 300: val=1.0│              │
│  └──────────┘   └──────────┘   └──────────────┘              │
│                                                                  │
│       ┌────────────────┬────────────────┐                      │
│       ↓                ↓                ↓                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐              │
│  │repeatable│   │  snap    │   │infiniteRepeat│              │
│  │          │   │          │   │  able        │              │
│  │ 有限重复 │   │ 立即跳转 │   │ 无限重复    │              │
│  │ 包装其他 │   │ 无动画   │   │ 包装其他    │              │
│  │ spec     │   │          │   │ spec        │              │
│  └──────────┘   └──────────┘   └──────────────┘              │
│                                                                  │
│  使用示例：                                                     │
│  animate*AsState(                                               │
│      targetValue = ...,                                         │
│      animationSpec = spring(                                    │
│          dampingRatio = Spring.DampingRatioMediumBouncy,        │
│          stiffness = Spring.StiffnessLow                       │
│      )                                                          │
│  )                                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.4 Transition API：多值协调

当多个属性需要根据同一个状态同步动画时，使用 `updateTransition`：

```kotlin
// 定义状态枚举
enum class BoxState { Collapsed, Expanded }

@Composable
fun AnimatedBox(boxState: BoxState) {
    val transition = updateTransition(boxState, label = "box")

    // 多个值跟随同一状态协调变化
    val size by transition.animateDp(
        transitionSpec = { spring(stiffness = Spring.StiffnessLow) }
    ) { state ->
        when (state) {
            BoxState.Collapsed -> 48.dp
            BoxState.Expanded -> 200.dp
        }
    }

    val color by transition.animateColor(
        transitionSpec = { tween(durationMillis = 500) }
    ) { state ->
        when (state) {
            BoxState.Collapsed -> Color.Blue
            BoxState.Expanded -> Color.Red
        }
    }

    val cornerRadius by transition.animateDp(...) { state -> ... }

    Box(
        Modifier
            .size(size)
            .background(color, RoundedCornerShape(cornerRadius))
    )
}
```

### 8.5 Compose 动画与 View 动画系统的对比


| 维度       | View 属性动画                        | Compose 动画                     |
| -------- | -------------------------------- | ------------------------------ |
| **编程范式** | 命令式（start/cancel）                | 声明式（状态驱动）                      |
| **生命周期** | 手动管理                             | 自动跟随 Composable                |
| **中断处理** | 需手动 cancel + 新建                  | 自动从当前值过渡                       |
| **类型安全** | 弱（反射 setter）                     | 强（泛型 + 编译期检查）                  |
| **帧驱动**  | AnimationHandler → Choreographer | withFrameNanos → Choreographer |
| **底层引擎** | 相同：Choreographer + Vsync         | 相同：Choreographer + Vsync       |


---

## 9. 动画性能优化

### 9.1 动画卡顿的常见原因

```
┌─────────────────────────────────────────────────────────────────┐
│                动画卡顿的常见原因                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  一帧的时间预算（120Hz）：8.33ms                                │
│                                                                  │
│  Vsync ─→ [Input][Animation][Traversal] ─→ sync ─→ GPU        │
│           │←──── 主线程预算 ≈ 4ms ─────→│                      │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  原因 1：Layout/Measure 过重                                    │
│  ─────────────────────────────                                  │
│  • 动画改变尺寸属性 → 每帧触发 requestLayout()                │
│  • 深层 View 嵌套 → Measure/Layout 递归耗时                   │
│  • 例：animating width/height 比 translationX 慢得多          │
│                                                                  │
│  原因 2：过度绘制（Overdraw）                                   │
│  ─────────────────────────────                                  │
│  • 动画区域有大量重叠层级                                      │
│  • 半透明元素导致 GPU 合成负担                                 │
│  • Alpha 动画叠加多层绘制                                      │
│                                                                  │
│  原因 3：频繁 GC                                                │
│  ─────────────────────────────                                  │
│  • UpdateListener 中创建对象                                   │
│  • 自定义 TypeEvaluator 中 new 对象                            │
│  • 避免在动画回调中分配内存                                    │
│                                                                  │
│  原因 4：主线程阻塞                                             │
│  ─────────────────────────────                                  │
│  • 动画帧之间穿插耗时操作（IO/网络/大计算）                   │
│  • 其他 Message 耗时过长，挤压动画的执行时间                   │
│                                                                  │
│  原因 5：DrawCall 过多                                          │
│  ─────────────────────────────                                  │
│  • 动画元素的 DisplayList 操作过多                             │
│  • 每帧重新构建复杂 DisplayList                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 优化策略一：使用 ViewPropertyAnimator

`ViewPropertyAnimator` 是为 View 属性动画优化的高性能 API：

```java
// 传统方式：多个 ObjectAnimator
ObjectAnimator.ofFloat(view, "translationX", 200f).start();
ObjectAnimator.ofFloat(view, "alpha", 0.5f).start();
ObjectAnimator.ofFloat(view, "scaleX", 1.5f).start();

// 优化方式：ViewPropertyAnimator
view.animate()
    .translationX(200f)
    .alpha(0.5f)
    .scaleX(1.5f)
    .setDuration(300)
    .start();
```

```
┌─────────────────────────────────────────────────────────────────┐
│          ViewPropertyAnimator 的优势                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 批量更新：多个属性在同一帧中一次性更新                     │
│     （而非多个 ObjectAnimator 各自调用 setter）                 │
│                                                                  │
│  2. 避免反射：直接调用 View 的 native setter                   │
│     ObjectAnimator 需要通过反射查找 setter 方法                 │
│     ViewPropertyAnimator 直接调用 setTranslationX 等           │
│                                                                  │
│  3. 自动硬件层：在动画期间自动启用硬件层                       │
│     withLayer() 等价于动画开始时 setLayerType(HARDWARE)        │
│     动画结束时 setLayerType(NONE)                               │
│                                                                  │
│  4. 操作 RenderNode 属性：                                     │
│     translationX/Y, scaleX/Y, rotation, alpha 等属性           │
│     直接修改 RenderNode，不触发 invalidate()                   │
│     → RenderThread 可以直接应用变换，无需重建 DisplayList      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 优化策略二：硬件层缓存

```
┌─────────────────────────────────────────────────────────────────┐
│              硬件层缓存原理                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  无硬件层（每帧重新绘制）：                                     │
│  ──────────────────────────                                     │
│                                                                  │
│  帧 1: View 树遍历 → 构建 DisplayList → GPU 渲染              │
│  帧 2: View 树遍历 → 构建 DisplayList → GPU 渲染              │
│  帧 3: View 树遍历 → 构建 DisplayList → GPU 渲染              │
│  ...每帧都要重新构建和渲染                                      │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  有硬件层（GPU 纹理缓存）：                                    │
│  ──────────────────────────                                     │
│                                                                  │
│  帧 1: View 树遍历 → DisplayList → GPU 渲染 → 存为纹理       │
│  帧 2: 直接使用纹理 + 矩阵变换 ← 跳过 View 遍历！            │
│  帧 3: 直接使用纹理 + 矩阵变换                                │
│  ...直到硬件层失效                                              │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  适用场景：                                                     │
│  • View 内容不变，只做变换（平移/缩放/旋转/透明度）           │
│  • 复杂 View（大量子 View、复杂绘制）做动画                   │
│                                                                  │
│  不适用场景：                                                   │
│  • View 内容每帧变化（如 TextView 文字变化）                   │
│  • 硬件层纹理占用 GPU 显存，大 View 会消耗大量内存            │
│                                                                  │
│  使用方式：                                                     │
│  view.setLayerType(View.LAYER_TYPE_HARDWARE, null);            │
│  // 动画期间启用                                                │
│  animator.addListener(object : AnimatorListenerAdapter() {      │
│      override fun onAnimationEnd(animation: Animator) {         │
│          view.setLayerType(View.LAYER_TYPE_NONE, null);        │
│      }                                                          │
│  })                                                             │
│                                                                  │
│  或使用 ViewPropertyAnimator 自动管理：                         │
│  view.animate().alpha(0f).withLayer().start()                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.4 优化策略三：RenderNode 属性动画

Android 中有一类特殊属性可以**直接在 RenderThread 上更新，完全不需要主线程参与**：

```
┌─────────────────────────────────────────────────────────────────┐
│           RenderNode 属性 vs 普通属性                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RenderNode 属性（高性能）：                                    │
│  ─────────────────────────                                      │
│  translationX / translationY / translationZ                     │
│  scaleX / scaleY                                                │
│  rotation / rotationX / rotationY                               │
│  alpha                                                          │
│  elevation                                                      │
│                                                                  │
│  特点：                                                         │
│  • 修改后不触发 invalidate()                                   │
│  • 不需要重新 Measure/Layout                                   │
│  • 不需要重新构建 DisplayList                                  │
│  • RenderThread 直接修改变换矩阵                               │
│  • 主线程几乎零开销！                                          │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  普通属性（需要重绘）：                                         │
│  ─────────────────────                                          │
│  width / height (→ requestLayout)                               │
│  backgroundColor (→ invalidate)                                 │
│  text (→ requestLayout + invalidate)                            │
│  padding / margin (→ requestLayout)                             │
│                                                                  │
│  特点：                                                         │
│  • 修改后触发 requestLayout 或 invalidate                      │
│  • 需要重新 Measure/Layout/Draw                                │
│  • 主线程开销大                                                 │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  结论：动画应尽量使用 RenderNode 属性                          │
│  例如用 translationX 替代 marginLeft                            │
│  例如用 scaleX/scaleY 替代 width/height                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.5 优化策略四：Lottie/PAG 方案选型


| 方案              | 原理                       | 优势           | 劣势             | 适用场景        |
| --------------- | ------------------------ | ------------ | -------------- | ----------- |
| **Lottie**      | 解析 AE 导出的 JSON，运行时构建属性动画 | 生态好、设计师工具链成熟 | 复杂动画性能差（主线程解析） | 简单矢量动画、图标动画 |
| **PAG**         | 解析二进制格式，GPU 渲染           | 性能好、支持复杂特效   | 包体积较大          | 复杂特效、视频编辑   |
| **AGSL/Shader** | GPU Shader 直接绘制          | 性能极佳         | 开发门槛高          | 粒子效果、着色器动画  |
| **属性动画**        | Android 原生               | 零依赖、性能可控     | 复杂动画实现困难       | 简单 UI 交互动画  |


### 9.6 Perfetto 中的动画帧分析

```
┌─────────────────────────────────────────────────────────────────┐
│              Perfetto 动画性能分析                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  在 Perfetto 中关注的 Track：                                   │
│                                                                  │
│  1. Choreographer#doFrame                                       │
│     ├─ input (CALLBACK_INPUT)                                   │
│     ├─ animation (CALLBACK_ANIMATION) ← ★ 动画耗时            │
│     ├─ traversal (CALLBACK_TRAVERSAL)                           │
│     │   ├─ measure                                              │
│     │   ├─ layout                                               │
│     │   └─ draw                                                 │
│     └─ commit                                                   │
│                                                                  │
│  2. RenderThread                                                │
│     ├─ syncFrameState                                           │
│     ├─ DrawFrame                                                │
│     └─ queueBuffer                                              │
│                                                                  │
│  3. FrameTimeline                                               │
│     └─ 帧颜色：绿色=正常 / 红色=卡顿                          │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  分析步骤：                                                     │
│  ──────────                                                     │
│  1. 在 Perfetto 中抓取动画过程的 trace                         │
│  2. 找到 FrameTimeline 中的红色帧（卡顿帧）                   │
│  3. 展开该帧的 doFrame slice，查看各阶段耗时                   │
│  4. 如果 animation 阶段过长 → UpdateListener 回调耗时          │
│  5. 如果 traversal 阶段过长 → Layout/Draw 过重                 │
│  6. 如果 RenderThread 过长 → GPU 渲染过重或 DrawCall 过多      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. 总结

### 10.1 动画框架核心组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│              Android 动画框架核心组件关系                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                     用户交互 / 状态变化                     │ │
│  └────────────────────────────┬──────────────────────────────┘ │
│                               │                                 │
│          ┌────────────────────┼────────────────────┐           │
│          ↓                    ↓                    ↓           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │  View 动画   │  │  属性动画    │  │  Compose 动画    │    │
│  │              │  │              │  │                  │    │
│  │ Animation    │  │ ValueAnimator│  │ animate*AsState  │    │
│  │ 矩阵变换    │  │ ObjectAnim   │  │ Animatable       │    │
│  │              │  │ AnimatorSet  │  │ Transition API   │    │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘    │
│         │                 │                    │               │
│         │   Transition    │    Physics         │               │
│         │   Framework     │    Animation       │               │
│         │   ┌─────────┐  │   ┌──────────┐     │               │
│         │   │Scene差异│  │   │SpringForce│     │               │
│         │   │自动生成 │──│   │Fling      │     │               │
│         │   └─────────┘  │   └──────────┘     │               │
│         │                │                    │               │
│         ↓                ↓                    ↓               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              AnimationHandler (线程级调度器)              │  │
│  │                                                          │  │
│  │  管理所有活跃动画，统一注册帧回调                       │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │ postFrameCallback                    │
│                         ↓                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Choreographer                               │  │
│  │                                                          │  │
│  │  CALLBACK_INPUT → CALLBACK_ANIMATION → CALLBACK_TRAVERSAL│  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │ scheduleVsync                        │
│                         ↓                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Vsync 信号 (SurfaceFlinger)                 │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                         │                                      │
│                         ↓                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │         RenderThread → GPU → Display                     │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 各类动画的选型决策

```
┌─────────────────────────────────────────────────────────────────┐
│                 动画选型决策树                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  需要做动画                                                     │
│  │                                                               │
│  ├─ 使用 Compose？                                              │
│  │  ├─ 是 → Compose 动画 API                                   │
│  │  │  ├─ 简单显示/隐藏 → AnimatedVisibility                   │
│  │  │  ├─ 单属性过渡 → animate*AsState                         │
│  │  │  ├─ 多属性协调 → updateTransition                        │
│  │  │  └─ 需要手势联动 → Animatable + 协程                    │
│  │  │                                                            │
│  │  └─ 否 → View 体系动画                                      │
│  │     │                                                         │
│  │     ├─ 简单 View 变换（平移/缩放/旋转/透明度）？            │
│  │     │  └─ 是 → view.animate() (ViewPropertyAnimator)        │
│  │     │                                                         │
│  │     ├─ 需要与手势联动/可中断？                               │
│  │     │  └─ 是 → SpringAnimation / FlingAnimation              │
│  │     │                                                         │
│  │     ├─ 布局状态变化过渡？                                    │
│  │     │  └─ 是 → TransitionManager.beginDelayedTransition()   │
│  │     │                                                         │
│  │     ├─ Activity/Fragment 转场？                               │
│  │     │  └─ 是 → Transition + SharedElement                    │
│  │     │                                                         │
│  │     ├─ 自定义属性动画（颜色、路径等）？                      │
│  │     │  └─ 是 → ValueAnimator + TypeEvaluator                │
│  │     │                                                         │
│  │     ├─ 复杂设计师动画（AE 导出）？                           │
│  │     │  └─ 是 → Lottie / PAG                                 │
│  │     │                                                         │
│  │     └─ 逐帧动画（GIF 风格）？                                │
│  │        └─ 是 → AnimationDrawable 或 GIF 库                  │
│  │                                                               │
│  └─ 性能要求极高？                                              │
│     └─ 尽量使用 RenderNode 属性                                 │
│        避免 requestLayout，考虑硬件层缓存                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.3 关键知识点速查


| 概念                            | 说明                                                      |
| ----------------------------- | ------------------------------------------------------- |
| **Choreographer**             | 编舞者，协调 App 渲染，按 Vsync 节拍调度回调                            |
| **AnimationHandler**          | 动画中央调度器，管理所有活跃 Animator，ThreadLocal                     |
| **ValueAnimator**             | 属性动画核心引擎，计算时间进度→插值→求值                                   |
| **ObjectAnimator**            | 继承 ValueAnimator，自动通过 setter 应用属性值                      |
| **AnimatorSet**               | 动画编排容器，支持并行/顺序/依赖关系                                     |
| **TimeInterpolator**          | 时间插值器，将均匀时间映射为非均匀动画进度                                   |
| **TypeEvaluator**             | 值计算器，根据进度计算具体属性值                                        |
| **ViewPropertyAnimator**      | View 专用高性能动画 API，批量更新+自动硬件层                             |
| **Transition**                | 转场框架，自动检测状态差异并生成动画                                      |
| **SpringAnimation**           | 弹簧动画，力驱动，可中断，速度连续                                       |
| **FlingAnimation**            | 惯性滑动，速度指数衰减                                             |
| **RenderNode 属性**             | translationX/Y, scale, rotation, alpha 等，不触发 invalidate |
| **硬件层 (LAYER_TYPE_HARDWARE)** | 将 View 缓存为 GPU 纹理，跳过重复绘制                                |
| **AnimationSpec (Compose)**   | Compose 动画规格，包含 tween/spring/keyframes 等                |
| **Animatable (Compose)**      | Compose 底层动画值持有者，支持协程驱动                                 |


### 10.4 推荐学习资源

- [Android 官方文档 - Animation](https://developer.android.com/develop/ui/views/animations)
- [Android 官方文档 - Compose Animation](https://developer.android.com/develop/ui/compose/animation)
- [Android Perfetto 系列文章](https://www.androidperformance.com/)
- [Perfetto UI](https://ui.perfetto.dev/) - 在线性能分析工具

---

