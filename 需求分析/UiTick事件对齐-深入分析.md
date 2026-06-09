# UiTick 事件对齐 — 深入分析文档

> **需求时间**：2023 Q4 启动，持续演进至 2026 年
> **发布版本**：5.0 / 5.1 及后续版本
> **核心目标**：将事件处理、动画驱动、组件预创建等操作与显示器 VSync 信号对齐，提升帧率稳定性、降低功耗、改善响应性

---

## 1. 问题本质

显示器以固定频率（60Hz/120Hz）刷新，VSync 信号标识每一帧的起始时间点。如果框架的内部事件处理不与 VSync 对齐，会导致三类问题：

1. **帧延迟**：事件在帧周期内任意时刻处理，可能错过当前帧的渲染窗口，用户感知一帧延迟
2. **功耗浪费**：无事件时仍持续请求 VSync，CPU/GPU 在空转
3. **帧抢占**：耗时操作（如组件创建）在帧的关键时刻执行，挤占渲染时间导致丢帧

"UiTick 事件对齐" 的本质是**让所有与帧相关的操作在 VSync 信号驱动下有序执行，同时让不需要帧同步的操作绕过 VSync 立即完成**。

---

## 2. 架构层级全景

| 层级 | 组件 | 作用 | 对齐方式 |
|------|------|------|----------|
| L0 硬件抽象 | `NativeVsyncHandle` | 封装 `OH_NativeVSync_*` API | 直接对齐 |
| L1 信号分发 | `UITicker` | VSync 信号分发器 | 订阅/取消，按需请求帧 |
| L2 事件驱动 | `EventBeat` | Fabric 事件节拍器 | VSync 到来时触发 induce |
| L2 事件驱动 | `VSyncListener` | 轻量级 VSync 监听 | 原子防止重复请求帧 |
| L3 动画驱动 | `RNInstanceInternal::onUITick` | Layout Animation 帧驱动 | 动画开始订阅，完成取消 |
| L3 动画驱动 | `NativeAnimatedTurboModule` | Native 动画帧驱动 | DisplaySoloist (API≥20) / VSyncListener (API<20) |
| L4 组件预创建 | `ComponentInstanceProvider` | VSync 驱动分帧预创建 | 按需订阅，预算控制 |
| L5 定时器 | `HarmonyTimerRegistry` | 短间隔定时器帧对齐 | deadline<1s 时用 VSync 替代 delayedTask |
| L6 性能监控 | `JSFpsMonitor` | JS FPS 与 VSync 关联 | 订阅 UITicker 计算帧率 |

---

## 3. 演进脉络：四个阶段

### Phase 1 — 原始 VSync 集成（2023 Q4 ~ 2024 Q2）

#### 3.1 动画驱动 VSync 化

| 提交 | 日期 | 说明 |
|------|------|------|
| `14a3e4f00` | 2023-10-03 | 首次引入 `NativeVsyncHandle` 替代 JS 定时器驱动动画 |

**架构解读**：React Native 上游的 Animated 模块使用 `requestAnimationFrame` 驱动动画帧，在浏览器中这天然与 VSync 对齐。但在鸿蒙上，最初的实现使用 JS 定时器，帧间隔不精确且与屏幕刷新不同步。

引入 `NativeVsyncHandle` 后，动画帧直接由硬件 VSync 信号驱动：

```
OH_NativeVSync_RequestFrame → onTick回调 → NativeAnimatedTurboModule::runUpdates → JS动画帧更新
```

#### 3.2 批量预创建（PreAllocationBuffer）

| 提交 | 日期 | 说明 |
|------|------|------|
| `f632aa695` | 2024-05-07 | 实现 `PreAllocationBuffer` 批量预创建 |
| `4a700bc04` | 2024-06-08 | 为 120Hz 调整预创建时间窗口 |

`PreAllocationBuffer` 是早期的预创建方案，不在 VSync 上对齐，只是简单地批量创建。后被 VSync 驱动的 `ComponentInstanceProvider` 替代。

---

### Phase 2 — VSync 驱动的 EventBeat 与动画 Tick（2024-07 ~ 2024-08）

#### 3.3 VSync-based AsyncEventBeat

| 提交 | 日期 | 说明 |
|------|------|------|
| `23f2bcc3f` | 2024-07-17 | 创建 `AsynchronousEventBeat`，将事件处理与 VSync 绑定 |
| `77310ef8c` | 2024-07-26 | 删除 `AsynchronousEventBeat`，概念合并入更简化的 `EventBeat` |

**架构解读**：Fabric 的 `EventBeat` 控制事件何时被"诱导"(induce)到 JS 线程处理。旧实现使用 `MessageQueueThread` 调度，事件处理时机与帧无关。

初始版本 `AsynchronousEventBeat` 在构造时永久订阅 UITicker，每帧都触发 induce。这有两个问题：
1. 无事件时仍消耗 VSync 帧资源
2. 事件处理不区分优先级

最终演化为当前的 `EventBeat`（按需订阅模式），详见 Phase 3。

#### 3.4 SynchronousEventBeat — 关键事件绕过 VSync

| 提交 | 日期 | 说明 |
|------|------|------|
| `7ccc011d0` | 2024-08-13 | 实现 `SynchronousEventBeat`，重要事件不等待 VSync |

**架构解读**：这是"UiTick 事件对齐"的关键洞察——**并非所有事件都应该等待 VSync**。

触摸事件、手势识别等用户交互需要最低延迟。如果等到下一个 VSync 才处理，延迟增加一个完整帧周期（8~16ms）。

`SynchronousEventBeat::induce()` 实现了短路逻辑：

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

`RNInstanceInternal.cpp` 中区分两种工厂：

```cpp
react::EventBeat::Factory asyncEventBeatFactory =    // ← 异步事件，VSync 对齐
    [...uiTicker](auto ownerBox) {
      return std::make_unique<EventBeat>(ownerBox, runtimeScheduler, uiTicker);
    };

react::EventBeat::Factory syncEventBeatFactory =     // ← 同步事件，绕过 VSync
    [...runtimeScheduler, ...taskExecutor](auto ownerBox) {
      return std::make_unique<SynchronousEventBeat>(ownerBox, runtimeScheduler, taskExecutor);
    };
```

#### 3.5 onUITick 崩溃修复

| 提交 | 日期 | 说明 |
|------|------|------|
| `0b9cc7c19` | 2024-08-20 | 修复 onUITick 中偶发崩溃 |
| `cac3d3d1d` | 2024-08-14 | onUITick nullptr 解引用修复 |

**根因**：VSync 回调在 RNInstance 尚未初始化完成（scheduler 为 nullptr）或已析构时触发。这说明 VSync 对齐优化需要配合对象生命周期保护。

---

### Phase 3 — 按需 VSync 与 EventBeat 简化（2024-10 ~ 2024-11）

#### 3.6 不空闲时不再请求 VSync 帧

| 提交 | 日期 | 说明 |
|------|------|------|
| `dc0169d6c` | 2024-10-30 | 按需请求 VSync 帧，功耗降低约 4% |

**核心改动**：将 VSync 订阅从"构造时永久订阅"改为"request() 时按需订阅，induce() 后立即取消"。

`EventBeat.cpp:34-49` 的 request() 实现：

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

`induce()` 中的安全检查（`EventBeat.cpp:51-57`）：

```cpp
void EventBeat::induce() const {
  if (auto runtimeScheduler = m_weakRuntimeScheduler.lock()) {
    facebook::react::EventBeat::induce();
  }
  // ← RuntimeScheduler 已销毁时安全跳过，防止悬空引用
}
```

同样，动画 tick 也从永久订阅改为动态订阅：

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

**功耗效果**：commit message 记录功耗降低约屏幕功耗的 4%。因为静态页面（无动画、无事件）时框架不再发出 VSync 请求。

#### 3.7 EventBeat 移除 TaskExecutor 依赖

| 提交 | 日期 | 说明 |
|------|------|------|
| `da35ecbb9` | 2024-10-23 | 从 EventBeat 移除 TaskExecutor |
| `9bba016ce` | 2024-10-17 | AsyncEventBeat 使用 RuntimeScheduler |

**架构解读**：EventBeat 原本持有 `TaskExecutor` 用于跨线程调度。改为使用 `RuntimeScheduler::scheduleWork()` 后：
1. 减少了 EventBeat 的依赖项
2. `scheduleWork` 比 `runTask` 优先级更高，事件处理可抢占低优先级 JS 任务
3. 与 SynchronousEventBeat 的调度方式统一

#### 3.8 UITicker listener ID 解绑

| 提交 | 日期 | 说明 |
|------|------|------|
| `bb289bb0d` | 2024-10-23 | UITicker listener ID 不再绑定 RNInstance ID |

listener ID 从 RNInstance 关联改为自增整数，避免多实例场景下 ID 冲突。

---

### Phase 4 — VSync 驱动组件预创建与 LTPO 支持（2024-11 ~ 2026）

#### 3.9 ComponentInstanceProvider — VSync 驱动分帧预创建

| 提交 | 日期 | 说明 |
|------|------|------|
| `385499f17` | 2024-11-04 | 创建 `ComponentInstanceProvider`，替代 `PreAllocationBuffer` |

**架构设计**：

```
JS线程: SchedulerDelegate → push request → ComponentInstancePreallocationRequestQueue
                                                    ↓ onPushPreallocationRequest()
ComponentInstanceProvider → subscribe UITicker
                                    ↓ VSync到来
MAIN线程: onUITick() → 循环处理请求 → shouldPausePreallocation? → 取消UITicker订阅
```

**关键代码**（`ComponentInstanceProvider.cpp:98-123`）：

```cpp
void ComponentInstanceProvider::onUITick(Timestamp recentVSyncTimestamp) {
  if (m_preallocationRequestQueue->isEmpty()) {
    // 没有请求了，取消 VSync 订阅
    std::lock_guard lock(m_unsubscribeUITickerListenerMtx);
    if (m_unsubscribeUITickerListener != nullptr) {
      m_unsubscribeUITickerListener();
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

**帧预算控制**（`ComponentInstanceProvider.cpp:143-153`）：

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

这是"UiTick 事件对齐"的精华：利用 VSync 时间戳精确计算帧内剩余时间，确保预创建只在安全的时间窗口内执行。

#### 3.10 DisplaySoloist — LTPO 动态帧率

| 提交 | 日期 | 说明 |
|------|------|------|
| `d4cf3562f` | 2025-06-27 | 支持 Native Animation LTPO 特性 |
| `db3d1196c` / `396a14944` | 2026-01-07/08 | 修复 AnimatedTM 销毁时 DisplaySoloist 回调崩溃 |

**架构解读**：鸿蒙 API 20+ 引入了 `OH_DisplaySoloist`，支持 LTPO（Low Temperature Polycrystalline Oxide）屏幕的动态帧率控制。

`NativeAnimatedTurboModule.cpp:317-349` 中根据动画速度投票帧率：

```cpp
void startDisplaySoloist() {
  m_nativeDisplaySoloist = OH_DisplaySoloist_Create(false);
  OH_DisplaySoloist_Start(m_nativeDisplaySoloist.get(), callback, this);
}

void setDisplaySoloistFrameRate() {
  // 根据动画速度选择 30/60/90/120fps
  OH_DisplaySoloist_SetExpectedFrameRateRange(m_nativeDisplaySoloist.get(), min, max, expected);
}
```

这是 VSync 对齐的进阶——不只是跟随固定频率的 VSync，而是**让屏幕刷新频率跟随动画需求动态调整**。

#### 3.11 VSync 线程异常保护

| 提交 | 日期 | 说明 |
|------|------|------|
| `4c8b9749b` | 2025-11-12 | 修复 VSync 线程因异常导致阻塞 |

在 `NativeAnimatedTurboModule::runUpdates()` 中添加 try-catch，防止动画异常阻塞 VSync 线程。捕获异常后重新请求 VSync 帧，确保动画循环不中断。

---

## 4. UITicker 核心实现分析

`UITicker.h` 是整个对齐体系的枢纽。它封装了 `NativeVsyncHandle` 并提供订阅/取消机制：

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
};
```

`tick()` 中的死锁防护设计值得注意：

```cpp
void tick(Timestamp timestamp) {
  // 先执行 tasks
  { auto lock = std::lock_guard(m_taskMtx); ... m_tasks.clear(); }

  // 拷贝 listener 列表后再执行，避免 listener 中 unsubscribe 导致死锁
  decltype(m_listenerById) listenerById;
  {
    std::lock_guard lock(listenersMutex);
    if (m_listenerById.empty()) return;
    listenerById = m_listenerById;  // ← 拷贝而非直接迭代
  }
  for (const auto& idAndListener : listenerById) {
    idAndListener.second(timestamp);
  }
  this->requestNextTick();
}
```

TODO 注释引用了 issue #1478，计划消除拷贝。当前拷贝是因为 `EventBeat::induce()` 在 listener 执行期间可能调用 unsubscribe，若此时持有 `listenersMutex` 将导致死锁。

---

## 5. 稳定性副作用与修复

VSync 对齐引入了一类特有的稳定性问题——**VSync 回调在对象析构后或跨线程竞争时到达**：

| 问题 | 修复提交 | 根因分析 |
|------|----------|----------|
| EventBeat::induce 析构/request 竞争 | `b67afac36` (2025-09) | MAIN线程析构与JS线程request()对 `m_unsubscribeUITickerListener` 的并发访问 |
| EventBeat 悬空 RuntimeScheduler 引用 | `2df264852` (2025-09) | shutdown 时 RuntimeScheduler 先销毁，VSync 回调仍触发 induce |
| onAnimationStarted UAF | `fe72ed2ed` (2025-04) | VSync 回调捕获 `this` 裸指针，实例释放后回调到达 |
| onUITick nullptr 崩溃 | `0b9cc7c19` (2024-08) | scheduler 未初始化或已释放时 VSync 回调触发 |
| AnimatedTM 销毁后 DisplaySoloist 回调 | `db3d1196c` (2026-01) | DisplaySoloist VSync 回调在 TM 销毁后仍执行 |
| VSync 线程异常阻塞 | `4c8b9749b` (2025-11) | 动画异常未捕获，VSync 线程被阻塞 |

稳定性编码规范（`docs/zh-cn/05-运维/稳定性编码规范/稳定性编码规范.md:173`）已明确规定：

> EventBeat、动画帧驱动、onUITick 等解绑。

这些修复的共同模式：**VSync/UITick 回调中使用 `weak_ptr` 替代 `this` 裸指针，在 mutex 保护下操作 unsubscribe handle，在 induce 中安全检查 RuntimeScheduler 生命周期**。

---

## 6. 与 React Native 上游的对齐

| RNOH 组件 | 上游对应 | 对齐情况 |
|-----------|----------|----------|
| `EventBeat` (VSync-aligned) | iOS `RCTEventBeat` / Android `EventBeat` | RNOH 增加了 VSync 驧动层，上游使用 MessageQueueThread |
| `SynchronousEventBeat` | Fabric `SynchronousEventBeat` | 完全对齐，概念来自上游 |
| `ComponentInstanceProvider` | iOS 无对应 / Android `ReactViewManager` 预创建 | RNOH 独有的 VSync 驧动分帧预创建 |
| `OH_DisplaySoloist` LTPO | Android `Choreographer` (API 31+ FrameRate API) | 概念对齐，API 不同 |
| `UITicker` 订阅/取消模式 | iOS `CADisplayLink` / Android `Choreographer.postVsyncCallback` | 概念对齐，RNOH 增加了按需订阅优化 |

---

## 7. 两个需求的交叉关系

"减少线程间通信" 和 "UiTick 事件对齐" 有显著的交叉点：

| 交叉点 | 减少线程间通信的视角 | UiTick事件对齐的视角 |
|--------|---------------------|---------------------|
| `isOnTaskThread()` 短路 | 避免跨线程 task 投递 | 在 MAIN 线程上零延迟执行 mounting 操作 |
| `performOnMainThread` | 减少一次线程跳转 | 让 mounting 在帧内立即完成而非下一帧 |
| VSync 按需订阅 | 减少空闲时的线程间通信（VSync 回调跨线程投递到 MAIN） | 降低功耗，只在需要时请求帧 |
| `pullTransaction` 在 MAIN | transaction 对象不再在 JS↔MAIN 间拷贝 | mounting 管线在同一帧内完成 |
| `callSync` 短路 | ImageLoader 同线程直接调用 | 图片更新在同一 VSync 帧内完成，不跨帧延迟 |

两个需求实际上是从不同角度解决同一个问题：**让操作在最合适的时间、在最合适的线程上完成**。