# 基于 VSync 对齐的 UiTick 事件驱动设计

更新时间: 2026-06-09

---

## 一、概述

### VSync 对齐机制简介

VSync（Vertical Synchronization，垂直同步）的全称是"**Vertical Synchronization Signal**"，中文译为"**垂直同步信号**"。

这是显示器以固定频率（60Hz/120Hz）刷新时发出的帧起始信号。通过将框架内部的事件处理与 VSync 信号对齐，实现帧同步、功耗降低和响应性提升。UiTick 事件对齐支持按需订阅 VSync 帧、区分异步/同步事件优先级、帧预算控制下的分帧预创建，使应用在高频交互场景下帧率稳定，而在静态场景中使用按需订阅模式降低功耗约 4%，最终提升用户体验。

通常，用"UiTick"指代 VSync 信号驱动的帧回调，用"EventBeat"指代 Fabric 事件节拍器。

---

## 二、特性介绍

UiTick 事件对齐是 RNOH 运行时性能优化的核心策略之一，按需将操作与显示器刷新周期对齐，优化帧率稳定性和功耗。

> **说明**
> 适配 UiTick 事件对齐的好处包括：事件处理与帧同步减少视觉延迟，关键事件绕过 VSync 实现低延迟响应，按需订阅 VSync 降低静态功耗约 4%，帧预算控制下的分帧预创建保障帧率稳定。

---

## 三、场景策略建议

UiTick 事件对齐的核心原则是"让所有与帧相关的操作在 VSync 信号驱动下有序执行，同时让不需要帧同步的操作绕过 VSync 立即完成"。

### ❌ 不建议所有事件都等待 VSync 处理

不建议将所有事件（包括触摸、手势等关键交互事件）都等待下一个 VSync 信号才处理。这会导致用户交互反馈延迟增加一个完整帧周期（8~16ms）。

主要原因有以下几点：

1. **关键事件延迟不可接受**：触摸事件、手势识别等用户交互需要最低延迟。等待 VSync 意味着用户触碰屏幕后，反馈最早也要等到下一帧才能出现，感知延迟明显。
2. **不必要的功耗浪费**：无事件时仍持续订阅 VSync 帧，CPU/GPU 在空转。早期 `AsynchronousEventBeat` 在构造时永久订阅 UITicker，静态页面也持续消耗帧资源。
3. **帧抢占风险**：组件预创建等耗时操作不做帧预算控制，可能在一帧的关键时刻挤占渲染时间，导致丢帧。

### ✅ 策略建议

| 类型 | 类型描述 | 参数建议（案例） |
|------|----------|------------------|
| **异步事件（帧同步型）** | 普通触摸事件、状态更新等，需要与帧渲染同步 | 使用 `EventBeat`（Async）：按需订阅 VSync，VSync 到来时 induce 处理，处理完立即取消订阅<br>使用 `RuntimeScheduler::scheduleWork` 高优先级调度到 JS 线程 |
| **同步事件（低延迟型）** | 手势识别、动画开始等关键交互，不能等待 VSync | 使用 `SynchronousEventBeat`：已在 JS 线程 → `lockExecutorAndBeat` 立即执行；不在 JS 线程 → `RuntimeScheduler::scheduleWork` 投递<br>绕过 VSync，零帧延迟 |
| **动画帧驱动型** | Layout Animation、Native Animation 等，每帧更新动画状态 | `RNInstanceInternal::onUITick`：动画开始时订阅 VSync，完成后取消订阅<br>`DisplaySoloist`（API≥20）：LTPO 动态帧率，根据动画速度投票 30/60/90/120fps |
| **分帧预创建型** | 组件实例预创建，需分散到多帧避免单帧超时 | `ComponentInstanceProvider`：有请求时订阅 VSync，在 `onUITick` 中按帧预算（半帧时间）逐个创建，空队列时取消订阅 |

---

## 四、性能验证工具

### JSFpsMonitor — JS FPS 与 VSync 关联

`JSFpsMonitor` 通过订阅 `UITicker` 计算 JS 线程实际帧率，并与 VSync 刷新率对比，识别帧对齐问题。

> 具体操作：启用 JSFpsMonitor 后，观察 JS FPS 是否稳定接近目标帧率（60/120fps），低于目标值时说明 JS 线程处理未与 VSync 对齐或存在帧抢占。

### callSync 超时与 TraceSection

参考"减少线程间通信最佳实践"第四章，通过 `callSync` 超时警告和 TraceSection 分析跨线程调用是否在 VSync 帧周期内完成。

### 功耗对比测试

**步骤1：准备静态页面**

部署一个无动画、无触摸交互的静态页面。

**步骤2：分别测试永久订阅和按需订阅模式**

在两种 VSync 订阅模式下运行相同页面，使用功耗监测工具采集数据。

**步骤3：对比功耗数据**

| 模式 | 静态页面功耗 | VSync 请求频率 |
|------|-------------|----------------|
| 永久订阅 UITicker | 基准值 | 每帧持续请求 |
| 按需订阅 UITicker | 降低约 4% | 仅在事件到来时请求 |

**步骤4：动态页面帧率测试**

在包含动画和频繁交互的页面中，对比 `SynchronousEventBeat` 和 `AsynchronousEventBeat` 的触摸响应延迟和帧率稳定性。

---

## 五、使用场景

### 场景说明

在 RNOH 多线程架构下，基于 `UITicker` 的 VSync 信号订阅/取消机制和 `EventBeat` 的异步/同步双通道体系，可以达到帧同步与低延迟的平衡目标。RNOH 支持按需 VSync 订阅、事件优先级区分和帧预算控制三种核心能力，开发者通过使用 `EventBeat`/`SynchronousEventBeat` 和 `ComponentInstanceProvider` 进行相关业务开发，可以享受 UiTick 事件对齐带来的帧率稳定和功耗优化收益。

UiTick 事件对齐支持开发者自定义事件优先级和帧预算策略，其常见使用场景：

- 通过 `EventBeat`（异步模式），用于普通事件的帧同步处理，具体可见本文第六章案例。
- 通过 `SynchronousEventBeat`，用于关键交互事件的低延迟响应，具体可见 `SynchronousEventBeat::induce` 实现。
- 通过 `ComponentInstanceProvider`，用于组件实例的 VSync 驱动分帧预创建，具体可见本文第七章案例。
- 通过 `DisplaySoloist`，用于 Native Animation 的 LTPO 动态帧率控制，具体可见 `NativeAnimatedTurboModule` 实现。

基于以上使用场景，结合 RNOH 实际演进历程，本文选取**两个**有代表性的案例展开介绍。

> **说明**
> VSync 回调中应使用 `weak_ptr` 替代 `this` 裸指针，在 mutex 保护下操作 unsubscribe handle，防止对象析构后回调仍执行导致崩溃。BACKGROUND 线程已废弃，不要开启 `enableBackgroundExecutor`。

---

## 六、基于按需订阅实现 EventBeat VSync 对齐

### 效果展示

EventBeat 从永久订阅 VSync 改为按需订阅后，静态页面功耗降低约 4%，且事件处理仍然与帧同步。

### 功耗对比

> **说明**
> 对比方法：在静态页面（无动画、无事件处理）下，分别测量永久订阅和按需订阅模式的屏幕功耗。

参考第四章"功耗对比测试"。

| 模式 | 静态页面功耗 | VSync 行为 |
|------|-------------|------------|
| 永久订阅（构造时 subscribe） | 基准值 | 每帧持续请求 VSync，CPU/GPU 空转 |
| 按需订阅（request 时 subscribe） | **降低约 4%** | 仅在事件到来时请求，处理完立即取消 |

### 实现说明

首先，理解 `UITicker` 的订阅/取消机制。

```cpp
// UITicker.h — subscribe 返回 unsubscribe 函数
std::function<void()> subscribe(std::function<void(Timestamp)>&& listener) {
  std::lock_guard lock(listenersMutex);
  auto id = m_nextListenerId++;
  auto listenersCount = m_listenerById.size();
  m_listenerById.insert_or_assign(id, std::move(listener));
  if (listenersCount == 0) {
    this->requestNextTick();  // ← 第一个注册者触发 VSync 请求
  }
  return [id, this]() {       // ← 返回 unsubscribe 函数
    std::lock_guard lock(listenersMutex);
    this->m_listenerById.erase(id);
  };
}
```

`UITicker.h`

**步骤一：EventBeat 按需订阅实现**

```cpp
void EventBeat::request() const {
  facebook::react::EventBeat::request();
  std::lock_guard lock(m_unsubscribeUITickerListenerMtx);
  if (m_unsubscribeUITickerListener != nullptr) {
    return;  // ← 已订阅，不重复订阅
  }
  m_unsubscribeUITickerListener = m_uiTicker->subscribe([this](auto timestamp) {
    this->induce();
    auto lock = std::lock_guard(m_unsubscribeUITickerListenerMtx);
    if (m_unsubscribeUITickerListener != nullptr) {
      m_unsubscribeUITickerListener();  // ← 一次性订阅，induce 后取消
      m_unsubscribeUITickerListener = nullptr;
    }
  });
}
```

`EventBeat.cpp:34-49`

**步骤二：induce 中安全检查 RuntimeScheduler 生命周期**

```cpp
void EventBeat::induce() const {
  if (auto runtimeScheduler = m_weakRuntimeScheduler.lock()) {
    facebook::react::EventBeat::induce();
  }
  // ← RuntimeScheduler 已销毁时安全跳过，防止悬空引用
}
```

`EventBeat.cpp:51-57`

**步骤三：动画 tick 同样采用按需订阅模式**

```cpp
void RNInstanceInternal::onAnimationStarted() {
  if (m_unsubscribeUITickListener != nullptr) return;  // ← 已订阅
  m_unsubscribeUITickListener = m_uiTicker->subscribe([this](auto ts) {
    m_taskExecutor->runTask(TaskThread::MAIN, [this, ts]() {
      this->onUITick(ts);
    });
  });
}

void RNInstanceInternal::onAllAnimationsComplete() {
  if (m_unsubscribeUITickListener == nullptr) return;
  m_unsubscribeUITickListener();            // ← 取消订阅
  m_unsubscribeUITickListener = nullptr;
}
```

`RNInstanceInternal.cpp`

**完整订阅/取消模式如下：**

```
事件到来 → request() → subscribe VSync → VSync 回调 → induce() → unsubscribe VSync
动画开始 → subscribe VSync → 每帧 onUITick → 动画完成 → unsubscribe VSync
预创建请求 → subscribe VSync → 每帧 onUITick 处理 → 队列空 → unsubscribe VSync
```

> **说明**
> 按需订阅的核心收益：只在需要时请求 VSync 帧，避免空闲时持续消耗帧资源。`induce()` 中使用 `weak_ptr` 检查 RuntimeScheduler 生命周期，防止 shutdown 时悬空引用。unsubscribe handle 在 mutex 保护下操作，防止析构与 request 的并发竞争。

---

## 七、基于帧预算控制实现 VSync 驱动分帧预创建

### 效果展示

`ComponentInstanceProvider` 在 VSync 驱动下按帧预算（半帧时间）逐个创建组件实例，避免单帧耗时过长导致丢帧。页面空闲时零 VSync 请求开销。

### 性能对比

> **说明**
> 对比方法：在包含大量组件首次渲染的场景下，对比一次性创建和分帧预创建的帧率稳定性。

| 指标 | 一次性创建（PreAllocationBuffer） | VSync 驱动分帧预创建 |
|------|----------------------------------|----------------------|
| 首帧丢帧风险 | 高（所有组件在一帧内创建） | 低（半帧时间预算控制） |
| 空闲时 VSync 请求 | 持续请求 | 零请求 |
| 帧率稳定性 | 不稳定 | 稳定 |

### 代码实现

**核心架构 — 请求队列驱动 VSync 订阅**

```cpp
void ComponentInstanceProvider::onPushPreallocationRequest() {
  std::lock_guard lock(this->m_unsubscribeUITickerListenerMtx);
  if (m_unsubscribeUITickerListener != nullptr) return;

  m_unsubscribeUITickerListener = m_uiTicker->subscribe(
      [weakSelf = this->weak_from_this(),
       weakTaskExecutor = this->m_weakTaskExecutor](auto ts) {
        auto taskExecutor = weakTaskExecutor.lock();
        if (taskExecutor == nullptr) return;
        taskExecutor->runTask(TaskThread::MAIN, [weakSelf, ts] {
          auto self = weakSelf.lock();
          if (self == nullptr) return;
          self->onUITick(ts);  // ← 在 MAIN 线程执行预创建
        });
      });
}
```

`ComponentInstanceProvider.cpp`

**帧预算控制 — onUITick 中按时间窗口逐个处理**

```cpp
void ComponentInstanceProvider::onUITick(Timestamp recentVSyncTimestamp) {
  if (m_preallocationRequestQueue->isEmpty()) {
    std::lock_guard lock(m_unsubscribeUITickerListenerMtx);
    if (m_unsubscribeUITickerListener != nullptr) {
      m_unsubscribeUITickerListener();      // ← 队列空，取消 VSync 订阅
      m_unsubscribeUITickerListener = nullptr;
    }
    return;
  }
  while (true) {
    if (shouldPausePreallocationToAvoidBlockingMainThread(recentVSyncTimestamp)) {
      break;  // ← 帧时间不够了，下一帧继续
    }
    auto maybeRequest = m_preallocationRequestQueue->pop();
    if (!maybeRequest.has_value()) break;
    processPreallocationRequest(maybeRequest.value());
  }
}
```

`ComponentInstanceProvider.cpp:98-123`

**帧预算计算 — 半帧时间阈值**

```cpp
bool ComponentInstanceProvider::shouldPausePreallocationToAvoidBlockingMainThread(
    Timestamp recentVSyncTimestamp) {
  constexpr int FPS = 120;
  static constexpr auto FRAME_DURATION = std::chrono::nanoseconds(1000000000 / FPS);
  auto timeLeftInFrame = FRAME_DURATION -
      (std::chrono::steady_clock::now() - recentVSyncTimestamp);
  return timeLeftInFrame < (FRAME_DURATION / 2);  // ← 剩余时间不足半帧则暂停
}
```

`ComponentInstanceProvider.cpp:143-153`

> **说明**
> 这是 UiTick 事件对齐的精华：利用 VSync 时间戳精确计算帧内剩余时间，确保预创建只在安全的时间窗口内执行。VSync 回调中使用 `weak_from_this()` 替代 `this` 裸指针，防止实例释放后回调仍执行。

---

## 八、总结

综上所述，使用 UiTick 事件对齐可以实现事件处理与帧同步减少视觉延迟，关键事件绕过 VSync 实现低延迟响应，按需订阅 VSync 降低静态功耗约 4%，帧预算控制下的分帧预创建保障帧率稳定。开发时推荐使用以下策略，享受帧率稳定和功耗优化效果：

| 策略 | 模式 | 典型实现 |
|------|------|----------|
| **异步事件 VSync 对齐** | 普通事件等待 VSync 触发 induce，处理完取消订阅 | `EventBeat`（按需订阅 UITicker） |
| **同步事件绕过 VSync** | 关键事件不等待 VSync，立即调度到 JS 线程 | `SynchronousEventBeat`（`scheduleWork` / `lockExecutorAndBeat`） |
| **按需订阅 VSync** | 仅在有待处理事件/动画时请求 VSync 帧，空闲时零请求 | `EventBeat::request()` / `onAnimationStarted` / `onPushPreallocationRequest` |
| **帧预算控制** | 利用 VSync 时间戳计算帧内剩余时间，超时则暂停操作 | `ComponentInstanceProvider::shouldPausePreallocationToAvoidBlockingMainThread` |
| **LTPO 动态帧率** | 根据动画速度投票屏幕刷新频率，屏幕跟随动画需求 | `DisplaySoloist`（API≥20，30/60/90/120fps） |
| **RuntimeScheduler 优先级调度** | 事件处理抢占 JS 线程低优先级任务 | `EventBeat` 使用 `scheduleWork` 替代 `RuntimeExecutor` |

---

## 九、示例代码

### UITicker — VSync 信号分发器

```cpp
class UITicker {
  std::function<void()> subscribe(std::function<void(Timestamp)>&& listener) {
    std::lock_guard lock(listenersMutex);
    auto id = m_nextListenerId++;
    auto listenersCount = m_listenerById.size();
    m_listenerById.insert_or_assign(id, std::move(listener));
    if (listenersCount == 0) {
      this->requestNextTick();  // ← 第一个注册者触发 VSync 请求
    }
    return [id, this]() {       // ← 返回 unsubscribe 函数
      std::lock_guard lock(listenersMutex);
      this->m_listenerById.erase(id);
    };
  }

  void tick(Timestamp timestamp) {
    decltype(m_listenerById) listenerById;
    {
      std::lock_guard lock(listenersMutex);
      if (m_listenerById.empty()) return;
      listenerById = m_listenerById;  // ← 拷贝避免死锁
    }
    for (const auto& idAndListener : listenerById) {
      idAndListener.second(timestamp);
    }
    this->requestNextTick();
  }
};
```

`UITicker.h`

### SynchronousEventBeat — 关键事件低延迟

```cpp
void SynchronousEventBeat::induce() const {
  if (!isRequested_) return;
  if (m_taskExecutor->isOnTaskThread(TaskThread::JS)) {
    lockExecutorAndBeat();  // ← 已在 JS 线程，立即执行
  } else {
    m_runtimeScheduler->scheduleWork([this](auto& runtime) {
      if (!isRequested_) return;
      beat(runtime);         // ← 通过 RuntimeScheduler 高优先级调度
    });
  }
}
```

`SynchronousEventBeat.cpp`

### EventBeat 工厂 — 异步/同步双通道

```cpp
// RNInstanceInternal.cpp
react::EventBeat::Factory asyncEventBeatFactory =    // ← 异步事件，VSync 对齐
    [...uiTicker](auto ownerBox) {
      return std::make_unique<EventBeat>(ownerBox, runtimeScheduler, uiTicker);
    };

react::EventBeat::Factory syncEventBeatFactory =     // ← 同步事件，绕过 VSync
    [...runtimeScheduler, ...taskExecutor](auto ownerBox) {
      return std::make_unique<SynchronousEventBeat>(ownerBox, runtimeScheduler, taskExecutor);
    };
```

`RNInstanceInternal.cpp`

### DisplaySoloist — LTPO 动态帧率

```cpp
void startDisplaySoloist() {
  m_nativeDisplaySoloist = OH_DisplaySoloist_Create(false);
  OH_DisplaySoloist_Start(m_nativeDisplaySoloist.get(), callback, this);
}

void setDisplaySoloistFrameRate() {
  OH_DisplaySoloist_SetExpectedFrameRateRange(m_nativeDisplaySoloist.get(), min, max, expected);
}
```

`NativeAnimatedTurboModule.cpp:317-349`

### VSync 回调稳定性编码规范

```cpp
// ✅ 推荐：使用 weak_ptr 替代 this 裸指针
m_unsubscribeUITickerListener = m_uiTicker->subscribe(
    [weakSelf = this->weak_from_this(), weakTaskExecutor = this->m_weakTaskExecutor](auto ts) {
      auto self = weakSelf.lock();
      if (self == nullptr) return;  // ← 实例已释放，安全跳过
      self->onUITick(ts);
    });

// ✅ 推荐：在 mutex 保护下操作 unsubscribe handle
void EventBeat::induce() const {
  if (auto runtimeScheduler = m_weakRuntimeScheduler.lock()) {
    facebook::react::EventBeat::induce();
  }
  // ← RuntimeScheduler 已销毁时安全跳过
}
```

`ComponentInstanceProvider.cpp` / `EventBeat.cpp`