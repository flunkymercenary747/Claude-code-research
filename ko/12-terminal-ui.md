# 제 12 장: 터미널 UI와 렌더링 엔진

## 12.1 개요 및 위상

Claude Code는 터미널에서 실행되는 대화형 AI 코딩 어시스턴트로, UI 레이어는 순수 텍스트 터미널 환경에서 GUI에 가까운 인터랙티브 경험을 구현해야 합니다. 스트리밍 메시지 렌더링, Vim 키 편집, 마우스 텍스트 선택, 검색 하이라이트, 다층 팝업 중첩 등이 그 요구 사항입니다. 이 요구들은 전통적인 CLI 도구의 능력 범위를 훨씬 초과합니다.

Claude Code의 터미널 UI 레이어는 세 가지 핵심 기술 결정 위에 구축되어 있습니다:

1. **커스텀 Ink 렌더러** (`src/ink/`): React + Ink 프레임워크를 깊이 개조하여 이중 버퍼링 스크린 모델, ANSI diff 엔진, 하드웨어 스크롤 가속, 그리고 IME 인식 물리적 커서 위치 지정을 도입했습니다.
2. **선언적 컴포넌트 트리** (`src/screens/REPL.tsx`, `src/components/`): React 컴포넌트로 UI 구조를 기술하고, 렌더링 엔진이 가상 DOM을 터미널 문자 행렬로 변환하는 역할을 담당합니다.
3. **완전한 Vim 모드** (`src/vim/`): 독립적인 상태 머신 구현으로, NORMAL/INSERT 이중 모드, operators/motions/textObjects 전체 집합, dot-repeat과 레지스터를 포함합니다.

전체 렌더링 파이프라인의 박자는 `FRAME_INTERVAL_MS = 16` (`src/ink/constants.ts:1`)이 제어하며, 이론적 프레임 속도는 62.5fps로 모니터 새로 고침 빈도와 맞춰져 있습니다.

---

## 12.2 이론적 기초

### 터미널 렌더링의 기반 원리

터미널 렌더링의 본질은 파일 디스크립터로 바이트 스트림을 쓰는 것이며, 터미널 에뮬레이터가 이를 파싱하여 화면을 업데이트합니다. 핵심 메커니즘:

- **ANSI escape codes**: `\x1b[` 로 시작하는 제어 시퀀스로, 커서 위치(CSI CUP), 색상(SGR), 스크롤 영역(DECSTBM), 마우스 추적(DEC private modes) 등을 제어합니다. Claude Code의 `src/ink/termio/` 디렉토리는 이 시퀀스들을 명명된 상수(`CURSOR_HOME`, `ENTER_ALT_SCREEN`, `ENABLE_MOUSE_TRACKING` 등)로 캡슐화합니다.
- **termios / raw mode**: raw mode 진입 후 키 입력이 즉시 프로그램에 전달되며, 라인 버퍼링이나 에코를 거치지 않습니다. Claude Code는 시작 시 `setRawMode(true)`로 stdin을 인계받고, `Ink.detachForShutdown()` 종료 시 복원합니다 (`ink.tsx:942-950`).
- **Alt Screen (`?1049h/?1049l`)**: 대체 스크린 버퍼로, 진입 시 메인 스크린 스크롤 기록을 오염시키지 않고 종료 시 원래 상태로 복원됩니다. 전체 화면 모드(`isFullscreenEnvEnabled()`)에서 REPL은 `<AlternateScreen>` 컴포넌트로 감쌉니다.

### 이중 버퍼링 렌더링

Claude Code는 앞뒤 두 개의 프레임 버퍼(`frontFrame` / `backFrame`)로 이중 버퍼링을 구현합니다 (`ink.tsx:196-197`):

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

각 프레임 렌더링 시:
1. Renderer가 React 가상 DOM을 `backFrame`(새 프레임)에 씁니다.
2. LogUpdate diff 엔진이 `frontFrame`(이전 프레임)과 `backFrame`을 비교하여 최소 패치 시퀀스를 생성합니다.
3. 패치 시퀀스가 터미널에 쓰이고, `frontFrame` ↔ `backFrame`이 교환됩니다 (`ink.tsx:594`).

이로써 터미널은 실제로 변화된 문자만 받게 되고, 매 프레임 전체를 다시 그리지 않아도 됩니다.

### 비-DOM 환경에서 React 적용 (Custom Renderer)

React의 렌더러 인터페이스(`react-reconciler`)는 임의의 호스트 환경을 허용합니다. Claude Code는 `src/ink/reconciler.ts`(512줄)에서 터미널용 Custom Renderer를 구현했습니다:

- 호스트 엘리먼트 타입은 커스텀 `dom.DOMElement` (`src/ink/dom.ts`)
- 레이아웃 엔진은 Yoga(Facebook의 Flexbox 구현)를 사용하며, `onComputeLayout` 콜백에서 구동됩니다 (`ink.tsx:239-258`)
- React Concurrent Root 모드(`ConcurrentRoot`)로 렌더링을 중단 가능하게 하여 메인 스레드 장시간 블로킹을 방지합니다.

---

## 12.3 Ink 커스텀 렌더러

### Claude Code의 Ink 커스텀 개조

`src/ink/ink.tsx`(1,722줄)는 전체 UI 레이어의 핵심 컨트롤러로, `Ink` 클래스를 정의합니다. 이것은 원본 Ink 패키지를 그대로 사용하는 것이 아니라 깊이 개조한 커스텀 버전으로, 핵심 향상 사항은 다음과 같습니다:

**이중 버퍼링 + 앞뒤 프레임 관리**: 원본 Ink는 문자열을 직접 출력하지만, 커스텀 버전은 프레임 객체(`Frame`)와 `Screen` 행렬을 도입했습니다.

**성능 모니터링**: 매 프레임마다 renderer, diff, optimize, write 각 단계 소요 시간과 Yoga 레이아웃의 visited/measured/cacheHits 지표를 기록하고, `onFrame` 콜백으로 노출합니다 (`ink.tsx:772-788`):

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

**외부 TUI 인계**: `enterAlternateScreen()` / `exitAlternateScreen()` 메서드를 제공하여, Ink 제어권을 일시 정지하고 kitty keyboard 프로토콜을 닫아 외부 편집기(vim, nano)가 정상 동작하게 하고, 복귀 시 완전히 복원합니다 (`ink.tsx:357-419`).

**SIGCONT 자동 복구**: `handleResume`은 프로세스가 일시 정지에서 재개될 때(fg 명령) 처리합니다. 메인 스크린은 버퍼를 재설정하고, alt 스크린은 재진입하고 화면을 지웁니다 (`ink.tsx:280-301`).

### 이중 버퍼링 + diff 업데이트 메커니즘

`src/ink/log-update.ts`의 `LogUpdate.render()`는 diff 엔진의 핵심입니다. `prev: Frame`과 `next: Frame`을 받아 `Diff` 패치 작업 집합을 반환합니다:

```
Diff patch 유형:
- { type: 'stdout', content: string }    — 터미널에 직접 쓰기
- { type: 'cursorMove', x, y }           — 상대적 커서 이동
- { type: 'cursorTo', x, y }             — 절대 커서 위치 지정
- { type: 'styleStr', str }              — SGR 스타일 전환
- { type: 'hyperlink', uri }             — OSC 8 하이퍼링크
- { type: 'cursorHide' | 'cursorShow' }  — 커서 가시성
- { type: 'clear', count }               — 문자 지우기
- { type: 'clearTerminal', reason }      — 전체 화면 다시 그리기 (flicker 유발)
```

diff 엔진은 `src/ink/optimizer.ts`에서 단일 패스 최적화를 거칩니다(`optimize(diff)`). 인접 `cursorMove` 병합, 연속 `cursorTo` 접기, 인접 `styleStr` 이어 붙이기, cursor hide/show 쌍 취소를 통해 실제 쓰기 양을 줄입니다.

### Int32Array 스크린 버퍼

`src/ink/screen.ts`는 `Screen` 타입을 정의하며, 객체 배열 대신 `Int32Array`를 사용하여 스크린 셀을 저장함으로써 GC 압박을 제거합니다 (`screen.ts:356-370`):

```typescript
// screen.ts:356-370 (주석)
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen`은 `ArrayBuffer`를 할당하고 두 개의 뷰(`Int32Array`와 `BigInt64Array`)를 동시에 마운트합니다. 후자는 `resetScreen`의 일괄 초기화에 사용됩니다 (`screen.ts:472-490`):

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes per cell
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

문자와 스타일은 **공유 Pool 객체** (`CharPool`, `StylePool`, `HyperlinkPool`)로 intern되어 스크린 간 ID를 공유합니다. 이로써 `blitRegion`은 re-intern 없이 정수 ID를 직접 복사할 수 있고, `diffEach`는 문자열 비교 대신 정수 비교를 사용할 수 있습니다 (`screen.ts:17-31`).

`StylePool.intern()`의 bit-0 인코딩은 가시성을 나타냅니다 (`screen.ts:148-161`): 홀수 ID는 해당 스타일이 공백에 대해 가시적임(배경색, 반전색, 밑줄)을 의미하여, 렌더러가 단일 비트 마스크로 보이지 않는 공백을 건너뛰고 불필요한 쓰기를 방지할 수 있습니다.

### 하드웨어 스크롤 (DECSTBM)

`log-update.ts`에는 DECSTBM 하드웨어 스크롤 최적화가 구현되어 있습니다 (`log-update.ts:149-185`):

alt-screen 모드에서 `ScrollBox`의 `scrollTop`이 변경될 때, 전체 스크롤 영역을 다시 쓰는 대신 `CSI top;bot r`(스크롤 영역 설정) + `CSI n S`(n행 위로 스크롤)를 전송합니다. 터미널 하드웨어가 콘텐츠 이동을 가속 처리하고, 새로 들어오는 행만 보충합니다. 동시에 `prev.screen`에 `shiftRows()`를 적용해 스크롤을 시뮬레이션함으로써, 이후 diff 루프가 차이 행을 자연스럽게 찾을 수 있게 합니다.

이 최적화는 `decstbmSafe=true` 일 때만 활성화됩니다. 외부 터미널이 DEC 2026(BSU/ESU 원자 연산)을 지원하지 않으면, tmux 등의 터미널이 중간 상태(스크롤 후 보충 전)를 렌더링하여 눈에 띄는 깜빡임이 발생하므로, 이때는 전체 diff로 돌아갑니다 (`log-update.ts:158-167`).

### 16,384자 캐시

`src/ink/output.ts`의 `Output` 클래스(렌더러 내부의 문자 측정 및 출력 엔진)는 `charCache`를 유지하여 문자에서 측정 너비로의 매핑을 캐시합니다. 캐시가 16,384개를 초과하면 직접 비우고 재구성합니다 (`output.ts:204`):

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

마찬가지로, `src/ink/line-width-cache.ts`는 행 너비 캐시(최대 4096개)를 유지하여, 스트리밍 출력 중 변하지 않은 많은 행의 `stringWidth` 호출을 재사용합니다 (`line-width-cache.ts`):

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### 성능 최적화 전략

| 전략 | 위치 | 효과 |
|------|------|------|
| 렌더링 스케줄 microtask 지연 | `ink.tsx:212-216` | scheduleRender이 `queueMicrotask`를 사용하여, useLayoutEffect가 렌더링 전에 완료되고 커서 위치 단일 프레임 지연을 방지함 |
| throttle 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`로 안정적인 프레임 속도 유지 |
| Drain timer 4배 프레임 속도 | `ink.tsx:757-758` | scroll 연속 drain은 `FRAME_INTERVAL_MS >> 2`(약 4ms)를 사용하여 스크롤 부드러움을 극대화함 |
| Pool 주기적 재설정 | `ink.tsx:600-603` | 5분마다 char/hyperlink pool 재설정으로 장시간 세션 메모리 증가 방지 |
| 손상 영역(damage rectangle) | `screen.ts:382` | 매 프레임마다 실제로 쓰여진 직사각형 영역만 diff하고 전체 화면은 하지 않음 |
| Alt-screen 커서 앵커링 | `ink.tsx:578-591` | 매 프레임 diff 전에 `ALT_SCREEN_ANCHOR_CURSOR`로 가상 커서를 (0,0)으로 재설정하여 tmux 등 터미널의 커서 드리프트 방지 |

---

## 12.4 REPL 메인 인터페이스

### REPL.tsx의 컴포넌트 구조

`src/screens/REPL.tsx`(5,005줄)는 Claude Code의 메인 인터페이스 컴포넌트이자, 프로젝트 전체에서 가장 큰 단일 파일입니다. 최상단 import가 150줄을 넘어 세션 관리, 권한 시스템, MCP 연결, Swarm 다중 에이전트, voice 통합 등 거의 모든 기능 모듈을 포함합니다.

REPL 렌더 트리 구조 (단순화):

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen (전체 화면 모드)
    │   └── Box (메인 레이아웃)
    │       ├── AnimatedTerminalTitle (터미널 탭 제목)
    │       ├── FullscreenLayout
    │       │   ├── Messages (메시지 목록)
    │       │   ├── VirtualMessageList (가상 스크롤)
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── 각종 Callout / Survey / Dialog
    │       └── PromptInput (입력 영역)
    └── DevBar (ant-only 디버그 바)
```

전체 화면 모드는 조건 분기로 `<AlternateScreen>` 감쌈 여부를 결정합니다 (`REPL.tsx:4998-5001`):

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### 메시지 렌더링 흐름

메시지 상태는 React `useState<MessageType[]>`가 보유하며, `handleMessageFromStream`으로 스트리밍 SSE 이벤트를 처리하여 메시지를 추가/업데이트합니다. 메시지 목록은 `<Messages>` 또는 `<VirtualMessageList>` 컴포넌트로 렌더링됩니다. 후자는 가상 스크롤로 보이는 영역만 렌더링하여 긴 대화에서 대량의 DOM 노드를 방지합니다.

REPL에는 불필요한 재렌더링을 피하는 최적화가 여러 곳에 있습니다: `AnimatedTerminalTitle`은 리프 노드로 별도 추출되어 960ms 애니메이션 tick이 전체 REPL 트리를 재렌더링하지 않게 합니다 (`REPL.tsx` 주석: `Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`).

`useDeferredValue(searchQuery)`는 검색 계산을 지연시키고, `React.memo`는 `PromptInput` 서브트리를 캐시하여 입력 문자 시 빈번한 렌더링이 상위로 전파되는 것을 방지합니다.

### 입력 처리 파이프라인

사용자 입력이 stdin raw mode에서 Ink의 `parse-keypress.ts`로 들어와 `ParsedKey` 객체로 파싱되고, FocusManager를 통해 현재 포커스 노드의 `useInput` 콜백으로 분배됩니다. `GlobalKeybindingHandlers`와 `CommandKeybindingHandlers`는 컴포넌트 수준 `useInput`보다 우선순위가 높은 전역 단축키 등록 메커니즘을 제공합니다.

입력은 최종적으로 `PromptInput` 컴포넌트에 도달하고, 현재 모드(Vim/Standard)에 따라 `VimTextInput` 또는 `TextInput`으로 분기됩니다.

---

## 12.5 Vim 모드

### Vim 모드의 구현 아키텍처

`src/vim/` 디렉토리(5개 파일, ~1,513줄)는 완전한 Vim 키 레이어를 구현합니다. 아키텍처 설계가 극도로 간결합니다: **타입이 곧 문서**이며, 모든 상태 머신 전환이 TypeScript 타입으로 인코딩되어 TypeScript의 철저한 검사가 상태 처리의 완전성을 보장합니다.

파일 역할:
- `types.ts`: 상태 머신 타입 정의 + 상수 집합
- `transitions.ts`: 전환 함수 (메인 진입점 `transition()`)
- `motions.ts`: 커서 이동 순수 함수 (`resolveMotion()`)
- `operators.ts`: 작업 실행 함수 (delete/change/yank 등)
- `textObjects.ts`: 텍스트 객체 경계 계산

### 지원 작업 (motions, operators, textObjects)

**Operators**: `d`(delete), `c`(change), `y`(yank) — `types.ts:OPERATORS`에 정의.

**Simple Motions** (`types.ts:SIMPLE_MOTIONS`):
- 기본 이동: `h`/`l`/`j`/`k`
- 단어 이동: `w`/`b`/`e`/`W`/`B`/`E`
- 행 위치: `0`/`^`/`$`

**Find Motions**: `f`/`F`/`t`/`T` (문자 찾기), `;`/`,` 반복/역방향.

**Text Objects** (`types.ts:TEXT_OBJ_TYPES`): `w`/`W` (단어), `"`/`'`/`` ` `` (따옴표), `(`/`)`/`b` (괄호), `[`/`]` (대괄호), `{`/`}`/`B` (중괄호), `<`/`>` (꺾쇠 괄호).

**행 수준 작업**: `dd`/`cc`/`yy`, `D`/`C`/`Y`, `o`/`O` (행 열기), `J` (결합), `>>`/`<<` (들여쓰기).

**기타 명령**: `r` (문자 대체), `~` (대소문자 전환), `x` (문자 삭제), `p`/`P` (붙여넣기), `u` (실행 취소), `G`/`gg` (행 이동), `.` (dot-repeat).

특수 처리: `cw`/`cW`는 다음 단어 시작이 아닌 단어 끝까지 변경합니다 (vim 클래식 동작, `operators.ts:91-102`). Image ref chip은 원자 단위로 인식되어 단어 이동이 chip 중간에 멈추지 않습니다 (`operators.ts:333-337`).

### 상태 머신과 모드 전환

`VimState`는 최상위 타입입니다 (`types.ts:52-55`):

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT 모드는 입력된 텍스트를 dot-repeat용으로 기록하고, NORMAL 모드는 `CommandState` 하위 상태 머신을 보유합니다.

`CommandState`는 11가지 상태가 있습니다 (`types.ts:62-75`). idle, count accumulation, operator pending, operatorCount, operatorFind, operatorTextObj, find, g-prefix, operatorG, replace, indent를 포함합니다. 상태 다이어그램은 `types.ts` 시작 부분에 ASCII art로 기록되어 있습니다 (`types.ts:1-26`):

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

`transition()` 함수(`transitions.ts:55-73`)는 현재 `CommandState` 타입에 따라 각 전용 처리 함수로 분배하며, 각 함수는 `{ next?: CommandState; execute?: () => void }`를 반환합니다. `execute`가 있으면 즉시 실행하고, `next`가 있으면 상태를 전진시키며, 둘 다 없으면 idle로 돌아갑니다.

`PersistentState` (`types.ts:80-87`)는 명령 간에 지속됩니다: `lastChange` (dot-repeat 기록), `lastFind` (`;`/`,` 반복), `register` (클립보드), `registerIsLinewise` (행 단위 paste 판단).

카운트 곱셈은 `fromOperatorCount`에서 구현됩니다 (`transitions.ts`): `effectiveCount = state.count * motionCount`로 `3d2w` = 6개 단어 삭제를 지원합니다. 카운트 상한은 `MAX_VIM_COUNT = 10000` (`types.ts:96`)으로 악의적인 입력을 방지합니다.

---

## 12.6 핵심 컴포넌트 분석

### PromptInput 입력 컴포넌트

`src/components/PromptInput/PromptInput.tsx`(2,338줄)는 Claude Code에서 기능 밀도가 가장 높은 컴포넌트 중 하나로, 거의 모든 입력 관련 인터랙션 로직을 담고 있습니다.

**Props 규모**: `Props` 타입 정의가 약 60줄에 걸쳐 있으며, 입력 값, 모드, Vim 상태, stash 스택, 제출 콜백, 권한 컨텍스트, MCP 클라이언트, IDE 선택, 음성 통합 등을 포함합니다.

**외부 입력 주입**: `insertTextRef`가 `{ insert, setInputWithCursor, cursorOffset }` 인터페이스를 노출하여, 음성 인식(STT)이 전체 입력을 대체하는 대신 커서 위치에 텍스트를 삽입할 수 있습니다 (`PromptInput.tsx:180-200`).

**커서 관리**: `cursorOffset` 상태를 독립적으로 추적하여, 내부 변경(`trackAndSetInput`으로 `onInputChange` 래핑)과 외부 주입(음성 입력)을 구분합니다. 외부 변경은 커서를 끝으로 이동시킵니다 (`PromptInput.tsx:164-176`).

**모드 라우팅**: `isVimModeEnabled()`에 따라 `<VimTextInput>` 또는 `<TextInput>`을 렌더링하며, 두 컴포넌트는 `BaseTextInputProps` 인터페이스를 공유합니다.

**하단 레이아웃 상수** (`PromptInput.tsx`):
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
하단 입력 슬롯의 최대 높이는 50%이며, footer, 테두리, 상태 힌트를 위해 5행을 예약합니다.

### 설정 인터페이스

`src/components/Settings/Config.tsx`(1,821줄)는 `/config` 명령의 설정 인터페이스를 구현합니다.

**검색 기반 내비게이션**: Config는 기본적으로 search 모드(`isSearchMode = true`)로 시작하여, 사용자가 방향키로 전체 목록을 스크롤하지 않고 직접 입력해 설정 항목을 필터링할 수 있습니다. `maxVisible = Math.max(5, paneCap - 10)`은 터미널 높이에 따라 표시 항목 수를 동적으로 계산합니다.

**설정 분류**: `settingsItems: Setting[]` 배열이 모든 설정 항목을 통합 정의하며, 각 항목은 `boolean`(토글), `enum`(열거형 선택), `managedEnum`(커스텀 컴포넌트 관리) 중 하나입니다.

**변경 추적 및 복원**: `changes` 상태가 모든 수정을 기록하고, `isDirty` ref는 Escape 키 시 저장 여부를 판단하여 열고 닫기만 해도 의미없는 디스크 쓰기가 발생하는 것을 방지합니다 (`Config.tsx`).

**서브메뉴 시스템**: `showSubmenu` 상태가 Theme, Model, TeammateModel, ExternalIncludes, OutputStyle, Language, EnableAutoUpdates 등 서브메뉴의 펼침을 제어하고, `setTabsHidden`으로 상위 컴포넌트 Settings에 Tab 헤더 숨김을 전달합니다.

**다중 소스 설정 아키텍처**: 설정 소스는 `localSettings`(프로젝트 수준)와 `userSettings`(사용자 수준)로 나뉘며, `getSettingsForSource`/`updateSettingsForSource`로 읽고 씁니다. Escape 복원 시 병합된 전역 뷰가 아닌 소스별 초기 스냅샷으로 복원합니다 (`Config.tsx`, 주석은 이것이 `undefined`-삭제-키의 의미론을 지원하기 위한 것이라고 설명합니다).

### 로그 선택기

`src/components/LogSelector.tsx`(1,574줄)는 세션 기록 선택 인터페이스(`/resume` 명령)를 구현합니다.

**트리 형태 표시**: `<TreeSelect<LogTreeNode>>`를 통해 분기 세션(fork)을 부모-자식 트리로 표시하고, 접기/펼치기는 `expandedGroupSessionIds`로 관리합니다.

**세 가지 뷰 모드**: `viewMode`는 `"list"` (기본 목록), `"search"` (검색), `"preview"` (세션 미리보기)가 될 수 있습니다.

**Fuse.js 퍼지 검색**: `FUSE_THRESHOLD = 0.3`의 퍼지 매칭을 사용하고, `DATE_TIE_THRESHOLD_MS = 60 * 1000`(1분 이내 결과는 관련성 순, 그 외는 시간 순)을 결합합니다.

**검색 결과 스니펫 미리보기**: `extractSnippet()`은 매칭 위치 앞뒤 각 `SNIPPET_CONTEXT_CHARS = 50` 문자를 추출하고, `chalk.dim`으로 컨텍스트를, 하이라이트 색으로 매칭 단어를 렌더링합니다 (`LogSelector.tsx`).

**심층 검색 상수**:
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
(현재 `isDeepSearchEnabled = false`로 활성화 대기 중인 기능)

**React Compiler 최적화**: 파일 상단의 `import { c as _c } from "react/compiler-runtime"`은 React Compiler로 컴파일되었음을 나타내며, `_c(247)`이 247개 슬롯의 memo 캐시를 할당하고, 다수의 조건부 기억화로 서브트리 재렌더링을 방지합니다.

---

## 12.7 설계 결정 분석

### 원본 Ink를 그대로 쓰지 않고 커스텀 렌더러를 만든 이유

원본 Ink는 React 트리를 문자열로 직렬화한 후 `log-update`로 증분 쓰기를 하며, 단순한 CLI 도구를 위해 설계되었습니다. Claude Code의 요구 사항은 이 모델의 한계를 초과합니다:

1. **마우스 선택**: 화면 수준의 셀 좌표 매핑(hit-test)이 필요하며, 문자열 모델로는 지원 불가.
2. **검색 하이라이트**: 이미 렌더링된 프레임에 반전된 셀을 겹쳐야 하므로 셀 행렬 조작이 필수.
3. **DECSTBM 하드웨어 스크롤**: 스크롤 영역에서 prev/next 두 프레임의 일관성을 제어해야 하며, 문자열 diff로는 터미널 하드웨어 스크롤 효과를 시뮬레이션할 수 없음.
4. **IME / a11y 커서 위치 지정**: 입력 커서의 물리적 화면 좌표를 정확히 알아야 하며, 물리적 터미널 커서를 올바른 위치에 놓아야 함 (CJK 입력기의 preedit 텍스트가 물리적 커서 위치에 렌더링됨).
5. **Alt-screen 원자성 (BSU/ESU)**: erase + paint를 `\x1b[?2026h`/`\x1b[?2026l`로 감싸서 tmux 등 외부 터미널이 중간 상태를 렌더링하는 것을 방지해야 함.

### 이중 버퍼링의 성능 고려

터미널 출력은 I/O 집약적 작업으로, `write()` 시스템 호출에는 고정 오버헤드가 있습니다. 이중 버퍼링 diff 전략은 매 프레임 쓰기 양을 O(화면 크기)에서 O(변화 영역)으로 줄입니다. 실측 주석에 따르면: drain-only 프레임(DECSTBM scroll만, React commit 없음)은 약 ~10개 패치, ~200바이트 출력을 생성하는 반면(`ink.tsx:756`), 전체 재그리기는 수천 바이트가 필요합니다.

`prevFrameContaminated` 플래그는 정확성 안전망입니다: 선택 하이라이트 오버레이, `resetFramesForAltScreen()`, `forceRedraw()` 등의 작업이 frontFrame 내용을 수정했을 때, 다음 프레임은 증분 blit 최적화에 의존하지 않고 전체 diff를 수행해야 합니다 (`ink.tsx:743`).

### Vim 모드의 설계 동기

Claude Code의 핵심 사용자층은 헤비 Vim 사용자들로, 그들에게 프롬프트 박스에서 Vim 키를 사용하는 것은 부가적인 기능이 아니라 기본 요구 사항입니다. 순수 상태 머신 구현(의존성 없음, 부작용 없음)으로 Vim 로직이 완전히 단위 테스트 가능하며, 터미널 렌더링 레이어와 분리됩니다.

`types.ts` 헤더의 주석은 설계 철학을 드러냅니다: **타입이 곧 문서**. `CommandState`의 union type은 어떤 주석보다 더 정확하게 상태 머신을 기술하며, TypeScript의 switch 철저 검사가 모든 상태의 처리를 보장하고, 새 상태의 누락은 컴파일 시간에 오류로 나타납니다 (`types.ts:1`).

---

## 12.8 이전 가능한 패턴

다음 설계 패턴들은 다른 터미널 애플리케이션이나 UI 프레임워크에서 참고할 수 있습니다:

**1. 공유 Pool + 정수 ID 스크린 버퍼**
문자열이나 객체 대신 `Int32Array`를 사용해 문자/스타일의 정수 ID를 저장합니다. 프레임 간 Pool 공유로 diff가 정수 비교로 줄어들고, GC 압박을 완전히 제거합니다. 고주파 diff가 필요한 임의의 직사각형 행렬 시나리오에 적용 가능합니다.

**2. 타입이 곧 상태 머신**
TypeScript union type으로 상태 머신을 정의하고, 각 상태가 정확히 필요한 데이터를 담으며, 상태 전환 함수가 exhaustive switch를 사용합니다. `transition()` + `TransitionResult`의 `{next?, execute?}` 두 키 반환 값이 극도로 간결하며, 다른 키 처리 시나리오에 직접 이식 가능합니다.

**3. 손상 영역(Damage Rectangle) 기반 증분 diff**
렌더링 시 실제로 쓰여진 직사각형 영역(`damage`)을 기록하고, diff는 이 영역만 스캔합니다. 터미널뿐 아니라 게임 UI, 임베디드 디스플레이 등 임의의 프레임 버퍼 시스템에 사용 가능합니다.

**4. 이중 버퍼링 + prevFrameContaminated 안전망**
일부 작업이 정상 렌더링 경로를 거치지 않고(앞 프레임을 직접 수정할 때), bool 플래그로 다음 프레임의 전체 diff를 강제하여 이중 버퍼링 불변성을 깨트리지 않습니다. 매번 전체 재그리기보다 효율적이고, 더티 데이터 허용보다 정확합니다.

**5. 렌더링 microtask 지연**
`scheduleRender = throttle(() => queueMicrotask(onRender), 16)` 패턴으로 React layout effect가 렌더링 전에 완전히 커밋되게 하여 커서 위치 단일 프레임 지연을 방지합니다. 다른 React + 커스텀 렌더러 시나리오에서도 동일하게 적용 가능합니다.

**6. 행 너비 캐시 (스트리밍 시나리오 전용 최적화)**
스트리밍 출력에서 변하지 않는 많은 행의 `stringWidth` 계산이 핫 패스입니다. `line-width-cache.ts`의 단순 Map + 가득 차면 비우기 전략이 20줄 코드로 ~50배의 계산 오버헤드를 해결합니다(주석 원문). 임의의 스트리밍 텍스트 렌더링 시나리오에 직접 이식 가능합니다.

---

## 12.9 소스 코드 인덱스

| 파일 | 줄 수 | 핵심 역할 |
|------|-------|----------|
| `src/ink/ink.tsx` | 1,722 | Ink 클래스: 프레임 관리, 렌더링 스케줄, SIGCONT, alt-screen 인계, 마우스 선택, 검색 하이라이트 |
| `src/ink/screen.ts` | 1,300+ | Screen 타입, Int32Array 셀 레이아웃, CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate: prev/next 프레임 diff 엔진, DECSTBM 하드웨어 스크롤 |
| `src/ink/optimizer.ts` | ~100 | Diff 단일 패스 최적화기 |
| `src/ink/reconciler.ts` | 512 | React Custom Renderer 호스트 구현 |
| `src/ink/render-node-to-output.ts` | 1,462 | DOM 노드에서 Screen으로의 렌더링 |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | 행 너비 LRU 캐시 (4096개) |
| `src/ink/output.ts` | ~300 | Output 렌더러, 문자 캐시 (16,384개) |
| `src/vim/types.ts` | ~150 | VimState, CommandState, PersistentState 타입 정의 및 상수 |
| `src/vim/transitions.ts` | ~350 | `transition()` 메인 진입점, 11개 상태 전환 함수 |
| `src/vim/motions.ts` | ~80 | `resolveMotion()` 커서 이동 순수 함수 |
| `src/vim/operators.ts` | ~500 | delete/change/yank/x/r/~/J/paste/indent/openLine 실행 함수 |
| `src/vim/textObjects.ts` | ~220 | 단어/따옴표/괄호 등 텍스트 객체 경계 찾기 |
| `src/screens/REPL.tsx` | 5,005 | 메인 REPL 인터페이스, 세션 관리, 메시지 스트림, 전체 화면/일반 모드 전환 |
| `src/components/PromptInput/PromptInput.tsx` | 2,338 | 입력 컴포넌트, Vim/Standard 모드 라우팅, 커서 관리, STT 주입 |
| `src/components/Settings/Config.tsx` | 1,821 | 설정 인터페이스, 검색 내비게이션, 다중 소스 설정 읽기/쓰기, Escape 복원 |
| `src/components/LogSelector.tsx` | 1,574 | 세션 기록 선택, Fuse.js 검색, 트리형 fork 표시 |

**핵심 상수 참조**:
- `FRAME_INTERVAL_MS = 16` (`constants.ts:1`)
- `MAX_CACHE_SIZE = 4096` (`line-width-cache.ts`)
- `charCache` 상한 `16384` (`output.ts:204`)
- `MAX_VIM_COUNT = 10000` (`vim/types.ts:96`)
- 셀 레이아웃: 8 bytes/cell, `word0=charId`, `word1=styleId[31:17]|hyperlinkId[16:2]|width[1:0]` (`screen.ts:356`)
