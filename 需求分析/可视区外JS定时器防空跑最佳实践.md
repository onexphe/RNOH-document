# 基于 VSync 帧对齐的可视区外 JS 定时器防空跑设计

更新时间: 2026-06-09

---

## 一、概述

### JS 定时器防空跑机制简介

JS 定时器防空跑的全称是"**可视区外 JS 轮播图防空跑**"，核心目标是当 JS 侧的轮播图（Carousel/Swiper）不在可视区时，避免其定时器（`setInterval` / `requestAnimationFrame`）空跑消耗 CPU 和电池。

这是 RNOH 定时器架构从 ArkTS 侧完全迁至 C++ 侧后的关键优化。通过 `HarmonyTimerRegistry`（替代旧版 `TimingTurboModule`）统一管理所有定时器，实现定时器与 VSync 帧对齐、到期定时器批量触发、无定时器时不请求帧。`HarmonyTimerRegistry` 支持短间隔定时器（< 1s）使用 VSync 帧回调驱动，长间隔定时器（≥ 1s）使用 `runDelayedTask` 精确等待，使应用在多定时器场景下每帧只唤醒 JS 线程一次，而在静态页面中零 VSync 请求开销。

通常，用"空跑"指代定时器在可视区外仍持续触发回调的无意义执行，用"帧对齐"指代定时器触发时机与 VSync 信号同步的调度模式。

---

## 二、特性介绍

JS 定时器防空跑是 RNOH 运行时性能优化的核心策略之一，按需将定时器触发与显示器刷新周期对齐，优化 CPU 使用率和电池续航。

> **说明**
> 适配 JS 定时器防空跑的好处包括：每帧只唤醒 JS 线程一次（N 个定时器 = 1 次唤醒），到期定时器批量触发合并为一次 JSI 调用，`requestAnimationFrame` 从每毫秒触发改为每帧一次，静态页面零 VSync 请求开销。

---

## 三、场景策略建议

JS 定时器防空跑的核心原则是"让所有定时器跟着屏幕刷新走，每帧只检查一次，到期的一批一起触发，没有定时器时不请求帧"。

### ❌ 不建议每个定时器独立调度

不建议为每个定时器独立创建 `runDelayedTask`，也不建议将 `requestAnimationFrame` 当作 `setTimeout(..., 1)` 处理。这会导致 N 个定时器 = N 次 JS 线程唤醒，`requestAnimationFrame` 每毫秒触发一次回调。

主要原因有以下几点：

1. **JS 线程反复唤醒**：每个定时器独立调度意味着每次到期都唤醒 JS 线程。页面有 5 个 `setInterval` 和 2 个 `requestAnimationFrame` 时，每帧最多被唤醒 7 次，CPU 频繁从低功耗状态恢复。
2. **`requestAnimationFrame` 灾难性触发频率**：旧实现中 `requestAnimationFrame` 被当作 `setTimeout(..., 1)` 处理，意味着每毫秒都可能触发一次回调，在 120Hz 屏幕上动画帧每秒跑 120 次，而非预期的 120 次。
3. **可视区外空跑浪费电池**：轮播图不在可视区时，`setInterval` 仍每 3 秒执行一次回调、触发 React 重新渲染，渲染结果用户根本看不到，全是白忙。多个轮播图同时空跑时，每帧被 JS 线程唤醒多次。

### ✅ 策略建议

| 类型 | 类型描述 | 参数建议（案例） |
|------|----------|------------------|
| **短间隔定时器（< 1s）** | `setInterval(fn, 100)` / `requestAnimationFrame` 等高频定时器 | 使用 VSync 帧回调驱动：`scheduleWakeUp` 中 `delay < 1s` 时走 `VSyncListener::requestFrame`<br>每帧只检查一次到期状态，到期定时器批量触发 |
| **长间隔定时器（≥ 1s）** | `setTimeout(fn, 3000)` / `setInterval(fn, 5000)` 等低频定时器 | 使用 `runDelayedTask` 精确等待：`scheduleWakeUp` 中 `delay ≥ 1s` 时走 `EventLoopTaskRunner::runDelayedTask`<br>避免每帧都检查远期定时器是否到期 |
| **`requestAnimationFrame`** | 动画帧回调，需要与屏幕刷新同步 | 使用 VSync 帧驱动：与短间隔定时器走相同调度路径，每帧最多触发一次<br>不再当作 `setTimeout(..., 1)`，消除每毫秒触发的问题 |
| **`requestIdleCallback`** | 低优先级任务，可在帧空闲时间执行 | 使用 `IdleCallbacksCxxTurboModule`：帧开始处理高优先级定时器，帧结束利用剩余时间执行 idle callback<br>与定时器帧对齐形成"帧前紧急、帧后闲"的完整调度策略 |
| **后台/可视区外** | 页面不可见或组件不在可视区 | `cancelWakeUp()` 取消所有唤醒，CPU 进入低功耗状态<br>回到前台时 `resumeTimers()` 恢复调度 |

---

## 四、性能验证工具

### 定时器触发频率分析

通过 TraceSection 标记 `triggerExpiredTimers` 和 `callTimer` 的执行时间点，分析定时器触发频率和 JS 线程唤醒次数。

> 具体操作：在 DevTools Performance 面板中筛选 `triggerExpiredTimers` 相关 TraceSection，统计每帧的定时器触发次数。旧实现每帧多次触发，新实现每帧一次批量触发。

### CPU 使用率对比

**步骤1：部署包含多定时器的页面**

部署一个包含 3 个 `setInterval`（间隔 100ms-3000ms）和 1 个 `requestAnimationFrame` 的页面。

**步骤2：测量旧实现和新实现的 CPU 使用率**

| 实现 | 每帧 JS 线程唤醒次数 | `requestAnimationFrame` 触发频率 | 综合开销 |
|------|----------------------|----------------------------------|----------|
| 旧实现（独立调度） | 最多 4 次 | 每毫秒 1 次 | 基准值 |
| 新实现（帧对齐+批量） | 1 次 | 每帧 1 次 | 降低 5-120 倍 |

**步骤3：可视区外空跑对比**

将轮播图滚出可视区，测量两种实现下的 CPU 使用率。

**步骤4：静态页面功耗对比**

部署无定时器的静态页面，测量 VSync 请求频率。

| 实现 | 静态页面行为 | VSync 请求 |
|------|-------------|-----------|
| 旧实现 | 可能仍有残留定时器触发 | 持续请求 |
| 新实现 | 无定时器 → 零唤醒 | 零请求 |

---

## 五、使用场景

### 场景说明

在 RNOH 多线程架构下，基于 `HarmonyTimerRegistry` 的帧对齐调度和 `VSyncListener` 的原子防重复帧请求机制，可以达到定时器高效触发与 CPU/电池节省的平衡目标。RNOH 支持定时器帧对齐、批量触发、按需调度和生命周期管控四种核心能力，开发者通过使用 `HarmonyTimerRegistry` 管理定时器和使用 `requestIdleCallback` 处理低优先级任务，可以享受防空跑带来的性能收益。

JS 定时器防空跑支持开发者自定义定时器调度策略，其常见使用场景：

- 通过 `HarmonyTimerRegistry` 帧对齐调度，用于 `setInterval` 轮播图等高频定时器的批量触发，具体可见本文第六章案例。
- 通过 VSync 帧驱动 `requestAnimationFrame`，用于动画帧回调的每帧一次触发，具体可见本文第七章案例。
- 通过 `runDelayedTask` 精确等待，用于 `setTimeout(fn, 3000)` 等长间隔定时器的调度，具体可见 `HarmonyTimerRegistry::scheduleWakeUp` 实现。
- 通过 `IdleCallbacksCxxTurboModule`，用于 `requestIdleCallback` 低优先级任务的帧空闲时间执行。

基于以上使用场景，结合 RNOH 实际演进历程，本文选取**两个**有代表性的案例展开介绍。

> **说明**
> `setTimeout(fn, 0)` 不再立即触发，改为至少等到下一帧，防止动画闪烁。后台时 `cancelWakeUp()` 取消所有唤醒，回到前台时 `resumeTimers()` 恢复。

---

## 六、基于 HarmonyTimerRegistry 实现定时器帧对齐与批量触发

### 效果展示

`HarmonyTimerRegistry` 替代旧版 `TimingTurboModule` 后，所有定时器共享一个 VSync 驱动的唤醒机制，每帧只唤醒 JS 线程一次，到期定时器合并为一次 `callTimers` 调用。

### 性能对比

> **说明**
> 对比方法：在包含 5 个 `setInterval`（间隔 100ms-3000ms）和 2 个 `requestAnimationFrame` 的页面中，统计每帧的 JS 线程唤醒次数和 JSI 调用次数。

参考第四章"定时器触发频率分析"。

| 指标 | 旧实现（独立调度） | 新实现（帧对齐+批量） | 提升倍数 |
|------|-------------------|----------------------|---------|
| 每帧定时器到期触发 | 最多 7 次独立触发 | 1 次批量触发 | ~7x |
| 每帧 JS 线程唤醒 | 最多 7 次 | 1 次 | ~7x |
| 综合定时器相关开销 | 基准值 | 降低 5-120 倍 | 5-120x |

### 实现说明

首先，理解 `HarmonyTimerRegistry` 的定时器数据结构。

```cpp
struct Timer {
    uint32_t id;
    double deadlineMs;  // ← 绝对到期时间戳(ms)
    double durationMs;  // ← 定时器间隔(ms)
    bool repeats;       // ← 是否循环
};
```

`HarmonyTimerRegistry.h`

旧版存储 `jsSchedulingTime`（JS 侧创建时间）用于计算相对 delay，新版改为 `deadlineMs`（绝对到期时间），`triggerExpiredTimers` 中直接用 `deadlineMs <= now` 判断是否到期。

**步骤一：定时器创建 — 只存储不调度**

```cpp
void HarmonyTimerRegistry::createTimer(uint32_t timerId, double delayMs) {
    assertJSThread();
    auto now = getMillisSinceEpoch();
    auto deadline = now + delayMs;
    m_activeTimerById.emplace(timerId, Timer{timerId, deadline, delayMs, false});

    if (isForeground && !m_vsyncListener->isScheduled()) {
        scheduleWakeUp();  // ← 只调度一次唤醒，不为每个定时器独立调度
    }
}
```

`HarmonyTimerRegistry.cpp`

**步骤二：到期定时器批量触发**

```cpp
void HarmonyTimerRegistry::triggerExpiredTimers() {
    std::vector<Timer> expiredTimers;
    auto now = getMillisSinceEpoch();

    for (auto const& [id, timer] : m_activeTimerById) {
        if (timer.deadlineMs <= now) {
            expiredTimers.push_back(timer);
        }
    }

    if (!expiredTimers.empty()) {
        std::sort(expiredTimers.begin(), expiredTimers.end(),
            [](auto a, auto b) { return a.deadlineMs < b.deadlineMs; });

        std::vector<uint32_t> expiredTimerIds;
        std::transform(expiredTimers.begin(), expiredTimers.end(),
            std::back_inserter(expiredTimerIds),
            [](auto timer) { return timer.id; });

        triggerTimers(expiredTimerIds);  // ← 批量触发!
    }

    if (!m_activeTimerById.empty()) {
        scheduleWakeUp();  // ← 还有未到期定时器，继续调度唤醒
    }
}
```

`HarmonyTimerRegistry.cpp`

**步骤三：唤醒调度 — 短间隔走 VSync，长间隔走 delayedTask**

```cpp
void HarmonyTimerRegistry::scheduleWakeUp() {
    constexpr double MINIMUM_SLEEP_DELAY = 1000.;  // ← 1秒阈值

    auto nextDeadline = getNextDeadline();
    auto now = getMillis_sinceEpoch();
    auto delay = nextDeadline - now;

    if (delay >= MINIMUM_SLEEP_DELAY) {
        // >= 1s: 用 delayedTask 精确等待
        if (nextDeadline >= m_nextTimerDeadline) return;
        m_nextTimerDeadline = nextDeadline;
        cancelWakeUp();
        m_wakeUpTask = m_taskExecutor->runDelayedTask(
            TaskThread::JS,
            [weakSelf = getWeakSelf()] {
                if (auto self = weakSelf.lock()) {
                    self->m_nextTimerDeadline = std::numeric_limits<double>::max();
                    self->triggerExpiredTimers();
                }
            },
            std::max(delay, 0.));
    } else {
        // < 1s: 用 VSync 帧回调检查
        m_vsyncListener->requestFrame(
            [taskExecutor = m_taskExecutor, weakSelf = getWeakSelf()](auto) {
                taskExecutor->runTask(TaskThread::JS, [weakSelf] {
                    if (auto self = weakSelf.lock()) {
                        self->triggerExpiredTimers();
                    }
                });
            });
    }
}
```

`HarmonyTimerRegistry.cpp`

> **说明**
> 关键区别：旧实现每个定时器独立 `runDelayedTask`，N 个定时器 = N 次 JS 线程唤醒；新实现所有定时器共享一个唤醒机制，N 个定时器 = 1 次唤醒 + 1 次 JSI 调用。VSync 回调中使用 `weak_ptr` 替代 `this` 裸指针，防止析构后回调仍执行。

---

## 七、基于 VSync 帧驱动实现 requestAnimationFrame 正确调度

### 效果展示

`requestAnimationFrame` 从旧实现的每毫秒触发改为每帧一次 VSync 驱动触发，动画帧频率从不可控变为与屏幕刷新率同步。

### 性能对比

> **说明**
> 对比方法：在 120Hz 屏幕上运行使用 `requestAnimationFrame` 驱动的动画，统计回调触发频率。

| 实现 | `requestAnimationFrame` 触发频率 | 动画帧率 | CPU 占用 |
|------|----------------------------------|----------|----------|
| 旧实现（`setTimeout(..., 1)`） | 每毫秒 1 次 | 不稳定，远超 120fps | 高 |
| 新实现（VSync 帧驱动） | 每帧 1 次 | 稳定 120fps | 低 |
| **触发频率降低** | — | — | **~120 倍** |

### 代码实现

**旧实现的问题**

```cpp
// 旧版：requestAnimationFrame 被当作 setTimeout(fn, 1ms) 处理
// 结果：每毫秒都可能触发一次回调
auto timerTask = m_ctx.taskExecutor->runDelayedTask(
    TaskThread::JS,
    [weakSelf = weak_from_this(), id] {
        auto self = weakSelf.lock();
        if (!self) return;
        self->triggerTimer(id);  // ← 单个定时器触发
    },
    1,  // ← 1ms 间隔！
    0);
```

**新实现 — `requestAnimationFrame` 走 VSync 帧对齐路径**

```cpp
// requestAnimationFrame 在 HarmonyTimerRegistry 中作为 duration=0 的定时器
// scheduleWakeUp 中 delay=0 < MINIMUM_SLEEP_DELAY(1s) → 走 VSync 路径
void HarmonyTimerRegistry::scheduleWakeUp() {
    // ... delay=0 走 VSync 分支：
    m_vsyncListener->requestFrame(
        [taskExecutor, weakSelf](auto) {
            taskExecutor->runTask(TaskThread::JS, [weakSelf] {
                if (auto self = weakSelf.lock()) {
                    self->triggerExpiredTimers();  // ← 每帧最多触发一次
                }
            });
        });
}
```

**VSyncListener — 原子防重复帧请求**

```cpp
void VSyncListener::scheduleNextVsync() {
    auto alreadyScheduled = m_scheduled.exchange(true);
    if (alreadyScheduled) return;  // ← 已请求帧，不重复请求
    m_vsyncHandle.requestFrame(callback, data);
}
```

`VSyncListener.h`

> **说明**
> `setTimeout(fn, 0)` 旧版被立即触发（绕过 VSync），导致 `requestAnimationFrame` 每毫秒执行。修复后 `duration=0` 的定时器也走 VSync 帧对齐流程，至少等到下一帧才触发。`VSyncListener` 使用原子 `m_scheduled` 标志确保即使多个短间隔定时器在同一帧内到期，也只请求一次 VSync 帧。

---

## 八、总结

综上所述，使用 JS 定时器防空跑策略可以让定时器跟着屏幕刷新走每帧只检查一次，到期定时器批量触发合并为一次 JSI 调用，没有定时器时不请求帧，后台时取消唤醒让 CPU 进入低功耗状态。开发时推荐使用以下策略，享受 CPU/电池节省和帧率稳定效果：

| 策略 | 模式 | 典型实现 |
|------|------|----------|
| **帧对齐调度** | 短间隔定时器跟随 VSync 信号触发，每帧最多检查一次 | `HarmonyTimerRegistry::scheduleWakeUp`（delay < 1s → VSyncListener） |
| **批量触发** | 所有到期定时器合并为一次 `callTimers` 调用 | `HarmonyTimerRegistry::triggerExpiredTimers` |
| **分级调度** | 短间隔走 VSync，长间隔走 delayedTask 精确等待 | `MINIMUM_SLEEP_DELAY = 1s` 阈值区分 |
| **`requestAnimationFrame` 帧驱动** | 从每毫秒触发改为每帧一次 VSync 驱动 | `duration=0` 走 VSync 路径，`setTimeout(fn,0)` 至少等到下一帧 |
| **按需请求帧** | 只在有到期定时器时请求 VSync，空闲帧零开销 | `VSyncListener` 原子防重复，`cancelWakeUp` 后台取消 |
| **requestIdleCallback** | 低优先级任务在帧空闲时间执行 | `IdleCallbacksCxxTurboModule`（帧前紧急、帧后闲） |

---

## 九、示例代码

### HarmonyTimerRegistry — 定时器数据结构

```cpp
struct Timer {
    uint32_t id;
    double deadlineMs;  // 绝对到期时间戳(ms)
    double durationMs;  // 定时器间隔(ms)
    bool repeats;       // 是否循环
};
```

`HarmonyTimerRegistry.h`

### VSyncListener — 原子防重复帧请求

```cpp
void VSyncListener::scheduleNextVsync() {
    auto alreadyScheduled = m_scheduled.exchange(true);
    if (alreadyScheduled) return;  // 已请求帧，不重复请求
    m_vsyncHandle.requestFrame(callback, data);
}
```

`VSyncListener.h`

### 定时器完整调度流程

```
创建定时器 → createTimer() → 存入 m_activeTimerById → scheduleWakeUp()
                                                    ↓
                    delay ≥ 1s? → runDelayedTask 精确等待到期 → triggerExpiredTimers()
                    delay < 1s? → VSyncListener::requestFrame → VSync 到来
                                                    ↓
                              triggerExpiredTimers() → 找出所有 deadline ≤ now 的定时器
                                                    ↓
                              批量 callTimers([A, B, C]) → 一次 JSI 调用 → 一次 React 渲染
                                                    ↓
                              循环定时器: deadline += duration → 下一周期
                              一次性定时器: 从 m_activeTimerById 移除
                                                    ↓
                              还有未到期定时器? → scheduleWakeUp() → 继续调度
                              无定时器? → 取消 VSync 订阅 → 雙开销
```

### requestIdleCallback — 帧空闲时间利用

```cpp
// IdleCallbacksCxxTurboModule
// 帧开始: triggerExpiredTimers() 处理高优先级定时器
// 帧结束: requestIdleCallback 利用剩余时间执行低优先级任务
// 形成"帧前紧急、帧后闲"的完整调度策略
```

`IdleCallbacksCxxTurboModule.cpp`

### 后台生命周期管控

```cpp
// 进入后台
cancelWakeUp();  // 取消所有 VSync 和 delayedTask 唤醒
// CPU 进入低功耗状态，定时器不再触发

// 回到前台
resumeTimers();  // 恢复定时器调度
// 为所有活跃定时器重新计算 deadline 并 scheduleWakeUp()
```

`HarmonyTimerRegistry.cpp`