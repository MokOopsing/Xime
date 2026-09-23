# Xime 键盘实现与悬浮键盘架构

本文面向需要改造或迁移键盘前端的 AI/开发者，说明 Xime 输入法键盘的组成、状态流、按键事件流、悬浮键盘实现方式，以及迁移到 Fcitx5 Android 时可以复用和必须重写的部分。

## 1. 总体结论

Xime 的键盘不是一个独立的 Android 悬浮窗，也不依赖系统悬浮窗权限。它仍然是 `InputMethodService` 的输入法窗口，只是在这个窗口内部绘制一个可以移动的 Compose 卡片。

```text
Android InputMethodService
  └── IME Window（全屏布局）
        └── VoiceKeyboardContainer（FrameLayout）
              └── ComposeView
                    └── KeyboardView
                          └── FloatingKeyboardContainer（悬浮模式）
                                └── KeyboardLayout / T9 / 手写 / 数字 / 符号等
```

普通模式和悬浮模式的主要区别：

| 项目 | 普通模式 | 悬浮模式 |
|---|---|---|
| IME 窗口 | 全屏窗口，但容器高度等于键盘高度 | 全屏窗口 |
| 键盘位置 | 底部对齐 | 底部对齐后向上偏移 |
| 宿主应用布局 | 通过 IME insets 避让键盘 | 不按键盘卡片高度避让 |
| 可触摸区域 | 输入法可见区域 | 只声明悬浮卡片矩形区域 |
| 键盘背景 | 外层 Compose 绘制 | 卡片内部绘制 |
| 位置保存 | 不需要 | `SharedPreferences` 保存 X/Y 偏移 |

## 2. 关键源文件

### Android 输入法窗口

- [XimeInputMethodService.kt](../app/src/main/java/com/kingzcheung/xime/service/XimeInputMethodService.kt)
  - 创建 IME 输入视图
  - 配置 IME Window
  - 计算屏幕、状态栏、导航栏和键盘高度
  - 上报 `InputMethodService.Insets`
  - 组装 `KeyboardView` 和服务层回调
- [VoiceKeyboardContainer.kt](../app/src/main/java/com/kingzcheung/xime/service/VoiceKeyboardContainer.kt)
  - 最外层 `FrameLayout`
  - 动态修改物理容器高度
  - 处理语音模式下的触摸区域
- [ImeWindowInsets.kt](../app/src/main/java/com/kingzcheung/xime/service/ImeWindowInsets.kt)
  - 查询状态栏、导航栏和可见导航栏高度

### Compose 键盘视图

- [KeyboardView.kt](../app/src/main/java/com/kingzcheung/xime/ui/keyboard/KeyboardView.kt)
  - 键盘根组件
  - 按页面和布局状态选择具体键盘
  - 包裹 `FloatingKeyboardContainer`
- [FloatingKeyboardContainer.kt](../app/src/main/java/com/kingzcheung/xime/ui/keyboard/FloatingKeyboardContainer.kt)
  - 悬浮卡片的尺寸、底部对齐、偏移、圆角、阴影和拖动条
- [KeyboardLayoutScreen.kt](../app/src/main/java/com/kingzcheung/xime/ui/keyboard/KeyboardLayoutScreen.kt)
  - 根据 `KeyboardLayoutState` 选择中文、英文、数字、符号、笔画、T9 或手写键盘
- [KeyboardLayout.kt](../app/src/main/java/com/kingzcheung/xime/ui/keyboard/KeyboardLayout.kt)
  - 中文/英文全键盘
  - 普通按键、手势、空格、回车、退格和工具栏
- [KeyboardCallbacks.kt](../app/src/main/java/com/kingzcheung/xime/ui/keyboard/KeyboardCallbacks.kt)
  - UI 到服务层的回调协议

### 状态和路由

- [KeyboardViewModel.kt](../app/src/main/java/com/kingzcheung/xime/viewmodel/KeyboardViewModel.kt)
  - 管理键盘布局、页面、大小写、Overlay 和布局切换
- [InputUIState.kt](../app/src/main/java/com/kingzcheung/xime/service/InputUIState.kt)
  - 服务层维护的输入法 UI 状态
- [ImeKeyboardCallbacks.kt](../app/src/main/java/com/kingzcheung/xime/service/ImeKeyboardCallbacks.kt)
  - 把服务层能力组装成 `KeyboardCallbacks`
- [ImeKeyRouter.kt](../app/src/main/java/com/kingzcheung/xime/service/ImeKeyRouter.kt)
  - 按键的核心路由
  - 普通字符、候选、空格、回车、退格、工具面板和特殊输入状态处理
- [SettingsPreferences.kt](../app/src/main/java/com/kingzcheung/xime/settings/SettingsPreferences.kt)
  - 键盘高度、底部留白、悬浮模式和悬浮位置的持久化

## 3. 输入视图创建流程

入口是 `XimeInputMethodService.onCreateInputView()`：

```text
onCreateInputView()
  1. 创建 VoiceKeyboardContainer
  2. 关闭容器的 clipChildren，允许候选编码气泡越界绘制
  3. 创建 ComposeView
  4. 在 setContent 中读取 InputUIState 和 CandidateState
  5. 计算屏幕/导航栏/键盘/面板尺寸
  6. 通过 SideEffect 调用 keyboardContainer.updateHeight()
  7. 创建 KeyboardCallbacks
  8. 创建 KeyboardView
  9. 将 ComposeView 加入 VoiceKeyboardContainer
```

`setInputView()` 之后，输入法视图被放置在 `inputArea` 中并设置为底部对齐。普通模式使用实际键盘高度作为容器高度；悬浮模式则使用全屏高度，让 Compose 自己决定卡片位置。

## 4. 状态分层

### 4.1 服务层输入法状态：`InputUIState`

服务层状态包括：

- 中英文模式和当前方案
- 深色模式、主题、键盘高度、底部留白
- 悬浮模式及 `floatingOffsetX/Y`
- 语音状态
- 当前输入会话 ID
- T9 重置信号和右侧候选选择状态
- 快捷发送表单和 AI 工具面板
- 插件工具栏按钮

这个状态通过 `StateFlow`/Compose 状态传给 UI。它描述“输入法当前处于什么状态”，不直接负责候选词内容。

### 4.2 键盘布局状态：`KeyboardViewModel`

`KeyboardViewModel` 管理中文、英文、数字、普通符号、笔画、T9 和手写布局，以及主键盘、Overlay、候选展开页、Shift 和大小写状态。

布局状态决定“渲染哪一种键盘”，服务层输入状态决定“这些按键如何处理”。

### 4.3 候选状态：`CandidateState`

候选状态独立于布局状态，通常包含：

- `inputText`、`preeditText`
- `candidates`、`candidateComments`
- `associationCandidates`
- `pendingEnglishText`
- `isComposing`
- 翻页状态和插件候选动作

这样做的目的是：长按退格或候选刷新时，只重组候选栏，不必重组整个键盘按键区域。

## 5. 键盘组件层级

```text
KeyboardView
  ├── FloatingKeyboardContainer
  │     ├── DragBar（仅悬浮模式）
  │     └── keyboardContent
  ├── CandidateBar / FloatingCandidateBar
  ├── KeyboardLayoutScreen
  │     ├── KeyboardLayout（中文/英文）
  │     ├── T9KeyboardLayout
  │     ├── StrokeKeyboardLayout
  │     ├── HandwritingKeyboardLayout
  │     ├── NumberKeyboardLayout
  │     └── CommonSymbolKeyboardLayout
  ├── MenuBar
  ├── Overlay 页面
  └── 快捷发送 / 工具面板 / 调整高度覆盖层
```

具体布局产生逻辑键名，例如 `"a"`、`"space"`、`"delete"`、`"enter"`、`"ime_switch"`、`"number"` 和 `"common_symbol"`。UI 不直接操作 `InputConnection`，而是调用 `KeyboardCallbacks`，再由服务层决定交给 Rime、直接提交，还是交给特殊输入控制器。

## 6. 按键事件流

### 6.1 普通按键

```text
Compose 按键
  -> KeyboardCallbacks.onKeyPress(key, isShifted)
  -> ImeKeyRouter.handleKeyPress()
  -> keyProcessingDispatcher / keyJobs
  -> RimeEngine.processKeyAndGetResult()
  -> 更新 CandidateState
  -> InputConnection.commitText()（如有 committedText）
```

`keyJobs` 用于保证 Rime 操作串行，避免快速输入时 JNI、候选读取和 UI 状态更新互相交错。

### 6.2 物理键盘

`XimeInputMethodService.onKeyDown()` 将 Android `KeyEvent` 转换为逻辑键名，再进入 `ImeKeyRouter.handleKeyPress()`。方向键、数字选词、空格/回车选中候选等硬件候选导航逻辑有部分直接调用候选选择方法。

### 6.3 按下、抬起和点击

- `onKeyPressDown`：音效、震动和按下反馈
- `onKeyPress`：实际逻辑动作
- `onKeyRelease`：抬起反馈

长按和滑动由具体 Compose 按键处理，最终转换成逻辑键名或 `GestureAction`。

### 6.4 系统取消触摸

系统手势可能导致 Compose 收不到正常的 UP/CANCEL，从而遗留长按协程或按下状态。`VoiceKeyboardContainer` 捕获 `ACTION_CANCEL` 后递增 `swipeCancelEpoch`，`KeyboardLayoutScreen` 用 `key(keyboardState)` 和 `key(uiState.swipeCancelEpoch)` 重建活动键盘，取消内部手势协程。

## 7. 悬浮键盘实现细节

### 7.1 核心模型

```text
系统 IME 全屏窗口
  -> Compose 全屏容器
      -> BoxWithConstraints(fillMaxSize)
          -> 底部对齐的键盘卡片
              -> x = offsetX
              -> y = -offsetY
```

这不是 `WindowManager.addView()`，不需要悬浮窗权限，也没有第二个 Android Window。

### 7.2 卡片尺寸和定位

`KeyboardView` 使用屏幕短边计算宽度：

```text
portraitWidth = min(screenWidth, screenHeight)
cardWidth = portraitWidth × 0.85
```

悬浮模式下，键盘高度使用保存的键盘高度，并乘以约 `0.85` 的缩放因子；卡片顶部额外增加约 `18dp` 的拖动条高度。

`FloatingKeyboardContainer` 使用：

- `fillMaxWidth(scaleFactor)` 设置宽度
- `height(cardTotalHeight)` 设置高度
- `Alignment.BottomCenter` 贴底对齐
- `offset(x = offsetX, y = -offsetY)` 移动
- `shadow(12.dp)` 和圆角绘制卡片外观

### 7.3 拖动流

```text
DragBar.detectDragGestures
  -> 像素拖动量转换为 dp
  -> onDrag(dx, -dy)
  -> onFloatingKeyboardDrag(dx, dy)
  -> 更新 InputUIState.floatingOffsetX/Y
  -> Compose 重组
  -> onFloatingKeyboardDragEnd()
  -> SharedPreferences 保存位置
```

Y 轴取反是因为视觉上向上拖动应增加“距离底部的偏移量”。

横向边界为：

```text
halfMargin = (screenWidth - cardWidth) / 2
offsetX ∈ [-halfMargin, halfMargin]
```

纵向边界为：

```text
maxOffsetY = max(screenHeight - cardHeight, floatingMinY)
offsetY ∈ [0, maxOffsetY]
```

位置按竖屏/横屏分别保存：

```text
floating_mode
floating_offset_x
floating_offset_y
floating_mode_landscape
floating_offset_x_landscape
floating_offset_y_landscape
```

### 7.4 IME Insets 和触摸区域

普通模式中，`onComputeInsets()` 将键盘容器顶部作为 `contentTopInsets` 和 `visibleTopInsets`，让宿主应用避让键盘。

悬浮模式中：

```kotlin
contentTopInsets = displayHeight
visibleTopInsets = displayHeight
touchableInsets = Insets.TOUCHABLE_INSETS_REGION
```

然后将 `touchableRegion` 设置为悬浮卡片的屏幕矩形。结果是：

1. 宿主应用不会按普通键盘高度整体上移。
2. IME 窗口仍然是全屏的。
3. 只有卡片区域拦截触摸。
4. 卡片外区域可以继续操作宿主应用。

卡片触摸区域大致计算为：

```text
left   = (screenWidth - cardWidth) / 2 + offsetX
top    = imeContentBottom - cardHeight - offsetY
right  = left + cardWidth
bottom = imeContentBottom - offsetY
```

`cardHeight` 使用 `currentEffectiveKeyboardHeight`，该值来自初始估算，并在 `onCardPositioned` 中根据 Compose 实际布局结果修正。

## 8. 普通模式的高度和导航栏

普通模式使用：

```text
键盘内容高度
  + 用户配置的 bottom padding
  + 导航栏留白
  = VoiceKeyboardContainer 的物理高度
```

键盘内容通过负的 Y offset 向上移动，导航栏区域由外层背景和 Spacer 覆盖。代码会根据实际底部 inset 做缩减：竖屏默认减少约 `8dp`，横屏约 `16dp`，三键导航栏再额外减少一档；完全检测不到 inset 时使用最小兜底高度。

迁移到其他 IME 框架时，不应直接复制这些固定数值，而应重新确认目标框架的 WindowInsets 和导航栏行为。

## 9. 页面和布局切换

```text
KeyboardPage.Main
  ├── FULL
  ├── T9 / 九键
  ├── HANDWRITING
  └── 其他主布局

KeyboardPage.Overlay
  ├── Emoji
  ├── Symbol
  ├── Clipboard
  ├── ToolPanel
  └── 其他覆盖页
```

数字和普通符号键盘通常作为主键盘布局状态切换；表情、剪贴板等页面属于 Overlay。Overlay 仍在同一个 Compose/IME 输入视图内。

工具面板分两类：

- 输入型面板：增加输入法容器高度，面板在键盘上方，输入焦点时按键路由到面板 EditText。
- 纯展示型面板：作为 Overlay 覆盖键盘内容，不额外撑高输入法窗口。

## 10. 不同输入模式的提交策略

迁移键盘时，不能假设所有按键都走同一条 Rime 路径。

| 模式 | 字符来源 | 提交方式 |
|---|---|---|
| 普通中文 | Rime composition | `RimeEngine.processKey`，读取候选和 `committedText` |
| 英文 | 直接输入 | `InputConnection.commitText`，并维护 `pendingEnglishText` |
| T9 | T9 控制器 + Rime | 独立处理数字、部分提交和候选拼音 |
| 手写 | 手写识别器 | 直接替换或提交识别结果 |
| 数字键盘 | 键盘自身 | 通常直接提交或发送退格 |
| 符号键盘 | 键盘自身 | 直接提交符号 |
| 工具面板 | 面板 EditText | 直接插入面板输入框 |
| 联想候选 | 预测引擎 | 点击或配置允许时直接提交 |

## 11. 当前空格语义

普通空格目前由 [ImeKeyRouter.kt](../app/src/main/java/com/kingzcheung/xime/service/ImeKeyRouter.kt) 分流，不是始终调用 `librime.process_key(0x20, 0)`：

```text
工具面板有焦点
  -> 插入面板 EditText

英文待处理文本存在
  -> 直接提交空格

中文有候选
  -> Xime 选择第一个候选

中文有组合文本但没有候选
  -> 直接提交原始输入

有联想候选且开启空格上屏
  -> 直接提交第一个联想词

其他情况
  -> 直接提交 " "
```

如果迁移目标要求由 Fcitx/Rime schema 完全控制空格，需要重新设计这部分路由，而不能只迁移 UI。

## 12. 迁移到 Fcitx5 Android 的建议分层

### 12.1 可直接复用的 UI 思路

- Compose 键盘组件树
- `KeyboardCallbacks` 形式的 UI/宿主解耦
- `KeyboardLayoutState` 页面和布局状态
- `FloatingKeyboardContainer` 的卡片绘制和拖动手势
- 键盘高度、主题、按键间距和布局配置
- `touchableRegion` 对悬浮键盘的区域限制思路

### 12.2 需要替换的宿主层

- `XimeInputMethodService`
- `VoiceKeyboardContainer`
- `onComputeInsets()`
- `InputConnection` 提交封装
- `WindowInsets` 和导航栏计算
- Rime JNI / `RimeEngine`

Fcitx5 Android 的输入法服务、输入上下文和 commit API 不应直接照搬 Xime 的服务类。

### 12.3 推荐目标结构

```text
FcitxInputMethodService / FcitxInputView
  ├── KeyboardWindowAdapter
  │     ├── 普通模式 inset
  │     ├── 悬浮模式 touchableRegion
  │     └── 容器高度同步
  ├── KeyboardUiState
  ├── KeyboardViewModel
  ├── KeyboardCallbacks
  ├── KeyboardView
  │     └── FloatingKeyboardContainer
  └── FcitxKeyRouter
        ├── processKey
        ├── commitString
        ├── candidate selection
        └── special keyboard modes
```

### 12.4 推荐迁移顺序

1. 普通底部键盘和 Fcitx commit 适配。
2. 中文、英文、退格、回车和候选栏。
3. 数字/符号/表情 Overlay。
4. 悬浮卡片绘制和拖动。
5. IME inset 与 touchable region。
6. T9、手写、语音和插件工具栏。
7. 空格、联想和 schema 自定义按键语义。

先完成第 1 至第 5 步，可以验证悬浮键盘本身，不会把输入引擎差异和 UI 定位问题混在一起。

## 13. 必须验证的行为

### 普通模式

- 宿主输入框是否正确避让键盘
- 三键导航栏和手势导航栏高度是否正确
- 键盘高度调整后 IME inset 是否同步
- 工具面板打开/关闭后窗口高度是否恢复
- 输入法切换和旋转屏幕后布局是否正确

### 悬浮模式

- 宿主应用是否保持原位置，不被整体顶起
- 卡片是否在屏幕边界内
- 卡片外区域是否可以点击宿主应用
- 卡片区域是否完整可点击
- 卡片拖动后触摸区域是否同步
- 旋转屏幕后 X/Y 位置是否使用正确方向的配置
- 键盘高度或面板状态变化后 touchable region 是否更新
- 首次进入悬浮模式时卡片高度估算是否导致点击偏移

### 输入行为

- 候选选择和提交是否重复
- 快速连续输入是否丢键
- 长按退格是否会堆积任务
- 系统手势取消后是否残留按下状态
- 空格、回车和数字选词是否符合目标输入引擎语义

## 14. 当前实现中的迁移注意点

1. `FloatingKeyboardContainer` 接收了 `minOffsetY`，但当前组件内部没有使用它；边界主要由服务层拖动回调限制。
2. 悬浮卡片的实际高度依赖初始估算和 `onGloballyPositioned` 回调，在首次布局或高度变化时需要验证触摸区域同步。
3. Xime 的 IME Window 是全屏的，但宿主避让逻辑由 `onComputeInsets()` 控制；只复制 Compose 卡片而不复制 inset 逻辑，悬浮效果不会完整。
4. `KeyboardCallbacks` 很大，包含 T9、手写、语音、插件和工具面板能力。迁移初期应拆成基础按键回调、候选回调和扩展能力回调，避免把 Xime 的服务依赖整体带入 Fcitx5 Android。
5. 空格、联想词、英文直上屏和 T9 部分提交都属于输入策略，不是单纯的键盘 UI。迁移时应由目标输入引擎重新定义。

## 15. 最小可迁移悬浮键盘模型

如果只需要在 Fcitx5 Android 中复现悬浮键盘外观和交互，最小模型可以是：

```text
全屏 IME 输入视图
  -> BoxWithConstraints(fillMaxSize)
      -> Box(bottomCenter)
          -> width = screenShortSide × 0.85
          -> height = keyboardHeight × scale + dragBarHeight
          -> offset(x, -offsetY)
          -> DragBar 修改 offset
```

宿主层只需要提供：

```text
isFloatingMode
offsetX
offsetY
keyboardHeight
onDrag(dx, dy)
onDragEnd()
onCardPositioned(bounds)
```

如果 Fcitx5 Android 的输入窗口允许修改可触摸区域，再补充：

```text
contentTopInsets = fullHeight
visibleTopInsets = fullHeight
touchableRegion = cardBounds
```

这部分就是 Xime 悬浮键盘最核心、最容易独立迁移的实现。
