---
layout: post
title:  "从按键到“你好”：输入法、操作系统与文本框的完整链路"
date:   2026-09-30 15:00:00 +0800
categories: 输入法 前端 技术
---

## 一、一次中文输入的旅程

你在一个网页的 `<textarea>` 里敲下 `nihao` 五个键，候选窗弹出"1.你好"，你按下空格，页面上出现了"你好"。

这条看似平凡的路径，实际上穿过了至少五个独立的软件世界：键盘驱动、操作系统的事件系统、输入法引擎、GUI 框架的文本输入协议、以及浏览器内核的编辑引擎。每个世界都通过严格的接口契约与相邻世界对话，而 **`<textarea>` 自己，恰恰是什么平台接口都没实现的那一个**。

理解这条链路，不仅能让你明白输入法的原理，还能回答一个实践中非常常见的问题：**为什么在 Canvas 上做编辑器时，必须偷偷藏一个 `<textarea>`？**

## 二、总纲：OS 是中介的三角关系

文本输入的本质是一个**经操作系统托管的三角关系**：

<img src="/assets/img/ime-triangle.svg" alt="OS 托管的三角关系：输入组件、OS 输入法框架与输入法三方经会话交互" style="display:block; width:249px; max-width:100%; height:auto; margin:1em auto;">

- **输入法（Provider）**的角色是"按键 → 候选 → 文本"的转换器；
- **输入组件（Client）**的角色是"声明我是一个文本输入目标"：报告光标位置、提供周围文本、消费合成文本；
- 两者**从不直接通信**。按键由 OS 路由给输入法，输入法的输出由 OS 投递给当前焦点的组件。输入法甚至不需要知道对面是系统文本框还是你自己写的组件。

围绕这个三角，每个平台都定义了一对**对称接口**：provider 侧（输入法要实现）和 client 侧（组件要实现）。下文分两侧展开。

## 三、Provider 侧：一个输入法的 anatomy

自定义输入法不是读键盘硬件的程序，而是一个被系统托管的组件。它向上游（应用）输出三类东西：

1. **合成文本（composition / preedit）**：未上屏的草稿，如拼音 `nihao`，通常带下划线；
2. **候选词列表（candidates）**：配套的候选 UI；
3. **提交文本（commit）**：用户选定后定稿上屏的"你好"。

它从系统接收的核心事件是：**按键按下/抬起、会话激活/失活、光标位置变化**；允许查询的上下文是 surrounding text（光标前后已提交文本）与选区。

各平台的 provider 接口：

| 平台 | 接口 | 按键入口 |
|---|---|---|
| macOS | InputMethodKit：`IMKServer` + `IMKInputController` | `handle(_:client:)` 接收 NSEvent |
| Windows | Text Services Framework：COM 接口 `ITfKeyEventSink` 等 | `OnTestKeyDown/OnTestKeyUp` 先决定"要不要这个键"，再 `OnKeyDown/OnKeyUp` 消费 |
| Linux | IBus engine（DBus `ProcessKeyEvent`）或 Fcitx5 addon | `ProcessKeyEvent(keyval, keycode, state)` |
| Wayland 原生 | compositor 的 `input-method-v2` 协议 | compositor 转发 |

工程上的主流架构是**引擎与外壳分离**：输入法引擎（分词、词典、语言模型）写成跨平台纯库，外壳只做"收事件 → 调引擎 → 回调系统 API"。librime（中州韵引擎）就是范例——它的 macOS 外壳叫 Squirrel（鼠须管），Windows 外壳叫 Weasel（小狼毫）。

## 四、Client 侧：GUI 组件要实现的接口

client 侧回答的问题是："我如何让系统把我当一个合法的文本编辑框？"

| 平台 | 现成实现 | 自写组件要实现 |
|---|---|---|
| macOS | `NSTextField` / `NSTextView` | `NSTextInputClient` 协议：`insertText`、`setMarkedText`、`hasMarkedText`、`attributedSubstring`、`firstRect(forCharacterRange:)`（候选窗定位靠它）等 |
| Windows | Win32 Edit / WPF | TSF 的 `ITfContextOwner`；或走老路 IMM32，处理 `WM_IME_STARTCOMPOSITION` / `WM_IME_COMPOSITION` / `WM_IME_CHAR` |
| Linux Wayland | GTK / Qt 输入框 | `zwp_text_input_v3` 协议：enable 后收 `preedit_string` / `commit_string` |
| Web | `<input>` / `<textarea>` / `contenteditable` | **浏览器已实现，无需任何接口**——这正是下节的主角 |

这套接口的存在意义，就是允许任何自定义视图（自绘文本框、代码编辑器、终端）成为一等公民的 IME 目标。组件要承担的实际职责是：渲染带下划线的 preedit 文本、维护选区模型、把"该自己处理的按键"（光标移动、快捷键）和"该交给 IME 的按键"分清楚——后者是 bug 最多的地方。

## 五、解剖 Chromium：一条 IME 消息的完整旅程

现在进入浏览器。如果第三节和第四节是"两边各自的外交部"，那么 Chromium 就是一个同时运转着**两套外交部**的双进程国家。`textarea` 接收 IME 的真相是：

<img src="/assets/img/chromium-ime-pipeline.svg" alt="Chromium 中一条 IME 消息从系统 IME 经 Browser 进程、IPC、Renderer、Blink 到 DOM 事件的五层旅程" style="display:block; width:702px; max-width:100%; height:auto; margin:1em auto;">

**① Browser 进程承接平台接口。** 这是系统 IME 真正的对话方。以 Aura（Linux/ChromeOS 路径）为例，`RenderWidgetHostViewAura` 明确实现了 `ui::TextInputClient`：`SetCompositionText()` 把合成文本转成 IPC 发给 renderer，`ConfirmCompositionText()` 对应 `ImeFinishComposingText`。macOS 上则是 `RenderWidgetHostViewMac` 实现 `NSTextInputClient`。这些类只是翻译官，把平台回调打包发 IPC——源码里甚至留着 WebKit 时代"composition 不能带 selection range"的 TODO 注释。

**② IPC 边界。** Browser 和 Renderer 是两个进程，IME 语义被编码为三条消息：`ImeSetComposition`（更新合成文本+下划线）、`ImeCommitText`（定稿上屏）、`ImeFinishComposingText`（结束合成）。另有一条反向通道：renderer 上报 `WebTextInputType`（None/Text/Password/Search/Email/URL/Number/ContentEditable…）和光标矩形——browser 据此决定**是否激活 IME、候选窗摆在哪**。所以 `type="checkbox"` 永远不会唤起输入法，不是"IME 认出了它"，而是它上报的 type 是 None。

**③④ Blink 的核心是 `InputMethodController`。** Renderer 收到 IPC 后交给 `WebViewImpl`，最终汇入 `blink::InputMethodController`。它是 Blink 内部的 IME 会话状态机：管理 composition 范围和 underlines、决定何时触发 `compositionstart`、把 composition 同步进可编辑元素、commit 时走 Editing 命令管线。注意它挂在 `LocalFrame` 上而**不挂在任何元素上**——元素焦点换来换去，会话状态由它统一管理。

**⑤ DOM 元素收到的是标准 DOM 事件。** `<textarea>`（Blink 里它是 `TextControlElement` 子类）作为焦点元素收到 `CompositionEvent` 三件套（compositionstart/update/end）和 `beforeinput`/`input`（带 `isComposing` 标志）。Web 开发者面对的这套 API，就是内部五层链路的对外投影。

所以严格回答"textarea 实现了哪些接口"：**平台 IME 接口——零个**。它的输入资格来自三点：可编辑（有值和选区模型）、可聚焦、renderer 为它上报了非 None 的 TextInputType。IME 回调从不按元素类型分发，只按**焦点**分发。

## 六、为什么 Canvas 上必须藏一个 textarea

现在一切铺垫都可以收拢到这个问题上。

**Canvas 元素永远不可能成为 IME 目标。** `<canvas>` 不是可编辑元素，没有文本模型、没有选区、不能聚焦成输入目标，OS 不会向它路由按键，更不会向它投递 composition。你在 canvas 上画的任何"光标"和"文本"对系统来说都只是像素。

于是在 Canvas 上做富文本编辑器（各类 Web 代码编辑器、绘图软件的文字工具，早期 VS Code 的 Web 版、xterm.js 等都走过这条路）时，标准的 hack 是：**在 Canvas 之上放一个不可见的 `<textarea>`（或 contenteditable div），让它做系统的"代理输入目标"，自己只做渲染。**

<img src="/assets/img/canvas-textarea.svg" alt="Canvas 编辑器分层：透明 textarea 作为代理输入目标叠加上方，值与选区同步回文档模型并重绘 canvas" style="display:block; width:551px; max-width:100%; height:auto; margin:1em auto;">

为什么必须这样做，而不是自己监听 `keydown` 拼字符串：

1. **IME 只认焦点，不认监听。** 系统 IME 的激活、候选窗定位、合成文本投递，全部经由焦点链完成。只有真实可聚焦、可编辑的元素才能让 OS 认为"这里有文本输入会话"。
2. **`keydown` 拿不到合成过程。** 中文输入时 `keydown` 只给你原始的 ASCII 字母，`compositionstart/update/end` 才携带 preedit 与候选生命周期。绕过 textarea 意味着 CJK 用户彻底无法输入。
3. **移动端虚拟键盘需要真实的输入目标。** 手机上唤起软键盘，靠的也是 OS 检测到焦点元素是可编辑控件；一个纯 canvas 应用很多浏览器里连键盘都弹不出。
4. **浏览器已经把平台接口都写好了。** textarea 免费获得了第五节那五层链路的全部基础设施，你只需要消费 DOM 事件。

具体工程细节（都是血泪经验）：

- textarea 必须**可聚焦**：用 `opacity: 0` 或移出视口（如 `position:fixed; left:-9999px`），**不能** `display:none` 或 `visibility:hidden`，否则无法聚焦；
- 必须处理 `compositionstart` 期间暂停快捷键响应，等 `compositionend` 后再处理（否则"回车选候选"会被误当成"提交表单"）；
- 用 `beforeinput` 的 `inputType` 和 `isComposing` 区分"用户直接输入"与"IME 提交"；
- iOS Safari 上 textarea 字号小于 16px 会触发聚焦缩放，需要特别处理；
- 要把 textarea 的 selectionStart/End 与你在 canvas 上画的选区保持双向同步（方向键、Home/End、鼠标点选都要走 textarea 的选区语义）；
- 多光标、块选区等编辑器高级特性无法通过 textarea 表达——这正是 VS Code 后来从"隐藏 textarea"走向自研输入处理层的原因，但那已经是另一个量级的工程。

## 七、EditContext：隐藏 textarea 的标准化替代

第六节描述的"隐藏 textarea"手法如此普遍，以至于 W3C 最终为这个痛点立了正式标准——**EditContext API**（[W3C 规范](https://www.w3.org/TR/edit-context/)，Web Editing WG，最初由微软在 WICG 孵化）。它的背景章节几乎逐字复述了上节的问题：文本画在 canvas 上时，user agent 无法生成驱动文本输入所需的 Text Edit Context，作者只好诉诸屏幕外可编辑元素，代价是无障碍受损、输入体验变差、同步逻辑复杂。这正是 Monaco、Google Docs、Word Online 当年的真实处境，微软提案时明确说明在与这几家合作。

它的核心思路一句话：**把 Text Edit Context 从 DOM 可编辑元素里抽出来，变成一个显式对象**——任何元素（包括 `<canvas>`）都可以关联一个 `EditContext`，浏览器把它当作与系统输入法对话的文本状态载体。于是浏览器与 canvas 编辑器之间形成一条双向协议：

- **作者 → 浏览器**：`updateText()`、`updateSelection()`、`updateControlBounds()`、`updateSelectionBounds()`、`updateCharacterBounds()`——把画在 canvas 上的文本、选区和**每个字符的屏幕矩形**喂回去，浏览器就能正确定位 IME 候选窗、唤起虚拟键盘；
- **浏览器 → 作者**：`textupdate`（文本/选区/合成区间变化）、`textformatupdate`（合成文本的下划线样式，用于渲染 preedit）、`characterboundsupdate`（要求作者回传字符矩形）；规范还明确规定：document 存在激活的 EditContext 时，composition 事件**改派发到 EditContext 而非编辑宿主**。

边界同样清楚：EditContext 只处理纯文本的 `beforeinput` inputType（`insertText`、`deleteContent*` 等），拼写检查、剪贴板、undo 不在其状态模型内；canvas 场景下**光标导航和选区必须由作者实现**——浏览器不知道你的文本是怎么排版的。

现状（截至 2026 年 9 月）：Chrome 和 Edge 从 121 版（2024 年 1 月）起已完整实现；Firefox 官方立场为 positive，Safari 尚无实现。所以短期内"隐藏 textarea"仍要写，但 Chrome 系已有一条标准逃生通道。

回看整条故事线，EditContext 的位置很清晰：第六节说"隐藏 textarea 是站在既有信任链上最便宜的做法"，而 EditContext 就是 W3C 承认了这个痛点之后，把这条信任链正式产品化的答案——**代理输入目标从 hack 变成了 API**。

## 八、总结：把整条链收进一句话

一次中文输入是**五个世界经由四层接口的接力**：键盘到 OS 是事件，OS 到输入法路由按键，输入法经 OS 框架向"焦点可编辑组件"投递合成文本，组件（在浏览器里是 Chromium 的 RenderWidgetHostView → InputMethodController 五层翻译链）把平台语义转成 DOM 事件，最终由 `textarea` 这样的可编辑元素作为事件终点。

`textarea` 的价值不在于它"实现了什么"，而在于**它已经被整个世界的每一层认定为合法对话方**。在 Canvas 上处理输入之所以需要它，是因为你想重新发明的不只是渲染，而是这条跨五个世界的信任链——而隐藏 textarea 是站在既有信任链上最便宜的做法。EditContext API 的出现（第七节）则标志着这条信任链终于被标准化：未来在 Canvas 上做编辑器，"代理输入目标"不再是一个 hack，而是一个有名字的 Web API。
