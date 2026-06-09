# UiTick 事件对齐 — 架构优化分析

> **需求时间**：2024 年中 ~ 2024 年底
> **发布版本**：5.0 / 5.1
> **核心目标**：将事件处理、组件预创建等操作与显示器的 VSync（垂直同步）信号对齐，提升帧率稳定性和响应性，降低功耗

---

## 1. 概述

在移动设备上，屏幕以固定频率刷新（通常 60Hz/120Hz）。VSync 信号标识了每一帧的开始。如果框架的内部事件处理（如触摸事件响应、布局动画）不与 VSync 对齐，可能出现以下问题：

- **事件处理滞后**：事件在帧周期的任意时刻被处理，可能错过当前帧的渲染窗口
- **不必要的 VSync 请求**：没有事件需要处理时仍然请求 VSync，浪费功耗
- **帧抢占**：组件预创建等耗时操作抢占主线程，导致丢帧
- **关键事件延迟**：重要事件（如手势开始）需要等待下一个 VSync 才能处理，影响响应性

"UiTick 事件对齐" 需求通过一系列增量优化，将各种事件处理机制与 VSync 信号对齐。关键提交如下：

| # | 优化项 | 提交时间 | 提交 |
|---|--------|----------|------|
| 1 | 基于 VSync 的 AsyncEventBeat | 2024-07 | `23f2bcc3f` |
| 2 | 实现 SyncEventBeat，重要事件不等待 VSync | 2024-08 | `7ccc011d0` |
| 3 | 延迟任务调度优化 | 2024-08 | `2a78c8105` |
| 4 | AsyncEventBeat 使用 RuntimeScheduler 调度 | 2024-10 | `9bba016ce` |
| 5 | EventBeat 移除 taskExecutor 依赖 | 2024-10 | `da35ecbb9` |
| 6 | 不空闲时不再请求 VSync 帧 | 2024-10 | `dc0169d6c` |
| 7 | VSync 驱动的组件预创建（ComponentInstanceProvider） | 2024-11 | `385499f17` |

---

## 2. 基础设施：UITicker 与 NativeVsyncHandle

### 2.1 OH_NativeVSync 封装

`NativeVsyncHandle.h` 是对鸿蒙 `OH_NativeVSync` API 的 C++ 封装，提供：

- **`requestFrame(callback, data)`** — 请求下一帧 VSync 信号，触发回调
- **`getPeriodNs()`** — 获取屏幕刷新周期（如 60Hz = 16666667ns）

### 2.2 UITicker

`UITicker.h` 是整个对齐机制的枢纽。它维护一组 VSync 监听器，在有监听器注册时持续请求 VSync 帧：

```cpp
class UITicker {
  std::function<void()> subscribe(std::function<void(Timestamp)>&& listener) {
    std::lock_guard lock(listenersMutex);
    auto id = m_nextListenerId;
    m_nextListenerId++;
    auto listenersCount = m_listenerById.size();
    m_listenerById.insert_or_assign(id, std::move(listener));
    if (listenersCount == 0) {
      this->requestNextTick();  // 第一个注册者触发 VSync 请求
    }
    return [id, this]() {       // 返回 unsubscribe 函数
      std::lock_guard lock(listenersMutex);
      this->m_listenerById.erase(id);
    };
  }
};
```

在 `tick()` 回调中，所有监听器在同一 tick 内依次执行，然后将 `m_listenerById` 拷贝一份后释放锁再依次执行，以避免监听器在 unsubscribe 自身时导致死锁：

```cpp
void tick(Timestamp timestamp) {
  // 先执行普通任务
  { ... m_tasks ... }

  // 拷贝 listener 列表，避免死锁
  decltype(m_listenerById) listenerById;
  {
    std::lock_guard lock(listenersMutex);
    if (m_listenerById.empty()) return;
    listenerById = m_listenerById;  // 拷贝
  }
  for (const auto& idAndListener : listenerById) {
    idAndListener.second(timestamp);
  }
  this->requestNextTick();  // 请求下一帧
}
```

---

## 3. 基于 VSync 的 AsyncEventBeat

### 3.1 背景

在 React Native 中，`EventBeat` 是事件驱动循环的核心机制。当某个事件发生时（如触摸事件），`EventBeat::request()` 被调用，随后 `induce()` 在合适的时机执行回调，驱动 JS 处理事件并触发 UI 更新。

在引入 VSync 对齐之前，`EventBeat` 使用 `MessageQueueThread` 调度，事件处理与屏幕刷新周期无关。

### 3.2 改动内容（commit `23f2bcc3f`）

创建 `AsynchronousEventBeat`，将事件处理与 VSync 信号绑定：

```cpp
class AsynchronousEventBeat final : public facebook::react::EventBeat {
  void induce() const override {
    if (!isRequested_ || m_isBeatScheduled) return;
    isRequested_ = false;
    m_isBeatScheduled = true;
    auto weakOwner = ownerBox_->owner;
    m_runtimeExecutor([this, weakOwner](facebook::jsi::Runtime& runtime) {
      m_isBeatScheduled = false;
      auto owner = weakOwner.lock();
      if (!owner) return;
      if (beatCallback_) {
        beatCallback_(runtime);  // 在 JS 线程上执行事件回调
      }
    });
  }
};
```

`AsynchronousEventBeat` 在构造时通过 `uiTicker->subscribe(...)` 注册 VSync 监听器。每次 VSync 到来时触发 `induce()`，后者通过 `RuntimeExecutor` 将事件处理投递到 JS 线程。

### 3.3 优化效果

- 事件处理与 VSync 帧同步，减少帧延迟
- 降低因事件处理时机不准导致的"丢帧"感

---

## 4. SyncEventBeat — 关键事件优先处理

### 4.1 背景

某些事件（如手势识别、动画开始）对延迟极其敏感。如果所有事件都等待下一个 VSync 才处理，触摸到反馈的延迟会增加一个完整的帧周期（8~16ms）。

### 4.2 改动内容（commit `7ccc011d0`）

创建 `SynchronousEventBeat`，区分异步（Async）和同步（Sync）两种 EventBeat 工厂：

```cpp
// RNInstanceInternal.cpp
react::EventBeat::Factory asyncEventBeatFactory =
    [runtimeExecutor, uiTicker](auto ownerBox) {
      return std::make_unique<AsynchronousEventBeat>(
          ownerBox, runtimeExecutor, uiTicker);
    };

react::EventBeat::Factory syncEventBeatFactory =
    [runtimeScheduler, taskExecutor](auto ownerBox) {
      return std::make_unique<SynchronousEventBeat>(
          ownerBox, runtimeScheduler, taskExecutor);
    };
```

`SynchronousEventBeat::induce()` 对已在线程上的情况做了短路优化：

```cpp
void SynchronousEventBeat::induce() const {
  if (!isRequested_) return;

  if (m_taskExecutor->isOnTaskThread(TaskThread::JS)) {
    lockExecutorAndBeat();  // 已在线程上，立即执行
  } else {
    m_runtimeScheduler->scheduleWork([this](auto& runtime) {
      if (!isRequested_) return;
      beat(runtime);
    });
  }
}
```

### 4.3 优化效果

- **异步事件**（如普通触摸事件）：对齐 VSync，等待下一帧处理 → 帧同步
- **同步事件**（如手势识别、布局动画）：通过 `RuntimeScheduler::scheduleWork` 或 `executeNowOnTheSameThread` 立即处理，不等待 VSync → 低延迟

---

## 5. AsyncEventBeat 使用 RuntimeScheduler 调度

### 5.1 改动内容（commit `9bba016ce`）

`AsynchronousEventBeat` 改用 `RuntimeScheduler::ScheduleWork` 替代原始的 `RuntimeExecutor`。`ScheduleWork` 会将事件处理插入 JS 任务队列的更高优先级位置，而非简单添加到队列末尾。

```cpp
// 改动前
m_runtimeExecutor([this, weakOwner](auto& runtime) { ... });

// 改动后
m_runtimeScheduler->scheduleWork([this, weakOwner](auto& runtime) { ... });
```

### 5.2 优化效果

- 事件处理可以**抢占** JS 线程上优先级较低的任务（如批处理更新）
- 减少关键事件在 JS 线程上的排队延迟

---

## 6. 按需请求 VSync — 降低功耗

### 6.1 背景

在早期实现中，`UITicker` 只要有监听器注册，就会在每次 `tick()` 末尾无条件调用 `requestNextTick()`，形成一个无限 VSync 循环。这意味着即使没有任何事件需要处理，VSync 信号仍会被持续请求。

### 6.2 改动内容（commit `dc0169d6c` & `385499f17`）

**核心思路**：将 VSync 订阅从"构造函数中永久订阅"改为"在 `request()` 时按需订阅，事件处理完成后立即取消订阅"。

AsynchronousEventBeat 的变化——从构造时订阅改为 request() 时订阅：

```cpp
// 改动前（23f2bcc3f 初始版本）
AsynchronousEventBeat::AsynchronousEventBeat(...) {
  m_unsubscribeUITickerListener = uiTicker->subscribe(
      [this](auto timestamp) { this->induce(); });
  // 一旦构造，就永远订阅 VSync
}

// 改动后（dc0169d6c）
void AsynchronousEventBeat::request() const {
  EventBeat::request();
  std::lock_guard lock(m_unsubscribeUITickerListenerMtx);
  if (m_unsubscribeUITickerListener == nullptr) {
    m_unsubscribeUITickerListener = m_uiTicker->subscribe(
        [this](auto timestamp) { this->induce(); });
    // 仅在事件待处理时订阅 VSync
  }
}

void AsynchronousEventBeat::induce() const {
  // ...事件处理完成后立即取消订阅...
  {
    auto lock = std::lock_guard(m_unsubscribeUITickerListenerMtx);
    if (m_unsubscribeUITickerListener != nullptr) {
      m_unsubscribeUITickerListener();  // 取消订阅
      m_unsubscribeUITickerListener = nullptr;
    }
  }
}
```

同样，RNInstanceInternal 的动画 tick 也从"永久订阅"改为"动画开始时订阅、动画完成后取消"：

```cpp
void RNInstanceInternal::onAnimationStarted() {
  if (m_unsubscribeUITickListener != nullptr) return;
  m_unsubscribeUITickListener = m_uiTicker->subscribe([this](auto ts) {
    m_taskExecutor->runTask(TaskThread::MAIN, [this, ts]() {
      this->onUITick(ts);
    });
  });
}

void RNInstanceInternal::onAllAnimationsComplete() {
  if (m_unsubscribeUITickListener == nullptr) return;
  m_unsubscribeUITickListener();      // 取消 VSync 订阅
  m_unsubscribeUITickListener = nullptr;
}
```

### 6.3 优化效果

根据 commit 记录，**功耗降低约屏幕功耗的 4%**。因为当页面处于静态（无动画、无事件处理）时，框架不再发出 VSync 请求。

---

## 7. VSync 驱动的组件预创建

### 7.1 背景

在 React Native 的 Fabric 架构中，ShadowTree 更新时需要在 UI 线程创建对应的 ComponentInstance。如果所有 ComponentInstance 都在 mounting 阶段一次性创建，可能导致该帧耗时过长，引发丢帧。

### 7.2 架构设计

**ComponentInstancePreallocationRequestQueue** — 线程安全的请求队列。当 JS 线程预见到即将需要新的 ComponentInstance 时，将请求入队：

```cpp
void ComponentInstancePreallocationRequestQueue::push(Request request) {
  auto lock = std::lock_guard(m_mtx);
  m_queue.push(std::move(request));
  auto delegate = m_weakDelegate.lock();
  if (delegate != nullptr) {
    delegate->onPushPreallocationRequest();  // 通知 provider
  }
}
```

**ComponentInstanceProvider** — 核心类，负责协调预创建流程。当收到 `onPushPreallocationRequest` 通知时，订阅 VSync：

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
          self->onUITick(ts);  // 在 MAIN 线程执行预创建
        });
      });
}
```

**VSync 对齐的预创建执行**：`onUITick` 在 MAIN 线程每次 VSync 到来时执行。它通过 `shouldPausePreallocationToAvoidBlockingMainThread` 确保不会超出一帧的时间预算：

```cpp
void ComponentInstanceProvider::onUITick(Timestamp recentVSyncTimestamp) {
  // 如果没有待处理的请求，取消 VSync 订阅
  if (m_preallocationRequestQueue->isEmpty()) {
    // 取消订阅...
    return;
  }
  while (true) {
    // 检查当前帧是否还有剩余时间
    if (shouldPausePreallocationToAvoidBlockingMainThread(recentVSyncTimestamp)) {
      break;  // 时间用完了，下一帧再继续
    }
    auto maybeRequest = m_preallocationRequestQueue->pop();
    if (!maybeRequest.has_value()) break;
    processPreallocationRequest(maybeRequest.value());
  }
}

bool ComponentInstanceProvider::shouldPausePreallocationToAvoidBlockingMainThread(
    Timestamp recentVSyncTimestamp) {
  constexpr int FPS = 120;
  static constexpr auto FRAME_DURATION = std::chrono::nanoseconds(1000000000 / FPS);
  auto timeLeftInFrame = FRAME_DURATION -
      (std::chrono::steady_clock::now() - recentVSyncTimestamp);
  return timeLeftInFrame < (FRAME_DURATION / 2);  // 超过半帧时间则暂停
}
```

### 7.3 优化效果

- 组件预创建分散到多帧完成，避免单帧耗时过长
- 预创建在 VSync 对齐的时间片内执行，不干扰正常渲染
- 页面空闲时零开销（无 VSync 请求）
- 移除了旧的 `PreAllocationBuffer` 实现，简化为基于 VSync 的模型

---

## 8. EventBeat 消除不必要的 taskExecutor 依赖

### 8.1 改动内容（commit `da35ecbb9`）

EventBeat 不再持有 `TaskExecutor` 引用，改由 `RuntimeScheduler` 直接调度。这简化了 EventBeat 的依赖关系，也消除了 EventBeat 生命周期中潜在的线程安全问题：

```cpp
// 改动前
EventBeat::EventBeat(..., TaskExecutor::Shared taskExecutor, ...);

// 改动后
EventBeat::EventBeat(..., RuntimeScheduler::Shared runtimeScheduler, ...);
```

### 8.2 优化效果

- 减少 EventBeat 的依赖项，降低耦合
- 利用 RuntimeScheduler 的原生优先级调度能力，而非通用 TaskExecutor

---

## 9. 总结

"UiTick 事件对齐" 需求的核心思路是将 RNOH 框架中各类事件处理与 VSync 信号对齐，同时保持关键路径的低延迟。其完整的演进路径如下：

| 层级 | 组件 | 对齐方式 | 目的 |
|------|------|----------|------|
| **内核** | `UITicker` / `NativeVsyncHandle` | 封装 OH_NativeVSync | 提供统一的 VSync 信号源 |
| **事件驱动** | `AsynchronousEventBeat` | 订阅 VSync，事件到来时触发 | 事件处理与帧同步 |
| **事件驱动** | `SynchronousEventBeat` | 绕过 VSync，立即调度 | 关键事件低延迟响应 |
| **JS 调度** | `RuntimeScheduler::ScheduleWork` | 优先级调度 | 事件处理优先于批处理 |
| **功耗管理** | 按需订阅 VSync | 仅在有待处理事件时请求 | 降低静态功耗约 4% |
| **组件预创建** | `ComponentInstanceProvider` | VSync 驱动，预算控制 | 帧率稳定性保障 |

最终效果：
- **帧同步**：异步事件处理与显示器刷新对齐
- **低延迟**：同步事件不等待 VSync，立即处理
- **功耗优化**：无事件时零 VSync 请求
- **帧率稳定**：组件预创建控制在半帧时间预算内
