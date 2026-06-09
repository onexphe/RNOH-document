# 可视区外JS轮播图防空跑 -- 深入分析文档

> **需求目标**: 当JS侧的轮播图(Carousel/Swiper)不在可视区时,避免其定时器(setInterval/requestAnimationFrame)空跑消耗CPU和电池
> **核心提交**: `2cec6326f`(call js timers once per frame), `89ad31f10`(timer flicker fix), `53f92f864`(requestIdleCallback)
> **后续演进**: HarmonyTimerRegistry替代TimingTurboModule
> **关联需求**: UiTick事件对齐、减少线程间通信

---

## 1. 通俗解释:什么是"空跑"?

想象一个电商首页,上方有一个广告轮播图,用`setInterval`每3秒切换一张图片。当用户往下滚动页面,轮播图不在屏幕可视区了,但`setInterval`仍在每3秒执行一次 -- **JS线程被唤醒、执行回调、更新图片状态、触发React重新渲染**,渲染出来的结果用户根本看不到,全是白忙。

如果页面同时有3个轮播图,那就是3个定时器每隔几秒各空跑一次,**每帧可能被JS线程唤醒3次**。

### 1.1 旧实现:每个定时器独立调度

```
旧实现: Timer A -> runDelayedTask(3s, triggerA)  // 3秒后触发A
旧实现: Timer B -> runDelayedTask(3s, triggerB)  // 3秒后触发B
旧实现: Timer C -> runDelayedTask(3s, triggerC)  // 3秒后触发C

结果: 每帧最多被JS线程唤醒3次, 3次JSI调用, 3次React渲染
```

`requestAnimationFrame`情况更糟 -- 它在RNOH旧实现中被当作`setTimeout(..., 1)`处理,意味着**每毫秒都可能触发一次回调**,动画每帧跑120次。

### 1.2 新实现:定时器与VSync帧对齐,每帧只触发一次

```
新实现: 所有定时器共享一个VSync驱动的唤醒机制
新实现: scheduleWakeUp() -> 下一帧VSync到来时 -> triggerExpiredTimers()
新实现: triggerExpiredTimers() -> 找出所有到期定时器 -> callTimers([A, B, C])  // 一次JSI调用
新实现:                                                    -> 一次React渲染
```

**关键区别**:
- 旧: 每个定时器独立触发, N个定时器 = N次JS线程唤醒 + N次JSI调用
- 新: 所有定时器合并触发, N个定时器 = 1次JS线程唤醒 + 1次JSI调用

---

## 2. 代码实现深入分析

### 2.1 核心架构: HarmonyTimerRegistry

`HarmonyTimerRegistry`(替代旧版`TimingTurboModule`)是整个优化的核心实现,位于`RNOH/HarmonyTimerRegistry.h`和`.cpp`。

#### 2.1.1 Timer数据结构

```cpp
struct Timer {
    uint32_t id;
    double deadlineMs;  // 绝对到期时间戳(ms)
    double durationMs;  // 定时器间隔(ms)
    bool repeats;       // 是否循环
};
```

旧版`TimingTurboModule`中定时器存储的是`jsSchedulingTime`(JS侧创建时间),用于计算相对delay。新版改为存储`deadlineMs`(绝对到期时间),这样在`triggerExpiredTimers`中可以直接用`deadlineMs <= now`判断是否到期,不需要再计算delay。

#### 2.1.2 定时器创建

```cpp
void HarmonyTimerRegistry::createTimer(uint32_t timerId, double delayMs) {
    assertJSThread();
    auto now = getMillisSinceEpoch();
    auto deadline = now + delayMs;
    m_activeTimerById.emplace(timerId, Timer{timerId, deadline, delayMs, false});

    if (isForeground && !m_vsyncListener->isScheduled()) {
        scheduleWakeUp();
    }
}
```

**关键变化**:
- 不再为每个定时器独立创建`runDelayedTask`,只存储到`m_activeTimerById`映射
- 只有当没有VSync已调度时才调用`scheduleWakeUp()`

#### 2.1.3 定时器触发 -- 核心优化

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
        // 按到期时间排序,先到期的先触发
        std::sort(expiredTimers.begin(), expiredTimers.end(),
            [](auto a, auto b) { return a.deadlineMs < b.deadlineMs; });

        // 收集所有到期定时器ID
        std::vector<uint32_t> expiredTimerIds;
        std::transform(expiredTimers.begin(), expiredTimers.end(),
            std::back_inserter(expiredTimerIds),
            [](auto timer) { return timer.id; });

        triggerTimers(expiredTimerIds);  // <- 批量触发!
    }

    if (!m_activeTimerById.empty()) {
        scheduleWakeUp();  // <- 还有未到期定时器,继续调度唤醒
    }
}
```

`triggerTimers`将所有到期定时器的ID合并为一次`JSTimers.callTimers`调用:

```cpp
void HarmonyTimerRegistry::triggerTimers(std::vector<uint32_t> const& timerIds) {
    if (auto timerManager = m_timerManager.lock()) {
        for (auto timerId : timerIds) {
            timerManager->callTimer(timerId);
        }
    }
    // 更新循环定时器的deadline,移除一次性定时器
    for (auto id : timerIds) {
        auto it = m_activeTimerById.find(id);
        if (it != m_activeTimerById.end()) {
            auto& timer = it->second;
            if (timer.repeats) {
                timer.deadlineMs += timer.durationMs;  // 循环定时器:推到下一周期
            } else {
                m_activeTimerById.erase(it);            // 一次性定时器:移除
            }
        }
    }
}
```

#### 2.1.4 唤醒调度 -- scheduleWakeUp()

这是连接定时器与VSync的关键桥梁,决定了定时器在何时被检查:

```cpp
void HarmonyTimerRegistry::scheduleWakeUp() {
    constexpr double MINIMUM_SLEEP_DELAY = 1000.;  // <- 1秒阈值

    auto nextDeadline = getNextDeadline();
    auto now = getMillisSinceEpoch();
    auto delay = nextDeadline - now;

    if (delay >= MINIMUM_SLEEP_DELAY) {
        // <- 下一个定时器到期时间 >= 1秒: 用delayedTask精确等待
        if (nextDeadline >= m_nextTimerDeadline) return;  // 已有更早的wakeUp
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
        // <- 下一个定时器到期时间 < 1秒: 用VSync帧回调检查
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

**策略解析**:

| 定时器到期时间 | 调度方式 | 示例 |
|--------------|--------|------|
| >= 1秒 | `runDelayedTask`精确等待 | `setTimeout(fn, 3000)` |
| < 1秒 | VSync帧回调检查 | `setInterval(fn, 100)` / `requestAnimationFrame` |

**为什么<1秒用VSync?** 因为短间隔定时器可能在帧中间到期,如果用delayedTask,可能每毫秒都触发一次检查。用VSync则最多每帧检查一次,与屏幕刷新对齐。

这也解释了`requestAnimationFrame`为什么不再每毫秒触发 -- 在旧实现中它被当作`setTimeout(..., 1)`处理,新实现中它跟其他短间隔定时器一样被VSync驱动。

### 2.2 旧版TimingTurboModule的架构

旧版为每个定时器独立创建`runDelayedTask`:

```cpp
// 旧版 createTimer
auto timerTask = m_ctx.taskExecutor->runDelayedTask(
    TaskThread::JS,
    [weakSelf = weak_from_this(), id, repeats] {
        auto self = weakSelf.lock();
        if (!self) return;
        self->triggerTimer(id);        // <- 单个定时器触发
    },
    duration - delay,
    repeats ? duration : 0);
m_timerTaskById.emplace(id, timerTask);
```

**问题**:
1. 每个定时器独立调度 -> N个定时器 = N次JS线程唤醒
2. `requestAnimationFrame`(setTimeout 1ms) -> 可能每毫秒触发一次
3. `resumeTimers`时为所有定时器重新创建delayedTask -> 大量task创建开销

### 2.3 VSyncListener

`VSyncListener`(RNOH/VSyncListener.h/cpp)是轻量级VSync监听器,使用原子`m_scheduled`标志防止重复请求帧:

```cpp
void VSyncListener::scheduleNextVsync() {
    auto alreadyScheduled = m_scheduled.exchange(true);
    if (alreadyScheduled) return;  // <- 已请求,不重复
    m_vsyncHandle.requestFrame(callback, data);
}
```

这确保即使多个短间隔定时器在同一帧内到期,也只请求一次VSync帧。

---

## 3. 演进脉络

### Phase 1 -- 定时器架构从ArkTS迁移到C++(2024-07)

| 提交 | 日期 | 说明 |
|------|------|------|
| `3ff649446` | 2024-07-03 | 重写`TimingTurboModule`,使用C++ `runDelayedTask`替代ArkTS定时器 |

**旧流程**(每个定时器周期需要2次跨线程通信):
```
JS线程: callSync("createTimer") -> [跨线程阻塞] -> ArkTS线程: setTimeout/setInterval
ArkTS线程: 定时器到期 -> [跨线程消息] -> JS线程: JSTimers.callTimers()
```

**新流程**(零跨线程通信):
```
JS线程: createTimer() -> 存入C++ map -> runDelayedTask到期 -> 直接在同一线程triggerTimer()
```

### Phase 2 -- VSync帧对齐+批量触发(2024-10)

| 提交 | 日期 | 说明 |
|------|------|------|
| `2cec6326f` | 2024-10-11 | **核心优化**: 每帧只触发一次定时器,短间隔定时器用VSync驱动 |
| `388e7963c` | 2024-10-15 | 合入主分支 |

commit message中明确指出:
> iOS和Android在帧回调中触发定时器。此PR使我们的行为与其他平台一致。
> 这也提升了使用`Animated`API(未启用native driver)时的性能:因为`requestAnimationFrame`被实现为`setTimeout(..., 1)`,在我们的实现中导致更新每毫秒触发一次。

### Phase 3 -- requestIdleCallback补充(2025-08)

| 提交 | 日期 | 说明 |
|------|------|------|
| `53f92f864` | 2025-08-26 | 实现`requestIdleCallback`,将低优先级任务放在帧空闲时间执行 |

`IdleCallbacksCxxTurboModule`(207行)在帧空闲时执行非紧急任务,与定时器帧对齐互补:
- 定时器帧对齐: 确保定时器在帧开始时集中触发
- `requestIdleCallback`: 确保低优先级任务在帧结束时利用剩余时间

---

## 4. 稳定性修复

| 问题 | 修复提交 | 日期 | 说明 |
|------|----------|------|------|
| TimingTurboModule析构崩溃 | `f36aace5b` / `3f57947f2` / `6a6d8ba1a` | 2024-10-17 | 添加`weak_ptr`保护,防止析构后回调 |
| 定时器实现错误导致崩溃 | `a6a72c733` | 2024-10-21 | 修复不正确的Timing实现 |
| setTimeout(fn,0)立即触发导致动画闪烁 | `89ad31f10` | 2025-05-28 | 移除`delayMs==0`立即触发优化,改为至少等到下一帧 |
| setInterval在cancelWakeUp时偶发停止 | `29803c757` | 2026-06-01 | 修复在wakeUp回调内调用cancelWakeUp时setInterval停止执行 |

`setTimeout(fn, 0)`旧版被立即触发(绕过VSync),导致`requestAnimationFrame`每毫秒执行。修复后,`duration=0`的定时器也走VSync帧对齐流程,至少等到下一帧才触发。

---

## 5. 性能收益量化

### 5.1 定时器触发次数对比

假设页面有5个`setInterval`(间隔100ms-3000ms),2个`requestAnimationFrame`:

| 场景 | 旧实现 | 新实现 | 提升倍数 |
|------|--------|--------|---------|
| 每帧定时器到期数 | 最多7次独立触发 | 1次批量触发 | ~7x |
| 每帧JS线程唤醒数 | 最多7次 | 1次 | ~7x |
| `requestAnimationFrame` | 每毫秒触发 | 每帧1次 | ~120x(120Hz) |

**综合节省**: 在典型页面中,定时器相关开销降低**5-120倍**。

### 5.2 CPU/电池影响

空跑定时器意味着CPU每帧被频繁唤醒。在移动设备上,这直接影响电池续航和发热。帧对齐后:
- CPU每帧只唤醒1次(而非N次)
- 空闲帧不唤醒(无到期定时器时不请求VSync)
- 后台时`cancelWakeUp()`取消所有唤醒,CPU进入低功耗状态

---

## 6. 与其他需求的关联

| 关联需求 | 关联点 |
|----------|--------|
| **UiTick事件对齐** | `scheduleWakeUp`中短间隔定时器使用VSync帧回调,是"UiTick对齐"在定时器层面的具体应用 |
| **减少线程间通信** | 批量`callTimers`将N次JSI调用合并为1次,直接减少线程间通信次数;C++ TimerRegistry替代ArkTS TimerModule,消除了JS-ArkTS-JS的跨线程循环 |
| **requestIdleCallback** | 低优先级任务放在帧空闲时间,与定时器帧对齐形成"帧前紧急、帧后闲"的完整调度策略 |

三个需求的关系图:

```
减少线程间通信          UiTick事件对齐           可视区外JS轮播图防空跑
    |                     |                         |
    v                     v                         v
C++ TimerRegistry     VSync帧驱动调度          帧对齐+批量触发+生命周期管控
(消除JS-ArkTS跨线程)  (定时器跟着VSync走)       (不在可视区不空跑)
    |                     |                         |
    +---------------------+-------------------------+
                          |
                    共同基础: EventLoopTaskRunner + VSyncListener + UITicker
```

---

## 7. 一句话总结

**可视区外JS轮播图防空跑 = 让所有定时器跟着屏幕刷新走,每帧只检查一次,到期的一批一起触发,没到期的不触发,没有定时器时不请求帧。**

- 跟着屏幕走 -> 帧同步,不空跑
- 每帧只检查一次 -> N个定时器 = 1次唤醒,不再N次
- 到期一起触发 -> 1次JSI调用替代N次
- 没到期不触发 -> 空闲帧零开销
- 没有定时器不请求帧 -> 静态页面零VSync请求
- 后台时取消唤醒 -> CPU进入低功耗状态