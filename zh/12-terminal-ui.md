# 第 12 章：终端 UI 与渲染引擎

## 12.1 概述与定位

Claude Code 是一个运行在终端里的交互式 AI 编码助手，其 UI 层需要在纯文本终端环境中实现接近 GUI 的交互体验：流式消息渲染、Vim 键位编辑、鼠标文本选择、搜索高亮、多层弹窗叠加。这套需求远超传统 CLI 工具的能力边界。

Claude Code 的终端 UI 层建立在三个核心技术决策之上：

1. **自定义 Ink 渲染器**（`src/ink/`）：对 React + Ink 框架进行深度改造，引入双缓冲屏幕模型、ANSI diff 引擎、硬件滚动加速，以及 IME 感知的物理光标定位。
2. **声明式组件树**（`src/screens/REPL.tsx`、`src/components/`）：用 React 组件描述 UI 结构，由渲染引擎负责将虚拟 DOM 翻译为终端字符矩阵。
3. **完整 Vim 模式**（`src/vim/`）：独立的状态机实现，覆盖 NORMAL/INSERT 双模式、operators/motions/textObjects 全集，以及 dot-repeat 与寄存器。

整个渲染管线的节拍由 `FRAME_INTERVAL_MS = 16`（`src/ink/constants.ts:1`）控制，理论帧率 62.5fps，与显示器刷新率对齐。

---

## 12.2 理论基础

### 终端渲染的底层原理

终端渲染本质上是向文件描述符写入字节流，终端模拟器负责解析并更新屏幕。关键机制包括：

- **ANSI escape codes**：`\x1b[` 开头的控制序列，控制光标位置（CSI CUP）、颜色（SGR）、滚动区域（DECSTBM）、鼠标追踪（DEC private modes）等。Claude Code 的 `src/ink/termio/` 目录将这些序列封装为命名常量（`CURSOR_HOME`、`ENTER_ALT_SCREEN`、`ENABLE_MOUSE_TRACKING` 等）。
- **termios / raw mode**：进入 raw mode 后，每次按键立即送达程序，不经过行缓冲或回显。Claude Code 在启动时通过 `setRawMode(true)` 接管 stdin，在 `Ink.detachForShutdown()` 退出时恢复（`ink.tsx:942-950`）。
- **Alt Screen（?1049h/?1049l）**：备用屏幕缓冲区，进入后不污染主屏幕滚动历史，退出后恢复原状。全屏模式（`isFullscreenEnvEnabled()`）下 REPL 包裹在 `<AlternateScreen>` 组件中。

### 双缓冲渲染

Claude Code 使用前后两个帧缓冲区（`frontFrame` / `backFrame`）实现双缓冲（`ink.tsx:196-197`）：

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

每帧渲染时：
1. Renderer 将 React 虚拟 DOM 写入 `backFrame`（新帧）
2. LogUpdate diff 引擎对比 `frontFrame`（旧帧）与 `backFrame`，生成最小 patch 序列
3. patch 序列写入终端，`frontFrame` ↔ `backFrame` 互换（`ink.tsx:594`）

这样终端只接收真正变化的字符，而非每帧全量重绘。

### React 在非 DOM 环境中的应用（Custom Renderer）

React 的渲染器接口（`react-reconciler`）允许任意宿主环境。Claude Code 在 `src/ink/reconciler.ts`（512 行）中实现了针对终端的 Custom Renderer：

- 宿主元素类型为自定义的 `dom.DOMElement`（`src/ink/dom.ts`）
- 布局引擎使用 Yoga（Facebook 的 Flexbox 实现），在 `onComputeLayout` 回调中驱动（`ink.tsx:239-258`）
- React Concurrent Root 模式（`ConcurrentRoot`）允许渲染可中断，避免长时间阻塞主线程

---

## 12.3 Ink 自定义渲染器

### Claude Code 对 Ink 的自定义改造

`src/ink/ink.tsx`（1,722 行）是整个 UI 层的核心控制器，定义了 `Ink` 类。这不是直接使用原版 Ink 包，而是深度改造后的自定义版本，核心增强点包括：

**双缓冲 + 前后帧管理**：原版 Ink 直接输出字符串，自定义版本引入帧对象（`Frame`）和 `Screen` 矩阵。

**性能监控**：每帧记录 renderer、diff、optimize、write 各阶段耗时，以及 Yoga 布局的 visited/measured/cacheHits 指标，通过 `onFrame` 回调暴露（`ink.tsx:772-788`）：

```typescript
// ink.tsx:772-788
this.options.onFrame?.({
  durationMs: performance.now() - renderStart,
  phases: {
    renderer: rendererMs,
    diff: diffMs,
    optimize: optimizeMs,
    write: writeMs,
    patches: diff.length,
    yoga: yogaMs,
    commit: commitMs,
    yogaVisited: yc.visited,
    yogaMeasured: yc.measured,
    yogaCacheHits: yc.cacheHits,
    yogaLive: yc.live,
  },
  flickers,
});
```

**外部 TUI 交接**：提供 `enterAlternateScreen()` / `exitAlternateScreen()` 方法，暂停 Ink 控制权、关闭 kitty keyboard 协议，让外部编辑器（vim、nano）正常运行，返回后完整恢复（`ink.tsx:357-419`）。

**SIGCONT 自愈**：`handleResume` 处理进程从暂停恢复（fg 命令），主屏幕重置缓冲区，alt 屏幕重新进入并清屏（`ink.tsx:280-301`）。

### 双缓冲 + diff 更新机制

`src/ink/log-update.ts` 中的 `LogUpdate.render()` 是 diff 引擎的核心。它接收 `prev: Frame` 和 `next: Frame`，返回一组 `Diff` patch 操作：

```
Diff patch 类型：
- { type: 'stdout', content: string }    — 直接写入终端
- { type: 'cursorMove', x, y }           — 相对光标移动
- { type: 'cursorTo', x, y }             — 绝对光标定位
- { type: 'styleStr', str }              — SGR 样式切换
- { type: 'hyperlink', uri }             — OSC 8 超链接
- { type: 'cursorHide' | 'cursorShow' }  — 光标可见性
- { type: 'clear', count }               — 擦除字符
- { type: 'clearTerminal', reason }      — 全屏重绘（触发 flicker）
```

diff 引擎在 `src/ink/optimizer.ts` 中经过单次 pass 的优化（`optimize(diff)`），合并相邻 `cursorMove`、折叠连续 `cursorTo`、拼接相邻 `styleStr`、取消 cursor hide/show 对，减少实际写入量。

### Int32Array 屏幕缓冲

`src/ink/screen.ts` 定义了 `Screen` 类型，使用 `Int32Array` 而非对象数组存储屏幕单元格，消除 GC 压力（`screen.ts:356-370`）：

```typescript
// screen.ts:356-370（注释）
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen` 分配一个 `ArrayBuffer`，同时挂载两个视图（`Int32Array` 和 `BigInt64Array`），后者用于 `resetScreen` 的批量清零（`screen.ts:472-490`）：

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes per cell
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

字符和样式通过 **共享 Pool 对象**（`CharPool`、`StylePool`、`HyperlinkPool`）进行 intern，跨 Screen 共享 ID，使 `blitRegion` 可以直接复制整数 ID 而无需重新 intern，`diffEach` 可以用整数比较代替字符串比较（`screen.ts:17-31`）。

`StylePool.intern()` 的 bit-0 编码可见性（`screen.ts:148-161`）：奇数 ID 表示该样式对空格可见（背景色、反色、下划线），使渲染器可用单次位掩码跳过不可见空格，避免不必要的写入。

### 硬件滚动（DECSTBM）

`log-update.ts` 中实现了 DECSTBM 硬件滚动优化（`log-update.ts:149-185`）：

当 alt-screen 模式下 `ScrollBox` 的 `scrollTop` 发生变化时，不重写整个滚动区域，而是发送 `CSI top;bot r`（设置滚动区域）+ `CSI n S`（向上滚动 n 行），终端硬件加速完成内容移位，只需补充新滚入的行。同时对 `prev.screen` 执行 `shiftRows()` 模拟滚动，使后续 diff 循环自然找到差异行而非全量重写。

此优化仅在 `decstbmSafe=true` 时启用——当外部终端不支持 DEC 2026（BSU/ESU 原子操作）时，tmux 等终端会渲染中间状态（滚动后尚未补行），产生可见跳动，此时退回全量 diff（`log-update.ts:158-167`）。

### 16,384 字符缓存

`src/ink/output.ts` 的 `Output` 类（渲染器内部的字符测量与输出引擎）维护一个 `charCache`，缓存字符到测量宽度的映射。当缓存超过 16,384 条时直接清空重建（`output.ts:204`）：

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

类似地，`src/ink/line-width-cache.ts` 维护行宽度缓存（最大 4096 条），用于流式输出中大量未变化行的 `stringWidth` 调用复用（`line-width-cache.ts`）：

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### 性能优化策略

| 策略 | 位置 | 效果 |
|------|------|------|
| 渲染调度 microtask 延迟 | `ink.tsx:212-216` | scheduleRender 使用 `queueMicrotask`，确保 useLayoutEffect 在渲染前完成，避免光标位置单帧滞后 |
| throttle 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`，稳定帧率 |
| Drain timer 四倍帧率 | `ink.tsx:757-758` | scroll 连续 drain 使用 `FRAME_INTERVAL_MS >> 2`（约 4ms），最大化滚动流畅度 |
| Pool 周期重置 | `ink.tsx:600-603` | 每 5 分钟重置 char/hyperlink pool，防止长会话内存增长 |
| 伤害区域（damage rectangle） | `screen.ts:382` | 每帧只 diff 实际写入的矩形区域，而非全屏 |
| Alt-screen 光标锚定 | `ink.tsx:578-591` | 每帧 diff 前用 `ALT_SCREEN_ANCHOR_CURSOR` 将虚拟光标重置到 (0,0)，防止 tmux 等终端的光标漂移 |

---

## 12.4 REPL 主界面

### REPL.tsx 的组件结构

`src/screens/REPL.tsx`（5,005 行）是 Claude Code 的主界面组件，也是整个项目最大的单文件。它的顶层 import 超过 150 行，覆盖会话管理、权限系统、MCP 连接、Swarm 多智能体、voice 集成等几乎所有功能模块。

REPL 的渲染树结构（简化）：

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen（全屏模式）
    │   └── Box（主布局）
    │       ├── AnimatedTerminalTitle（终端 tab 标题）
    │       ├── FullscreenLayout
    │       │   ├── Messages（消息列表）
    │       │   ├── VirtualMessageList（虚拟滚动）
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── 各类 Callout / Survey / Dialog
    │       └── PromptInput（输入区）
    └── DevBar（ant-only 调试栏）
```

全屏模式通过条件分支决定是否包裹 `<AlternateScreen>`（`REPL.tsx:4998-5001`）：

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### 消息渲染流程

消息状态由 React `useState<MessageType[]>` 持有，通过 `handleMessageFromStream` 处理流式 SSE 事件追加/更新消息。消息列表渲染到 `<Messages>` 或 `<VirtualMessageList>` 组件，后者通过虚拟滚动只渲染可见区域，避免长对话中的大量 DOM 节点。

REPL 内有多处避免不必要重渲染的优化：`AnimatedTerminalTitle` 被单独提取为叶节点，防止 960ms 动画 tick 重渲染整个 REPL 树（`REPL.tsx` 中注释说明：`Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`）。

`useDeferredValue(searchQuery)` 用于延迟搜索计算，`React.memo` 缓存在 `PromptInput` 子树，防止输入字符时的频繁渲染向上传播。

### 输入处理管线

用户输入从 stdin raw mode 进入 Ink 的 `parse-keypress.ts`，解析为 `ParsedKey` 对象，经由 FocusManager 分发到当前焦点节点的 `useInput` 回调。`GlobalKeybindingHandlers` 和 `CommandKeybindingHandlers` 提供全局快捷键注册机制，优先级高于组件级 `useInput`。

输入最终到达 `PromptInput` 组件，根据当前模式（Vim/Standard）分路到 `VimTextInput` 或 `TextInput`。

---

## 12.5 Vim 模式

### Vim 模式的实现架构

`src/vim/` 目录（5 个文件，~1,513 行）实现了完整的 Vim 键位层。架构设计极为简洁：**类型即文档**，所有状态机转换以 TypeScript 类型编码，TypeScript 的穷举检查保证状态处理完整性。

文件分工：
- `types.ts`：状态机类型定义 + 常量集合
- `transitions.ts`：转换函数（主入口 `transition()`）
- `motions.ts`：光标移动纯函数（`resolveMotion()`）
- `operators.ts`：操作执行函数（delete/change/yank 等）
- `textObjects.ts`：文本对象边界计算

### 支持的操作（motions, operators, textObjects）

**Operators（操作符）**：`d`（delete）、`c`（change）、`y`（yank）——定义在 `types.ts:OPERATORS`。

**Simple Motions**（`types.ts:SIMPLE_MOTIONS`）：
- 基础移动：`h`/`l`/`j`/`k`
- 词移动：`w`/`b`/`e`/`W`/`B`/`E`
- 行位置：`0`/`^`/`$`

**Find Motions**：`f`/`F`/`t`/`T`（字符查找），`;`/`,` 重复/反向。

**Text Objects**（`types.ts:TEXT_OBJ_TYPES`）：`w`/`W`（词）、`"`/`'`/`` ` ``（引号）、`(`/`)`/`b`（括号）、`[`/`]`（方括号）、`{`/`}`/`B`（花括号）、`<`/`>` （尖括号）。

**行级操作**：`dd`/`cc`/`yy`、`D`/`C`/`Y`、`o`/`O`（开行）、`J`（join）、`>>`/`<<`（缩进）。

**其他命令**：`r`（replace char）、`~`（toggle case）、`x`（delete char）、`p`/`P`（paste）、`u`（undo）、`G`/`gg`（跳行）、`.`（dot-repeat）。

特殊处理：`cw`/`cW` 改变到词尾而非下一词首（vim 经典行为，`operators.ts:91-102`）；Image ref chip 被识别为原子单位，词移动不会停在 chip 中间（`operators.ts:333-337`）。

### 状态机与模式切换

`VimState` 是顶层类型（`types.ts:52-55`）：

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT 模式记录已输入文本用于 dot-repeat；NORMAL 模式持有 `CommandState` 子状态机。

`CommandState` 有 11 种状态（`types.ts:62-75`），覆盖 idle、count accumulation、operator pending、operatorCount、operatorFind、operatorTextObj、find、g-prefix、operatorG、replace、indent。状态图在 `types.ts` 开头以 ASCII art 形式记录（`types.ts:1-26`）：

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

`transition()` 函数（`transitions.ts:55-73`）以当前 `CommandState` 类型分派到各专用处理函数，每个函数返回 `{ next?: CommandState; execute?: () => void }`。`execute` 存在时立即运行，`next` 存在时推进状态，两者都不存在则回到 idle。

`PersistentState`（`types.ts:80-87`）跨命令保持：`lastChange`（dot-repeat 录像）、`lastFind`（`;`/`,` 重复）、`register`（剪贴板）、`registerIsLinewise`（行级 paste 判断）。

计数乘法在 `fromOperatorCount` 中实现（`transitions.ts`）：`effectiveCount = state.count * motionCount`，支持 `3d2w` = 删除 6 个词。计数上限为 `MAX_VIM_COUNT = 10000`（`types.ts:96`），防止恶意输入。

---

## 12.6 核心组件分析

### PromptInput 输入组件

`src/components/PromptInput/PromptInput.tsx`（2,338 行）是 Claude Code 功能密度最高的组件之一，承载了几乎所有输入相关的交互逻辑。

**Props 规模**：`Props` 类型定义跨越约 60 行，包含输入值、模式、Vim 状态、stash 栈、提交回调、权限上下文、MCP 客户端、IDE 选择、语音集成等。

**外部输入注入**：`insertTextRef` 暴露 `{ insert, setInputWithCursor, cursorOffset }` 接口，供语音识别（STT）在光标位置插入文本而非替换整个输入（`PromptInput.tsx:180-200`）。

**光标管理**：独立追踪 `cursorOffset` state，区分内部变更（通过 `trackAndSetInput` 包装 `onInputChange`）和外部注入（语音输入），外部变更将光标移到末尾（`PromptInput.tsx:164-176`）。

**模式路由**：根据 `isVimModeEnabled()` 渲染 `<VimTextInput>` 或 `<TextInput>`，两者共享 `BaseTextInputProps` 接口。

**底部布局常量**（`PromptInput.tsx`）：
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
底部输入槽最大高度 50%，保留 5 行给 footer、边框和状态提示。

### 设置界面

`src/components/Settings/Config.tsx`（1,821 行）实现 `/config` 命令的设置界面。

**搜索驱动导航**：Config 默认进入 search 模式（`isSearchMode = true`），用户可直接输入过滤设置项，无需方向键滚动整个列表。`maxVisible = Math.max(5, paneCap - 10)` 根据终端高度动态计算可见条目数。

**设置分类**：`settingsItems: Setting[]` 数组统一定义所有配置项，每项为 `boolean`（开关）、`enum`（枚举选择）或 `managedEnum`（自定义组件管理）之一。

**变更追踪与还原**：`changes` state 记录所有修改；`isDirty` ref 用于 Escape 键判断是否需要写盘还原，避免打开后关闭触发无意义的磁盘写入（`Config.tsx`）。

**子菜单系统**：`showSubmenu` state 控制 Theme、Model、TeammateModel、ExternalIncludes、OutputStyle、Language、EnableAutoUpdates 等子菜单的展开，通过 `setTabsHidden` 向父组件 Settings 通信以隐藏 Tab 头。

**多源配置架构**：设置来源分为 `localSettings`（项目级）和 `userSettings`（用户级），通过 `getSettingsForSource`/`updateSettingsForSource` 读写，Escape 还原时使用初始快照逐源恢复，而非读取合并后的全局视图（`Config.tsx`，注释说明这是为了支持 `undefined`-删除-key 的语义）。

### 日志选择器

`src/components/LogSelector.tsx`（1,574 行）实现会话历史选择界面（`/resume` 命令）。

**树形展示**：通过 `<TreeSelect<LogTreeNode>>` 将分叉会话（fork）以父子树形展示，折叠/展开由 `expandedGroupSessionIds` 管理。

**三种视图模式**：`viewMode` 可为 `"list"`（默认列表）、`"search"`（搜索）、`"preview"`（会话预览）。

**Fuse.js 模糊搜索**：使用 `FUSE_THRESHOLD = 0.3` 的模糊匹配，结合 `DATE_TIE_THRESHOLD_MS = 60 * 1000`（1分钟内的结果按相关性排序，否则按时间排序）。

**搜索结果片段预览**：`extractSnippet()` 在匹配位置前后各取 `SNIPPET_CONTEXT_CHARS = 50` 字符，用 `chalk.dim` 渲染上下文、高亮色渲染匹配词（`LogSelector.tsx`）。

**深度搜索常量**：
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
（当前 `isDeepSearchEnabled = false`，为待启用特性）

**React Compiler 优化**：文件顶部 `import { c as _c } from "react/compiler-runtime"` 表明已通过 React Compiler 编译，`_c(247)` 分配 247 个槽位的 memo 缓存，大量条件记忆避免子树重渲染。

---

## 12.7 设计决策分析

### 为什么自定义 Ink 渲染器而非直接使用

原版 Ink 将 React 树序列化为字符串后用 `log-update` 增量写入，设计针对简单 CLI 工具。Claude Code 的需求超出这一模型的极限：

1. **鼠标选择**：需要屏幕级单元格坐标映射（hit-test），字符串模型无法支持。
2. **搜索高亮**：需要在已渲染帧上叠加反色单元格，必须操作单元格矩阵。
3. **DECSTBM 硬件滚动**：需要控制 prev/next 两帧在滚动区域的一致性，字符串 diff 无法模拟终端硬件滚动的效果。
4. **IME / a11y 光标定位**：需要精确知道输入光标的物理屏幕坐标，将物理终端光标停在正确位置（CJK 输入法的 preedit 文本在物理光标处渲染）。
5. **Alt-screen 原子性（BSU/ESU）**：需要将 erase + paint 包裹在 `\x1b[?2026h`/`\x1b[?2026l` 中，防止 tmux 等外层终端渲染中间状态。

### 双缓冲的性能考量

终端输出是 I/O 密集操作，`write()` 系统调用有固定开销。双缓冲 diff 策略将每帧写入量从 O(屏幕尺寸) 降至 O(变化区域)。实测注释显示：drain-only 帧（仅 DECSTBM scroll，无 React commit）约产生 ~10 个 patch、~200 字节输出（`ink.tsx:756`），而全量重绘需要数千字节。

`prevFrameContaminated` 标志是正确性安全网：当选择高亮叠加、`resetFramesForAltScreen()`、`forceRedraw()` 等操作修改了 frontFrame 的内容，下一帧必须做全量 diff 而非依赖增量 blit 优化（`ink.tsx:743`）。

### Vim 模式的设计动机

Claude Code 的核心用户群是重度 Vim 用户——对他们而言，在提示框中使用 Vim 键位不是锦上添花而是基本需求。纯状态机实现（无依赖、无副作用）使得 Vim 逻辑完全可单元测试，与终端渲染层解耦。

`types.ts` 头部的注释揭示了设计哲学：**类型即文档**。`CommandState` 的 union type 比任何注释都更精确地描述状态机——TypeScript 的 switch 穷举检查保证每个状态都被处理，新状态的遗漏会在编译时报错（`types.ts:1`）。

---

## 12.8 可迁移模式

以下设计模式可在其他终端应用或 UI 框架中借鉴：

**1. 共享 Pool + Integer ID 屏幕缓冲**
用 `Int32Array` 存储字符/样式的整数 ID，而非字符串或对象。跨帧共享 Pool 使 diff 降为整数比较，彻底消除 GC 压力。适用于任何需要高频 diff 的矩形矩阵场景。

**2. 类型即状态机**
用 TypeScript union type 定义状态机，每种状态携带恰好需要的数据，状态转换函数用 exhaustive switch。`transition()` + `TransitionResult` 的 `{next?, execute?}` 双键返回值极为简洁，可以直接移植到其他键位处理场景。

**3. 伤害区域（Damage Rectangle）驱动的增量 diff**
渲染时记录实际写入的矩形区域（`damage`），diff 只扫描此区域。适用于任何帧缓冲系统——不仅终端，游戏 UI、嵌入式显示也可使用此模式。

**4. 双缓冲 + prevFrameContaminated 安全网**
当某些操作不走正常渲染路径（直接修改前帧）时，用一个 bool 标志强制下帧全量 diff，而非破坏双缓冲不变量。比每次都全量重绘更高效，比允许脏数据更正确。

**5. 渲染 microtask 延迟**
`scheduleRender = throttle(() => queueMicrotask(onRender), 16)` 的模式确保 React layout effects 在渲染前完整提交，避免光标位置单帧滞后。在其他 React + 自定义渲染器场景中同样适用。

**6. 行宽缓存（streaming 场景专项优化）**
流式输出中大量未变化行的 `stringWidth` 计算是热点。`line-width-cache.ts` 的简单 Map + 满则清空策略以 20 行代码解决了 `~50x` 的计算开销（注释原文），可直接移植到任何流式文本渲染场景。

---

## 12.9 源码索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `src/ink/ink.tsx` | 1,722 | Ink 类：帧管理、渲染调度、SIGCONT、alt-screen 交接、鼠标选择、搜索高亮 |
| `src/ink/screen.ts` | 1,300+ | Screen 类型、Int32Array 单元格布局、CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate：prev/next 帧 diff 引擎、DECSTBM 硬件滚动 |
| `src/ink/optimizer.ts` | ~100 | Diff 单 pass 优化器 |
| `src/ink/reconciler.ts` | 512 | React Custom Renderer 宿主实现 |
| `src/ink/render-node-to-output.ts` | 1,462 | DOM 节点到 Screen 的渲染 |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | 行宽度 LRU 缓存（4096 条） |
| `src/ink/output.ts` | ~300 | Output 渲染器，字符缓存（16,384 条） |
| `src/vim/types.ts` | ~150 | VimState、CommandState、PersistentState 类型定义及常量 |
| `src/vim/transitions.ts` | ~350 | `transition()` 主入口，11 个状态转换函数 |
| `src/vim/motions.ts` | ~80 | `resolveMotion()` 光标移动纯函数 |
| `src/vim/operators.ts` | ~500 | delete/change/yank/x/r/~/J/paste/indent/openLine 执行函数 |
| `src/vim/textObjects.ts` | ~220 | 词/引号/括号等文本对象边界查找 |
| `src/screens/REPL.tsx` | 5,005 | 主 REPL 界面，会话管理，消息流，全屏/普通模式切换 |
| `src/components/PromptInput/PromptInput.tsx` | 2,338 | 输入组件，Vim/Standard 模式路由，光标管理，STT 注入 |
| `src/components/Settings/Config.tsx` | 1,821 | 设置界面，搜索导航，多源配置读写，Escape 还原 |
| `src/components/LogSelector.tsx` | 1,574 | 会话历史选择，Fuse.js 搜索，树形 fork 展示 |

**关键常量速查**：
- `FRAME_INTERVAL_MS = 16`（`constants.ts:1`）
- `MAX_CACHE_SIZE = 4096`（`line-width-cache.ts`）
- `charCache` 上限 `16384`（`output.ts:204`）
- `MAX_VIM_COUNT = 10000`（`vim/types.ts:96`）
- 单元格布局：8 bytes/cell，`word0=charId`，`word1=styleId[31:17]|hyperlinkId[16:2]|width[1:0]`（`screen.ts:356`）
