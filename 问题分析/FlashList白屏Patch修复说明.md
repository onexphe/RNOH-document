# FlashList 白屏问题 Patch 修复说明

## 适用版本

- 受影响RNOH 版本：**0.77.11 - 0.77.59**
- 修复目标版本：**0.77.64 - 0.77.72** （或手动合入以下 Patch）

## 问题现象

使用 FlashList 滑动列表时出现异常跳动，快速跳动导致白屏。

## 根因

`adjustVisibleContentPosition` 无条件触发 `scrollTo`，即使 contentSize 未变也每帧强制修正视口。FlashList 滑动时会频繁触发 `onStateChanged`，导致视口位置被反复强制调整 → 跳动 → 渲染管线来不及完成帧 → 白屏。

## 关键修复合入

| 修复 | 首次合入版本 | 适用范围 |
|------|-------------|----------|
| **scrollTo 条件化**（`7721c456d`） | **0.77.64** | 所有版本通用 |

**结论**：scrollTo 条件化是消除 FlashList 跳动/白屏的必要修复。

---

## Patch：scrollTo 修正条件化

### 提交

`7721c456d` — "Fix: Scrolling component with mvcp enabled, normal scrolling exhibits jumping position issue"

对应PR： https://gitcode.com/CPF-RN/ohos_react_native/pull/2581

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

## 修复效果

scrollTo 条件化防止高频更新时的视口跳跃，仅在内容尺寸真正变化时修正，消除 FlashList 滑动时的跳动和白屏现象。

> **补充**：如果你的版本开启了并行渲染（`IsParallelizationEnabled()`），还需额外合入串行渲染 Patch（提交 `8c1ed3ff8`），改动文件为 `SchedulerDelegate.cpp/h`。

---

## 合入方式

### 方案一：升级 RNOH 版本（推荐）

直接升级到 **v0.77.64**及以上版本，scrollTo 条件化修复已包含。

### 方案二：手动合入 Patch

如无法升级版本，需手动从 RNOH v0.77.64 源码中提取以下提交的改动并合入：

| Patch | 提交 | 源码路径 |
|--------|------|----------|
| scrollTo 条件化 | `7721c456d` | `ScrollViewComponentInstance.cpp/h` |

合入步骤：

1. 从 RNOH v0.77.64 仓库获取对应提交的 diff
2. 将改动合入项目中对应的 C++ 文件

---

## 验证方法

1. 启动含 FlashList 的页面
2. 快速滑动列表，观察是否仍有跳动或白屏
4. 反复下拉刷新，确认列表恢复正常显示
