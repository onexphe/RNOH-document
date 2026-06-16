# FlashList 白屏问题 Patch 修复说明

## 适用版本

- 当前 RNOH 版本：**0.77.59**
- 修复目标版本：**0.77.64**（或手动合入以下 Patch）

## 问题现象

使用 FlashList 滑动列表时出现异常跳动，快速跳动导致白屏。日志中可见 `AutoLayoutView` / `CellContainer` 相关的渲染异常。

## 根因

FlashList 的 `AutoLayoutView` 和 `CellContainer` 组件依赖严格的先后顺序计算布局偏移。RNOH 0.77.59 开启了并行渲染，导致：

1. **并行渲染打乱布局时序** — Create/Update/Insert 指令并行执行，偏移计算顺序被破坏 → 列表项跳跃
2. **scrollTo 修正每帧触发** — `adjustVisibleContentPosition` 无条件触发 `scrollTo`，即使 contentSize 未变也每帧强制修正视口 → 跳动加剧

两者叠加，渲染管线来不及完成帧 → 白屏。

## 0.77.59 修复状态

| 修复 | 0.77.59 状态 | 说明 |
|------|-------------|------|
| Create 重排安全化（`06b1b7794`） | ✅ 已合入 | Create mutation 依赖顺序正确 |
| **FlashList 串行渲染**（`8c1ed3ff8`） | ❌ **缺失** | FlashList 组件仍并行渲染 → 白屏 |
| **scrollTo 条件化**（`7721c456d`） | ❌ **缺失** | 视口修正每帧触发 → 跳动 |
| horizontal prop 修复（`5e7327d97`） | ❌ 缺失 | ScrollView 方向可能错乱（次要） |

**结论**：仅有 Create 重排安全化不足以消除白屏，必须合入串行渲染和 scrollTo 条件化两个 Patch。

---

## Patch 一：FlashList 串行渲染

### 提交

`8c1ed3ff8` — "fix: When there is a flashlist on the surface, disable the parallelization of instructions."

### 改动文件

`SchedulerDelegate.cpp` / `SchedulerDelegate.h`

### 改动说明

当检测到 Surface 上存在 FlashList 组件（`AutoLayoutView` 或 `CellContainer`）时，强制该 Surface 的所有事务切换为串行执行，避免并行渲染打乱布局时序。

### 新增内容

| 新增项 | 说明 |
|--------|------|
| `isFlashListSerialOnlyComponent()` | 检测组件名是否为 `AutoLayoutView` 或 `CellContainer` |
| `shouldForceSerialForSurface()` | 遍历事务 mutations，若含 FlashList 组件则记录 SurfaceId 到缓存 |
| `g_surfaceSerialForce` | `unordered_map<SurfaceId, time_point>`，记录含 FlashList 的 Surface，带 TTL（10分钟）和容量上限（128）自动淘汰 |
| `pruneSurfaceSerialForceLocked()` | 超量时按 TTL 清理过期 Surface 记录 |

### 关键逻辑变更

```cpp
// 之前（0.77.59）：并行渲染无条件开启
if (IsParallelizationEnabled()) {
    // 所有 Surface 都走并行路径
    ...
}

// 之后（Patch）：检测到 FlashList 时强制串行
const bool forceSerialForSurface =
    shouldForceSerialForSurface(surfaceId, transaction->getMutations());
if (IsParallelizationEnabled() && !forceSerialForSurface) {
    // 非 FlashList Surface 走并行路径
    ...
} else {
    // FlashList Surface 走串行路径
    ...
}
```

---

## Patch 二：scrollTo 修正条件化

### 提交

`7721c456d` — "Fix: Scrolling component with mvcp enabled, normal scrolling exhibits jumping position issue"

### 改动文件

`ScrollViewComponentInstance.cpp` / `ScrollViewComponentInstance.h`

### 改动说明

新增 `m_contentSizeChanged` 标记，`adjustVisibleContentPosition()` 中的 `scrollTo` 修正仅在 contentSize 真正变化时触发，避免 FlashList 高频更新时每帧都强制修正视口。

### 关键逻辑变更

```cpp
// 之前（0.77.59）：任何 delta != 0 都触发 scrollTo 修正
if (deltaY != 0) {
    m_scrollNode.scrollTo(...);
}

// 之后（Patch）：只在 contentSize 变化时修正
m_contentSizeChanged = m_contentSize != stateData.getContentSize();
if (deltaY != 0 && m_contentSizeChanged) {
    m_scrollNode.scrollTo(...);
}
```

### 新增成员变量

`ScrollViewComponentInstance.h` 中新增：

```cpp
bool m_contentSizeChanged{false};
```

---

## 修复协同效果

两个 Patch 协同作用：

1. **串行渲染** → 防止 FlashList mutation 乱序执行，布局偏移计算顺序正确
2. **scrollTo 条件化** → 防止高频更新时的视口跳跃，仅在内容尺寸变化时修正

二者共同消除 FlashList 滑动时的跳动和白屏现象。

---

## 合入方式

### 方案一：升级 RNOH 版本（推荐）

直接升级到 **v0.77.64**，两个核心修复均已包含。

```sh
# 更新 RNOH 依赖
ohpm update @react-native-oh/react-native-harmony
# 确保 oh-package.json5 中版本为 0.77.64
```

升级后必须重新编译原生库（参考问题十）：

1. DevEco Studio → Build → Clean Project
2. DevEco Studio → Build → Build Hap(s)/APP(s)

### 方案二：手动合入 Patch

如无法升级版本，需手动从 RNOH v0.77.64 源码中提取以下提交的改动并合入：

| Patch | 提交 | 源码路径 |
|--------|------|----------|
| FlashList 串行渲染 | `8c1ed3ff8` | `SchedulerDelegate.cpp/h` |
| scrollTo 条件化 | `7721c456d` | `ScrollViewComponentInstance.cpp/h` |

合入步骤：

1. 从 RNOH v0.77.64 仓库获取对应提交的 diff
2. 将改动合入项目中对应的 C++ 文件
3. Clean Project → Build Hap(s)/APP(s)

---

## 验证方法

1. 启动含 FlashList 的页面
2. 快速滑动列表，观察是否仍有跳动或白屏
3. 检查 Logcat 中是否还有 `AutoLayoutView` / `CellContainer` 相关的异常日志
4. 反复下拉刷新，确认列表恢复正常显示
