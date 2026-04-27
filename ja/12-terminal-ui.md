# 第 12 章：ターミナル UI とレンダリングエンジン

## 12.1 概要と位置づけ

Claude Code はターミナル上で動作するインタラクティブな AI コーディングアシスタントであり、その UI 層は純粋なテキストターミナル環境において GUI に近いインタラクション体験を実現する必要がある：ストリーミングメッセージのレンダリング、Vim キーバインド編集、マウステキスト選択、検索ハイライト、多層ポップアップオーバーレイ。これらの要件は従来の CLI ツールの能力範囲をはるかに超えている。

Claude Code のターミナル UI 層は 3 つのコア技術的意思決定の上に構築されている：

1. **カスタム Ink レンダラー**（`src/ink/`）：React + Ink フレームワークを深度カスタマイズし、ダブルバッファスクリーンモデル、ANSI diff エンジン、ハードウェアスクロール加速、IME 対応の物理カーソル位置決めを導入。
2. **宣言型コンポーネントツリー**（`src/screens/REPL.tsx`、`src/components/`）：React コンポーネントで UI 構造を記述し、レンダリングエンジンが仮想 DOM をターミナル文字マトリックスに変換する役割を担う。
3. **完全な Vim モード**（`src/vim/`）：独立したステートマシン実装で、NORMAL/INSERT デュアルモード、operators/motions/textObjects の全セット、dot-repeat とレジスタをカバーする。

レンダリングパイプライン全体のテンポは `FRAME_INTERVAL_MS = 16`（`src/ink/constants.ts:1`）によって制御され、理論フレームレートは 62.5fps でディスプレイのリフレッシュレートに合わせている。

---

## 12.2 理論的基礎

### ターミナルレンダリングの基本原理

ターミナルレンダリングとは本質的にファイルディスクリプタにバイトストリームを書き込むことであり、ターミナルエミュレータがこれを解析して画面を更新する。主要なメカニズムには以下が含まれる：

- **ANSI エスケープコード**：`\x1b[` で始まる制御シーケンスで、カーソル位置（CSI CUP）、色（SGR）、スクロール領域（DECSTBM）、マウストラッキング（DEC プライベートモード）などを制御する。Claude Code の `src/ink/termio/` ディレクトリはこれらのシーケンスを名前付き定数（`CURSOR_HOME`、`ENTER_ALT_SCREEN`、`ENABLE_MOUSE_TRACKING` など）としてカプセル化している。
- **termios / raw mode**：raw mode に入ると、各キー入力はラインバッファリングやエコーを経ずに即座にプログラムに届く。Claude Code は起動時に `setRawMode(true)` で stdin を引き継ぎ、`Ink.detachForShutdown()` 終了時に復元する（`ink.tsx:942-950`）。
- **Alt Screen（?1049h/?1049l）**：代替スクリーンバッファで、入ると主画面のスクロール履歴を汚染せず、退出後に元の状態に戻る。全画面モード（`isFullscreenEnvEnabled()`）では REPL が `<AlternateScreen>` コンポーネント内に包まれる。

### ダブルバッファリングレンダリング

Claude Code は前後 2 つのフレームバッファ（`frontFrame` / `backFrame`）を使ってダブルバッファリングを実装している（`ink.tsx:196-197`）：

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

各フレームのレンダリング時：
1. Renderer が React 仮想 DOM を `backFrame`（新フレーム）に書き込む
2. LogUpdate diff エンジンが `frontFrame`（旧フレーム）と `backFrame` を比較し、最小パッチシーケンスを生成する
3. パッチシーケンスをターミナルに書き込み、`frontFrame` ↔ `backFrame` を入れ替える（`ink.tsx:594`）

これにより、ターミナルは毎フレーム全量再描画ではなく、実際に変化した文字のみを受け取る。

### 非 DOM 環境における React の応用（カスタムレンダラー）

React のレンダラーインターフェース（`react-reconciler`）は任意のホスト環境を許容する。Claude Code は `src/ink/reconciler.ts`（512 行）でターミナル向けのカスタムレンダラーを実装している：

- ホスト要素型はカスタムの `dom.DOMElement`（`src/ink/dom.ts`）
- レイアウトエンジンは Yoga（Facebook の Flexbox 実装）を使用し、`onComputeLayout` コールバックで駆動される（`ink.tsx:239-258`）
- React Concurrent Root モード（`ConcurrentRoot`）によりレンダリングを中断可能にし、メインスレッドの長時間ブロッキングを回避する

---

## 12.3 Ink カスタムレンダラー

### Claude Code による Ink のカスタマイズ

`src/ink/ink.tsx`（1,722 行）は UI 層全体のコアコントローラーであり、`Ink` クラスを定義している。これはオリジナルの Ink パッケージをそのまま使用したものではなく、深度カスタマイズされたバージョンであり、コアの拡張点は以下の通り：

**ダブルバッファ + 前後フレーム管理**：オリジナルの Ink は文字列を直接出力するが、カスタム版ではフレームオブジェクト（`Frame`）と `Screen` マトリックスを導入している。

**パフォーマンス監視**：各フレームで renderer、diff、optimize、write の各フェーズの所要時間、および Yoga レイアウトの visited/measured/cacheHits 指標を記録し、`onFrame` コールバックで公開する（`ink.tsx:772-788`）：

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

**外部 TUI への引き渡し**：`enterAlternateScreen()` / `exitAlternateScreen()` メソッドを提供し、Ink の制御権を一時停止して kitty keyboard プロトコルを閉じ、外部エディター（vim、nano）が正常動作できるようにし、戻り時に完全に復元する（`ink.tsx:357-419`）。

**SIGCONT 自己修復**：`handleResume` がプロセスの一時停止からの復帰（fg コマンド）を処理し、主画面でバッファをリセットし、alt 画面で再入して画面をクリアする（`ink.tsx:280-301`）。

### ダブルバッファ + diff 更新メカニズム

`src/ink/log-update.ts` の `LogUpdate.render()` は diff エンジンのコアである。`prev: Frame` と `next: Frame` を受け取り、`Diff` パッチ操作のセットを返す：

```
Diff パッチタイプ：
- { type: 'stdout', content: string }    — ターミナルへの直接書き込み
- { type: 'cursorMove', x, y }           — 相対カーソル移動
- { type: 'cursorTo', x, y }             — 絶対カーソル位置決め
- { type: 'styleStr', str }              — SGR スタイル切替
- { type: 'hyperlink', uri }             — OSC 8 ハイパーリンク
- { type: 'cursorHide' | 'cursorShow' }  — カーソルの可視性
- { type: 'clear', count }               — 文字の消去
- { type: 'clearTerminal', reason }      — 全画面再描画（フリッカーを引き起こす）
```

diff エンジンは `src/ink/optimizer.ts` で単一パスの最適化（`optimize(diff)`）を経て、隣接する `cursorMove` をマージし、連続する `cursorTo` を折りたたみ、隣接する `styleStr` を結合し、cursor hide/show ペアをキャンセルすることで、実際の書き込み量を削減する。

### Int32Array スクリーンバッファ

`src/ink/screen.ts` は `Screen` 型を定義し、オブジェクト配列ではなく `Int32Array` を使って画面セルを格納し、GC プレッシャーを排除する（`screen.ts:356-370`）：

```typescript
// screen.ts:356-370（コメント）
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen` は `ArrayBuffer` を割り当て、同時に 2 つのビュー（`Int32Array` と `BigInt64Array`）をマウントし、後者は `resetScreen` の一括ゼロクリアに使用される（`screen.ts:472-490`）：

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes per cell
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

文字とスタイルは**共有 Pool オブジェクト**（`CharPool`、`StylePool`、`HyperlinkPool`）を通じてインターンされ、Screen 間で ID を共有する。これにより `blitRegion` は再インターンなしに整数 ID をそのままコピーでき、`diffEach` は文字列比較の代わりに整数比較を使用できる（`screen.ts:17-31`）。

`StylePool.intern()` の bit-0 エンコードは可視性を表す（`screen.ts:148-161`）：奇数 ID はそのスタイルが空白に対して可視（背景色、反転色、下線）であることを示し、レンダラーは単一のビットマスクで不可視の空白をスキップでき、不必要な書き込みを回避する。

### ハードウェアスクロール（DECSTBM）

`log-update.ts` では DECSTBM ハードウェアスクロール最適化が実装されている（`log-update.ts:149-185`）：

alt-screen モードで `ScrollBox` の `scrollTop` が変化した場合、スクロール領域全体を再書き込みするのではなく、`CSI top;bot r`（スクロール領域の設定）+ `CSI n S`（n 行上スクロール）を送信し、ターミナルハードウェアがコンテンツの移動を加速完了させ、新たにスクロールインした行の補填のみが必要になる。同時に `prev.screen` に対して `shiftRows()` を実行してスクロールをシミュレートし、後続の diff ループが差分行を自然に検出し、全量再書き込みとならないようにする。

この最適化は `decstbmSafe=true` の場合にのみ有効になる——外部ターミナルが DEC 2026（BSU/ESU アトミック操作）をサポートしていない場合、tmux などのターミナルは中間状態（スクロール後に行が補填される前）をレンダリングし、目に見えるちらつきを引き起こすため、全量 diff にフォールバックする（`log-update.ts:158-167`）。

### 16,384 文字キャッシュ

`src/ink/output.ts` の `Output` クラス（レンダラー内部の文字測定と出力エンジン）は `charCache` を維持し、文字から測定幅へのマッピングをキャッシュする。キャッシュが 16,384 件を超えると直接クリアして再構築する（`output.ts:204`）：

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

同様に、`src/ink/line-width-cache.ts` は行幅キャッシュ（最大 4096 件）を維持し、ストリーミング出力中の大量の未変化行の `stringWidth` 呼び出しを再利用する（`line-width-cache.ts`）：

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### パフォーマンス最適化戦略

| 戦略 | 場所 | 効果 |
|------|------|------|
| レンダリングスケジューリング microtask 遅延 | `ink.tsx:212-216` | scheduleRender は `queueMicrotask` を使用し、useLayoutEffect がレンダリング前に完了することを保証し、カーソル位置の 1 フレーム遅延を回避する |
| throttle 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`、安定したフレームレートを維持 |
| Drain タイマー 4 倍フレームレート | `ink.tsx:757-758` | スクロール連続 drain は `FRAME_INTERVAL_MS >> 2`（約 4ms）を使用し、スクロールの滑らかさを最大化 |
| Pool 定期リセット | `ink.tsx:600-603` | 5 分ごとに char/hyperlink pool をリセットし、長時間セッションでのメモリ増大を防ぐ |
| ダメージ領域（damage rectangle） | `screen.ts:382` | 各フレームで実際に書き込まれた矩形領域のみを diff し、全画面 diff を回避 |
| Alt-screen カーソルアンカリング | `ink.tsx:578-591` | 各フレームの diff 前に `ALT_SCREEN_ANCHOR_CURSOR` で仮想カーソルを (0,0) にリセットし、tmux などのターミナルでのカーソルドリフトを防ぐ |

---

## 12.4 REPL メインインターフェース

### REPL.tsx のコンポーネント構造

`src/screens/REPL.tsx`（5,005 行）は Claude Code のメインインターフェースコンポーネントであり、プロジェクト全体で最大の単一ファイルである。トップレベルの import は 150 行を超え、セッション管理、権限システム、MCP 接続、Swarm マルチエージェント、voice 統合など、ほぼすべての機能モジュールをカバーしている。

REPL のレンダリングツリー構造（簡略化）：

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen（全画面モード）
    │   └── Box（メインレイアウト）
    │       ├── AnimatedTerminalTitle（ターミナルタブタイトル）
    │       ├── FullscreenLayout
    │       │   ├── Messages（メッセージリスト）
    │       │   ├── VirtualMessageList（仮想スクロール）
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── 各種 Callout / Survey / Dialog
    │       └── PromptInput（入力エリア）
    └── DevBar（ant-only デバッグバー）
```

全画面モードは条件分岐によって `<AlternateScreen>` で包むかどうかを決定する（`REPL.tsx:4998-5001`）：

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### メッセージレンダリングフロー

メッセージ状態は React `useState<MessageType[]>` で保持され、`handleMessageFromStream` がストリーミング SSE イベントを処理してメッセージを追加/更新する。メッセージリストは `<Messages>` または `<VirtualMessageList>` コンポーネントにレンダリングされ、後者は仮想スクロールによって可視領域のみをレンダリングし、長い会話での大量の DOM ノードを回避する。

REPL には不必要な再レンダリングを回避するための複数の最適化がある：`AnimatedTerminalTitle` は葉ノードとして独立して抽出されており、960ms アニメーションティックが REPL ツリー全体を再レンダリングすることを防ぐ（`REPL.tsx` のコメント：`Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`）。

`useDeferredValue(searchQuery)` は検索計算の遅延に使用され、`React.memo` は `PromptInput` サブツリーでキャッシュし、文字入力時の頻繁なレンダリングが上位に伝播することを防ぐ。

### 入力処理パイプライン

ユーザー入力は stdin raw mode から Ink の `parse-keypress.ts` に入り、`ParsedKey` オブジェクトとして解析され、FocusManager を通じて現在のフォーカスノードの `useInput` コールバックに配信される。`GlobalKeybindingHandlers` と `CommandKeybindingHandlers` はグローバルショートカットキー登録メカニズムを提供し、コンポーネントレベルの `useInput` より高い優先度を持つ。

入力は最終的に `PromptInput` コンポーネントに到達し、現在のモード（Vim/Standard）に応じて `VimTextInput` または `TextInput` にルーティングされる。

---

## 12.5 Vim モード

### Vim モードの実装アーキテクチャ

`src/vim/` ディレクトリ（5 ファイル、約 1,513 行）は完全な Vim キーバインド層を実装している。アーキテクチャの設計は極めてシンプルである：**型がドキュメントである**、すべてのステートマシン遷移が TypeScript 型としてエンコードされ、TypeScript の網羅的チェックがステート処理の完全性を保証する。

ファイルの役割分担：
- `types.ts`：ステートマシン型定義 + 定数セット
- `transitions.ts`：遷移関数（メインエントリーポイント `transition()`）
- `motions.ts`：カーソル移動純粋関数（`resolveMotion()`）
- `operators.ts`：操作実行関数（delete/change/yank など）
- `textObjects.ts`：テキストオブジェクト境界計算

### サポートされる操作（motions、operators、textObjects）

**Operators（演算子）**：`d`（delete）、`c`（change）、`y`（yank）——`types.ts:OPERATORS` で定義。

**Simple Motions**（`types.ts:SIMPLE_MOTIONS`）：
- 基本移動：`h`/`l`/`j`/`k`
- 単語移動：`w`/`b`/`e`/`W`/`B`/`E`
- 行位置：`0`/`^`/`$`

**Find Motions**：`f`/`F`/`t`/`T`（文字検索）、`;`/`,` 繰り返し/逆方向。

**Text Objects**（`types.ts:TEXT_OBJ_TYPES`）：`w`/`W`（単語）、`"`/`'`/`` ` ``（引用符）、`(`/`)`/`b`（括弧）、`[`/`]`（角括弧）、`{`/`}`/`B`（波括弧）、`<`/`>` （山括弧）。

**行レベル操作**：`dd`/`cc`/`yy`、`D`/`C`/`Y`、`o`/`O`（行を開く）、`J`（join）、`>>`/`<<`（インデント）。

**その他のコマンド**：`r`（文字置換）、`~`（大文字小文字切り替え）、`x`（文字削除）、`p`/`P`（ペースト）、`u`（undo）、`G`/`gg`（行ジャンプ）、`.`（dot-repeat）。

特別処理：`cw`/`cW` は次の単語先頭ではなく単語末尾まで変更する（vim のクラシックな動作、`operators.ts:91-102`）；Image ref chip はアトミック単位として認識され、単語移動は chip の途中で停止しない（`operators.ts:333-337`）。

### ステートマシンとモード切り替え

`VimState` はトップレベル型である（`types.ts:52-55`）：

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT モードは dot-repeat のために入力テキストを記録し；NORMAL モードは `CommandState` サブステートマシンを保持する。

`CommandState` には 11 種類のステートがある（`types.ts:62-75`）、idle、count 累積、operator pending、operatorCount、operatorFind、operatorTextObj、find、g-prefix、operatorG、replace、indent をカバーする。ステート図は `types.ts` の先頭に ASCII art 形式で記録されている（`types.ts:1-26`）：

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

`transition()` 関数（`transitions.ts:55-73`）は現在の `CommandState` 型に基づいて各専用処理関数にディスパッチし、各関数は `{ next?: CommandState; execute?: () => void }` を返す。`execute` が存在する場合は即座に実行し、`next` が存在する場合はステートを進め、両方とも存在しない場合は idle に戻る。

`PersistentState`（`types.ts:80-87`）はコマンド間で保持される：`lastChange`（dot-repeat レコーディング）、`lastFind`（`;`/`,` 繰り返し）、`register`（クリップボード）、`registerIsLinewise`（行レベル paste 判定）。

カウント乗算は `fromOperatorCount` で実装されている（`transitions.ts`）：`effectiveCount = state.count * motionCount`、`3d2w` = 6 単語削除をサポート。カウント上限は `MAX_VIM_COUNT = 10000`（`types.ts:96`）で、悪意ある入力を防ぐ。

---

## 12.6 コアコンポーネント分析

### PromptInput 入力コンポーネント

`src/components/PromptInput/PromptInput.tsx`（2,338 行）は Claude Code で機能密度が最も高いコンポーネントの一つであり、ほぼすべての入力関連インタラクションロジックを担っている。

**Props の規模**：`Props` 型定義は約 60 行にわたり、入力値、モード、Vim 状態、stash スタック、送信コールバック、権限コンテキスト、MCP クライアント、IDE 選択、音声統合などを含む。

**外部入力インジェクション**：`insertTextRef` は `{ insert, setInputWithCursor, cursorOffset }` インターフェースを公開し、音声認識（STT）がカーソル位置にテキストを挿入できるようにし、入力全体を置換しない（`PromptInput.tsx:180-200`）。

**カーソル管理**：独立した `cursorOffset` state を追跡し、内部変更（`trackAndSetInput` で `onInputChange` をラップ）と外部インジェクション（音声入力）を区別する。外部変更はカーソルを末尾に移動する（`PromptInput.tsx:164-176`）。

**モードルーティング**：`isVimModeEnabled()` に基づいて `<VimTextInput>` または `<TextInput>` をレンダリングし、両者は `BaseTextInputProps` インターフェースを共有する。

**ボトムレイアウト定数**（`PromptInput.tsx`）：
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
ボトムの入力スロットの最大高さは 50%で、footer、ボーダー、状態ヒント用に 5 行を確保する。

### 設定インターフェース

`src/components/Settings/Config.tsx`（1,821 行）は `/config` コマンドの設定インターフェースを実装している。

**検索駆動ナビゲーション**：Config はデフォルトで search モード（`isSearchMode = true`）に入り、ユーザーは直接入力して設定項目をフィルタリングでき、方向キーでリスト全体をスクロールする必要がない。`maxVisible = Math.max(5, paneCap - 10)` はターミナルの高さに基づいて表示可能なアイテム数を動的に計算する。

**設定カテゴリ**：`settingsItems: Setting[]` 配列ですべての設定項目を統一定義し、各項目は `boolean`（スイッチ）、`enum`（列挙選択）、`managedEnum`（カスタムコンポーネント管理）のいずれかである。

**変更追跡と元に戻す**：`changes` state がすべての変更を記録し；`isDirty` ref は Escape キーが押された際にディスクへの書き込みが必要かどうかを判断するために使用され、開いてすぐ閉じた場合に無意味なディスク書き込みを引き起こすことを回避する（`Config.tsx`）。

**サブメニューシステム**：`showSubmenu` state が Theme、Model、TeammateModel、ExternalIncludes、OutputStyle、Language、EnableAutoUpdates などのサブメニューの展開を制御し、`setTabsHidden` を通じて親コンポーネント Settings にタブヘッダーを非表示にするよう通知する。

**マルチソース設定アーキテクチャ**：設定ソースは `localSettings`（プロジェクトレベル）と `userSettings`（ユーザーレベル）に分かれており、`getSettingsForSource`/`updateSettingsForSource` で読み書きし、Escape で元に戻す際は合算されたグローバルビューを読み取るのではなく、初期スナップショットをソースごとに復元する（`Config.tsx`、コメントでは `undefined`-削除-key のセマンティクスをサポートするためと説明されている）。

### ログセレクター

`src/components/LogSelector.tsx`（1,574 行）はセッション履歴選択インターフェース（`/resume` コマンド）を実装している。

**ツリー表示**：`<TreeSelect<LogTreeNode>>` を通じて分岐セッション（fork）を親子ツリー形式で表示し、折りたたみ/展開は `expandedGroupSessionIds` で管理される。

**3 種類のビューモード**：`viewMode` は `"list"`（デフォルトリスト）、`"search"`（検索）、`"preview"`（セッションプレビュー）のいずれかになれる。

**Fuse.js ファジー検索**：`FUSE_THRESHOLD = 0.3` のファジーマッチングと、`DATE_TIE_THRESHOLD_MS = 60 * 1000`（1 分以内の結果は関連性でソート、それ以外は時間でソート）を組み合わせて使用する。

**検索結果スニペットプレビュー**：`extractSnippet()` はマッチ位置の前後それぞれ `SNIPPET_CONTEXT_CHARS = 50` 文字を取得し、`chalk.dim` でコンテキストをレンダリングし、ハイライト色でマッチした単語をレンダリングする（`LogSelector.tsx`）。

**ディープサーチ定数**：
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
（現在 `isDeepSearchEnabled = false` で、有効化待ちの機能）

**React Compiler 最適化**：ファイル先頭の `import { c as _c } from "react/compiler-runtime"` は React Compiler でコンパイル済みであることを示し、`_c(247)` は 247 スロットの memo キャッシュを割り当て、大量の条件付きメモ化によりサブツリーの再レンダリングを回避している。

---

## 12.7 設計上の意思決定の分析

### なぜオリジナルの Ink レンダラーをそのまま使用しないのか

オリジナルの Ink は React ツリーを文字列にシリアライズした後、`log-update` で増分書き込みを行い、シンプルな CLI ツール向けに設計されている。Claude Code の要件はこのモデルの限界を超えている：

1. **マウス選択**：画面レベルのセル座標マッピング（ヒットテスト）が必要で、文字列モデルではサポートできない。
2. **検索ハイライト**：既にレンダリングされたフレームに反転セルをオーバーレイする必要があり、セルマトリックスを操作しなければならない。
3. **DECSTBM ハードウェアスクロール**：スクロール領域での prev/next 2 フレームの一貫性を制御する必要があり、文字列 diff ではターミナルハードウェアスクロールの効果をシミュレートできない。
4. **IME / a11y カーソル位置決め**：入力カーソルの物理的な画面座標を正確に知る必要があり、物理ターミナルカーソルを正しい位置に配置する（CJK 入力メソッドの preedit テキストは物理カーソル位置でレンダリングされる）。
5. **Alt-screen アトミック性（BSU/ESU）**：erase + paint を `\x1b[?2026h`/`\x1b[?2026l` で囲む必要があり、tmux などの外部ターミナルが中間状態をレンダリングすることを防ぐ。

### ダブルバッファリングのパフォーマンス考慮

ターミナル出力は I/O 集約的な操作であり、`write()` システムコールには固定のオーバーヘッドがある。ダブルバッファ diff 戦略は各フレームの書き込み量を O(画面サイズ) から O(変化領域) に削減する。実測コメントによると：drain-only フレーム（DECSTBM スクロールのみ、React コミットなし）は約 10 個のパッチ、約 200 バイトの出力を生成する（`ink.tsx:756`）が、全量再描画では数千バイトが必要になる。

`prevFrameContaminated` フラグは正確性のセーフティネットである：選択ハイライトオーバーレイ、`resetFramesForAltScreen()`、`forceRedraw()` などの操作が frontFrame の内容を変更した場合、次のフレームは増分 blit 最適化に依存するのではなく、全量 diff を行わなければならない（`ink.tsx:743`）。

### Vim モードの設計動機

Claude Code のコアユーザー層は重度の Vim ユーザーであり——彼らにとって、プロンプトボックスで Vim キーバインドを使用することは付加価値ではなく基本的な要件である。純粋なステートマシン実装（依存なし、副作用なし）により、Vim ロジックは完全に単体テスト可能であり、ターミナルレンダリング層から分離されている。

`types.ts` の先頭コメントは設計思想を明らかにしている：**型がドキュメントである**。`CommandState` の union type はいかなるコメントよりも正確にステートマシンを記述する——TypeScript の switch 網羅的チェックにより各ステートが処理されることが保証され、新しいステートの漏れはコンパイル時にエラーとなる（`types.ts:1`）。

---

## 12.8 移植可能なパターン

以下の設計パターンは他のターミナルアプリケーションや UI フレームワークで参考にできる：

**1. 共有 Pool + 整数 ID スクリーンバッファ**
`Int32Array` を使用して文字/スタイルの整数 ID を格納し、文字列やオブジェクトの代わりに使用する。フレーム間で Pool を共有することで diff が整数比較に降下し、GC プレッシャーを完全に排除する。頻繁な diff が必要な任意の矩形マトリックスシナリオに適用可能。

**2. 型がステートマシン**
TypeScript union type でステートマシンを定義し、各ステートが必要なデータのみを持ち、遷移関数に網羅的な switch を使用する。`transition()` + `TransitionResult` の `{next?, execute?}` デュアルキー戻り値は極めてシンプルで、他のキーバインド処理シナリオに直接移植できる。

**3. ダメージ領域（Damage Rectangle）駆動の増分 diff**
レンダリング時に実際に書き込まれた矩形領域（`damage`）を記録し、diff はこの領域のみをスキャンする。任意のフレームバッファシステムに適用可能——ターミナルだけでなく、ゲーム UI、組み込みディスプレイにもこのパターンを使用できる。

**4. ダブルバッファ + prevFrameContaminated セーフティネット**
一部の操作が通常のレンダリングパスを経ない場合（前フレームを直接変更する場合）、bool フラグを使用して次フレームの全量 diff を強制し、ダブルバッファの不変条件を破壊しない。毎回全量再描画するよりも効率的で、ダーティデータを許可するよりも正確である。

**5. レンダリング microtask 遅延**
`scheduleRender = throttle(() => queueMicrotask(onRender), 16)` のパターンにより、React layout effects がレンダリング前に完全にコミットされることが保証され、カーソル位置の 1 フレーム遅延を回避する。他の React + カスタムレンダラーシナリオにも同様に適用可能。

**6. 行幅キャッシュ（ストリーミングシナリオ専用最適化）**
ストリーミング出力中の大量の未変化行の `stringWidth` 計算はホットスポットである。`line-width-cache.ts` のシンプルな Map + 満杯でクリア戦略は 20 行のコードで `~50x` の計算オーバーヘッドを解決している（コメント原文）。任意のストリーミングテキストレンダリングシナリオに直接移植可能。

---

## 12.9 ソースコードインデックス

| ファイル | 行数 | コアの役割 |
|------|------|----------|
| `src/ink/ink.tsx` | 1,722 | Ink クラス：フレーム管理、レンダリングスケジューリング、SIGCONT、alt-screen 引き渡し、マウス選択、検索ハイライト |
| `src/ink/screen.ts` | 1,300+ | Screen 型、Int32Array セルレイアウト、CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate：prev/next フレーム diff エンジン、DECSTBM ハードウェアスクロール |
| `src/ink/optimizer.ts` | ~100 | Diff 単一パスオプティマイザー |
| `src/ink/reconciler.ts` | 512 | React カスタムレンダラーホスト実装 |
| `src/ink/render-node-to-output.ts` | 1,462 | DOM ノードから Screen へのレンダリング |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | 行幅 LRU キャッシュ（4096 件） |
| `src/ink/output.ts` | ~300 | Output レンダラー、文字キャッシュ（16,384 件） |
| `src/vim/types.ts` | ~150 | VimState、CommandState、PersistentState 型定義および定数 |
| `src/vim/transitions.ts` | ~350 | `transition()` メインエントリーポイント、11 個のステート遷移関数 |
| `src/vim/motions.ts` | ~80 | `resolveMotion()` カーソル移動純粋関数 |
| `src/vim/operators.ts` | ~500 | delete/change/yank/x/r/~/J/paste/indent/openLine 実行関数 |
| `src/vim/textObjects.ts` | ~220 | 単語/引用符/括弧などのテキストオブジェクト境界検索 |
| `src/screens/REPL.tsx` | 5,005 | メイン REPL インターフェース、セッション管理、メッセージストリーム、全画面/通常モード切り替え |
| `src/components/PromptInput/PromptInput.tsx` | 2,338 | 入力コンポーネント、Vim/Standard モードルーティング、カーソル管理、STT インジェクション |
| `src/components/Settings/Config.tsx` | 1,821 | 設定インターフェース、検索ナビゲーション、マルチソース設定読み書き、Escape 元に戻す |
| `src/components/LogSelector.tsx` | 1,574 | セッション履歴選択、Fuse.js 検索、ツリー形式の fork 表示 |

**主要定数クイックリファレンス**：
- `FRAME_INTERVAL_MS = 16`（`constants.ts:1`）
- `MAX_CACHE_SIZE = 4096`（`line-width-cache.ts`）
- `charCache` 上限 `16384`（`output.ts:204`）
- `MAX_VIM_COUNT = 10000`（`vim/types.ts:96`）
- セルレイアウト：8 bytes/cell、`word0=charId`、`word1=styleId[31:17]|hyperlinkId[16:2]|width[1:0]`（`screen.ts:356`）
