# Capítulo 12: UI de Terminal y Motor de Renderizado

## 12.1 Descripción General y Posicionamiento

Claude Code es un asistente de codificación de IA interactivo que se ejecuta en el terminal; su capa de UI necesita lograr una experiencia de interacción cercana a la GUI dentro de un entorno de terminal de texto puro: renderizado de mensajes en streaming, edición con atajos de Vim, selección de texto con ratón, búsqueda con resaltado y superposición de múltiples ventanas emergentes. Estos requisitos superan con creces los límites de las herramientas CLI tradicionales.

La capa de UI de terminal de Claude Code se asienta sobre tres decisiones tecnológicas centrales:

1. **Renderizador Ink personalizado** (`src/ink/`): profunda adaptación del framework React + Ink, introduciendo un modelo de pantalla con doble búfer, motor de diff ANSI, aceleración de desplazamiento por hardware y posicionamiento físico del cursor sensible a IME.
2. **Árbol de componentes declarativo** (`src/screens/REPL.tsx`, `src/components/`): describe la estructura de UI con componentes React; el motor de renderizado traduce el DOM virtual en una matriz de caracteres de terminal.
3. **Modo Vim completo** (`src/vim/`): implementación independiente de máquina de estados, cubriendo los modos NORMAL/INSERT, el conjunto completo de operators/motions/textObjects, así como dot-repeat y registros.

El ritmo de todo el pipeline de renderizado está controlado por `FRAME_INTERVAL_MS = 16` (`src/ink/constants.ts:1`), con una tasa de fotogramas teórica de 62,5 fps, alineada con la tasa de refresco del monitor.

---

## 12.2 Fundamentos Teóricos

### Principios Subyacentes del Renderizado de Terminal

El renderizado de terminal es esencialmente escribir una secuencia de bytes en un descriptor de archivo; el emulador de terminal es responsable de parsear y actualizar la pantalla. Los mecanismos clave incluyen:

- **Códigos de escape ANSI**: secuencias de control que comienzan con `\x1b[`, controlando la posición del cursor (CSI CUP), el color (SGR), la región de desplazamiento (DECSTBM), el seguimiento del ratón (modos privados DEC), etc. El directorio `src/ink/termio/` de Claude Code encapsula estas secuencias como constantes con nombre (`CURSOR_HOME`, `ENTER_ALT_SCREEN`, `ENABLE_MOUSE_TRACKING`, etc.).
- **termios / modo raw**: en modo raw, cada pulsación de tecla llega inmediatamente al programa sin pasar por el buffer de línea ni el eco. Claude Code toma el control de stdin al inicio mediante `setRawMode(true)` y lo restaura al salir con `Ink.detachForShutdown()` (`ink.tsx:942-950`).
- **Alt Screen (?1049h/?1049l)**: buffer de pantalla alternativa; al entrar no contamina el historial de desplazamiento de la pantalla principal; al salir restaura el estado original. En modo pantalla completa (`isFullscreenEnvEnabled()`), el REPL se envuelve en el componente `<AlternateScreen>`.

### Renderizado con Doble Búfer

Claude Code usa dos búferes de fotogramas (front/back, `frontFrame` / `backFrame`) para implementar el doble búfer (`ink.tsx:196-197`):

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

En cada fotograma:
1. El Renderer escribe el DOM virtual de React en `backFrame` (nuevo fotograma)
2. El motor de diff LogUpdate compara `frontFrame` (fotograma antiguo) con `backFrame`, generando la secuencia mínima de parches
3. La secuencia de parches se escribe en el terminal; `frontFrame` ↔ `backFrame` intercambian (`ink.tsx:594`)

Así el terminal solo recibe los caracteres que realmente han cambiado, en lugar de redibujar todo en cada fotograma.

### Aplicación de React en un Entorno sin DOM (Custom Renderer)

La interfaz del renderizador de React (`react-reconciler`) permite cualquier entorno anfitrión. Claude Code implementa un Custom Renderer orientado al terminal en `src/ink/reconciler.ts` (512 líneas):

- El tipo de elemento anfitrión es el `dom.DOMElement` personalizado (`src/ink/dom.ts`)
- El motor de layout usa Yoga (la implementación Flexbox de Facebook), impulsado en el callback `onComputeLayout` (`ink.tsx:239-258`)
- El modo React Concurrent Root (`ConcurrentRoot`) permite que el renderizado sea interrumpible, evitando bloquear el hilo principal durante períodos prolongados

---

## 12.3 Renderizador Ink Personalizado

### La Adaptación Personalizada de Claude Code a Ink

`src/ink/ink.tsx` (1.722 líneas) es el controlador central de toda la capa de UI, definiendo la clase `Ink`. No es el paquete Ink original; es una versión profundamente adaptada, con las siguientes mejoras centrales:

**Gestión de doble búfer + fotogramas**: el Ink original emite cadenas directamente; la versión personalizada introduce objetos `Frame` y la matriz `Screen`.

**Monitoreo de rendimiento**: cada fotograma registra la duración de las fases renderer, diff, optimize y write, así como las métricas visited/measured/cacheHits del layout de Yoga; se expone a través del callback `onFrame` (`ink.tsx:772-788`):

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

**Traspaso a TUI externa**: proporciona los métodos `enterAlternateScreen()` / `exitAlternateScreen()`, que pausan el control de Ink, cierran el protocolo de teclado kitty y permiten que editores externos (vim, nano) funcionen normalmente; al volver, se restaura completamente (`ink.tsx:357-419`).

**Autorreparación ante SIGCONT**: `handleResume` gestiona la reanudación del proceso desde pausa (comando fg); en pantalla principal se reinicia el búfer; en pantalla alternativa se vuelve a entrar y se limpia la pantalla (`ink.tsx:280-301`).

### Mecanismo de Actualización con Doble Búfer + Diff

`LogUpdate.render()` en `src/ink/log-update.ts` es el núcleo del motor de diff. Recibe `prev: Frame` y `next: Frame` y devuelve un conjunto de operaciones de parche `Diff`:

```
Tipos de parche Diff:
- { type: 'stdout', content: string }    — escribe directamente en el terminal
- { type: 'cursorMove', x, y }           — movimiento relativo del cursor
- { type: 'cursorTo', x, y }             — posicionamiento absoluto del cursor
- { type: 'styleStr', str }              — cambio de estilo SGR
- { type: 'hyperlink', uri }             — hipervínculo OSC 8
- { type: 'cursorHide' | 'cursorShow' }  — visibilidad del cursor
- { type: 'clear', count }               — borrar caracteres
- { type: 'clearTerminal', reason }      — redibujado completo (provoca parpadeo)
```

El motor de diff se optimiza en `src/ink/optimizer.ts` con un único pase (`optimize(diff)`): fusiona `cursorMove` adyacentes, colapsa `cursorTo` consecutivos, concatena `styleStr` adyacentes, cancela pares cursor hide/show, reduciendo el volumen real de escritura.

### Búfer de Pantalla Int32Array

`src/ink/screen.ts` define el tipo `Screen`, usando `Int32Array` en lugar de arrays de objetos para almacenar las celdas de la pantalla, eliminando la presión sobre el GC (`screen.ts:356-370`):

```typescript
// screen.ts:356-370 (comentario)
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen` asigna un `ArrayBuffer` y monta dos vistas (`Int32Array` y `BigInt64Array`); esta última se usa para el zeroing masivo en `resetScreen` (`screen.ts:472-490`):

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes por celda
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

Los caracteres y estilos se internalizan a través de **objetos Pool compartidos** (`CharPool`, `StylePool`, `HyperlinkPool`), compartiendo IDs entre Screens, permitiendo que `blitRegion` copie directamente IDs enteros sin necesidad de reinternalizar, y que `diffEach` use comparaciones de enteros en lugar de comparaciones de cadenas (`screen.ts:17-31`).

La codificación de bit-0 de `StylePool.intern()` codifica la visibilidad (`screen.ts:148-161`): los IDs impares indican que el estilo es visible en espacios en blanco (color de fondo, color invertido, subrayado), permitiendo que el renderizador omita espacios en blanco invisibles con una única máscara de bits, evitando escrituras innecesarias.

### Desplazamiento Hardware (DECSTBM)

`log-update.ts` implementa la optimización de desplazamiento hardware DECSTBM (`log-update.ts:149-185`):

Cuando `scrollTop` del `ScrollBox` cambia en modo alt-screen, en lugar de reescribir toda la región de desplazamiento, envía `CSI top;bot r` (establece la región de desplazamiento) + `CSI n S` (desplaza n líneas hacia arriba); el hardware del terminal completa el desplazamiento del contenido de forma acelerada, y solo es necesario rellenar las nuevas líneas que aparecen. Simultáneamente se ejecuta `shiftRows()` en `prev.screen` para simular el desplazamiento, haciendo que el bucle de diff subsiguiente encuentre naturalmente las líneas diferentes en lugar de reescribir todo.

Esta optimización solo se activa cuando `decstbmSafe=true`: cuando el terminal externo no soporta DEC 2026 (operaciones atómicas BSU/ESU), terminales como tmux renderizan estados intermedios (la pantalla después del desplazamiento pero antes de rellenar las nuevas líneas), produciendo saltos visibles; en ese caso se recurre al diff completo (`log-update.ts:158-167`).

### Caché de 16.384 Caracteres

La clase `Output` en `src/ink/output.ts` (motor interno de medición y salida de caracteres del renderizador) mantiene un `charCache` con el mapeo de caracteres a anchuras medidas. Cuando la caché supera las 16.384 entradas, se limpia directamente y se reconstruye (`output.ts:204`):

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

De forma similar, `src/ink/line-width-cache.ts` mantiene una caché de anchuras de línea (máximo 4.096 entradas) para reutilizar las llamadas `stringWidth` de las numerosas líneas sin cambios durante la salida en streaming (`line-width-cache.ts`):

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### Estrategias de Optimización de Rendimiento

| Estrategia | Ubicación | Efecto |
|---|---|---|
| Retraso de microtask en la planificación de renderizado | `ink.tsx:212-216` | `scheduleRender` usa `queueMicrotask`, garantizando que `useLayoutEffect` complete antes del renderizado, evitando el retraso de un fotograma en la posición del cursor |
| Throttle a 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`, tasa de fotogramas estable |
| Timer de drain a cuatro veces la tasa de fotogramas | `ink.tsx:757-758` | El drain continuo del scroll usa `FRAME_INTERVAL_MS >> 2` (~4ms), maximizando la fluidez del desplazamiento |
| Reset periódico del Pool | `ink.tsx:600-603` | Reset del char/hyperlink pool cada 5 minutos, previniendo el crecimiento de memoria en sesiones largas |
| Rectángulo de daño (damage rectangle) | `screen.ts:382` | Solo se hace diff del rectángulo realmente escrito en cada fotograma, no de toda la pantalla |
| Anclaje del cursor en alt-screen | `ink.tsx:578-591` | Antes del diff de cada fotograma se usa `ALT_SCREEN_ANCHOR_CURSOR` para resetear el cursor virtual a (0,0), previniendo la deriva del cursor en terminales como tmux |

---

## 12.4 Interfaz Principal REPL

### Estructura de Componentes de REPL.tsx

`src/screens/REPL.tsx` (5.005 líneas) es el componente de interfaz principal de Claude Code y también el archivo individual más grande de todo el proyecto. Sus imports de nivel superior superan las 150 líneas, cubriendo gestión de sesiones, sistema de permisos, conexiones MCP, multi-agente Swarm, integración de voz y prácticamente todos los módulos funcionales.

Estructura del árbol de renderizado del REPL (simplificada):

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen (modo pantalla completa)
    │   └── Box (layout principal)
    │       ├── AnimatedTerminalTitle (título de pestaña del terminal)
    │       ├── FullscreenLayout
    │       │   ├── Messages (lista de mensajes)
    │       │   ├── VirtualMessageList (desplazamiento virtual)
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── Varios Callout / Survey / Dialog
    │       └── PromptInput (área de entrada)
    └── DevBar (barra de depuración solo para ant)
```

El modo pantalla completa se determina por una rama condicional que decide si envolver con `<AlternateScreen>` (`REPL.tsx:4998-5001`):

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### Flujo de Renderizado de Mensajes

El estado de los mensajes se mantiene mediante `useState<MessageType[]>` de React; `handleMessageFromStream` procesa los eventos SSE de streaming para añadir/actualizar mensajes. La lista de mensajes se renderiza en los componentes `<Messages>` o `<VirtualMessageList>`; este último usa desplazamiento virtual para renderizar solo el área visible, evitando una gran cantidad de nodos DOM en conversaciones largas.

Hay múltiples optimizaciones en el REPL para evitar rerenderizados innecesarios: `AnimatedTerminalTitle` se extrae como nodo hoja independiente, evitando que el tick de animación de 960ms rerenderice todo el árbol del REPL (el comentario en `REPL.tsx` explica: `Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`).

`useDeferredValue(searchQuery)` se usa para retrasar el cálculo de búsqueda; `React.memo` cachea en el subárbol `PromptInput` para evitar que los frecuentes rerenderizados al escribir se propaguen hacia arriba.

### Pipeline de Procesamiento de Entrada

La entrada del usuario entra desde stdin en modo raw al `parse-keypress.ts` de Ink, se parsea como objetos `ParsedKey`, se distribuye mediante FocusManager a los callbacks `useInput` del nodo con foco actual. `GlobalKeybindingHandlers` y `CommandKeybindingHandlers` proporcionan un mecanismo de registro de atajos globales con mayor prioridad que los `useInput` a nivel de componente.

La entrada finalmente llega al componente `PromptInput`, que según el modo actual (Vim/Estándar) la enruta a `VimTextInput` o `TextInput`.

---

## 12.5 Modo Vim

### Arquitectura de Implementación del Modo Vim

El directorio `src/vim/` (5 archivos, ~1.513 líneas) implementa una capa completa de atajos Vim. El diseño arquitectónico es extremadamente conciso: **los tipos son documentación**; todas las transiciones de máquina de estados están codificadas en tipos TypeScript; la comprobación de exhaustividad de TypeScript garantiza que todos los estados sean manejados.

División de responsabilidades por archivo:
- `types.ts`: definiciones de tipos de máquina de estados + conjuntos de constantes
- `transitions.ts`: funciones de transición (punto de entrada principal `transition()`)
- `motions.ts`: funciones puras de movimiento del cursor (`resolveMotion()`)
- `operators.ts`: funciones de ejecución de operaciones (delete/change/yank, etc.)
- `textObjects.ts`: cálculo de límites de objetos de texto

### Operaciones Soportadas (motions, operators, textObjects)

**Operators (operadores)**: `d` (delete), `c` (change), `y` (yank) — definidos en `types.ts:OPERATORS`.

**Simple Motions** (`types.ts:SIMPLE_MOTIONS`):
- Movimiento básico: `h`/`l`/`j`/`k`
- Movimiento de palabra: `w`/`b`/`e`/`W`/`B`/`E`
- Posición en línea: `0`/`^`/`$`

**Find Motions**: `f`/`F`/`t`/`T` (búsqueda de caracteres), `;`/`,` para repetir/invertir.

**Text Objects** (`types.ts:TEXT_OBJ_TYPES`): `w`/`W` (palabra), `"`/`'`/`` ` `` (comillas), `(`/`)`/`b` (paréntesis), `[`/`]` (corchetes), `{`/`}`/`B` (llaves), `<`/`>` (ángulos).

**Operaciones de línea**: `dd`/`cc`/`yy`, `D`/`C`/`Y`, `o`/`O` (abrir línea), `J` (join), `>>`/`<<` (indentación).

**Otros comandos**: `r` (replace char), `~` (toggle case), `x` (delete char), `p`/`P` (paste), `u` (undo), `G`/`gg` (saltar a línea), `.` (dot-repeat).

Manejo especial: `cw`/`cW` cambia hasta el final de la palabra en lugar del inicio de la siguiente (comportamiento clásico de vim, `operators.ts:91-102`); los chips de referencia de imagen se reconocen como unidades atómicas, y el movimiento de palabras no se detiene en medio de un chip (`operators.ts:333-337`).

### Máquina de Estados y Cambio de Modo

`VimState` es el tipo de nivel superior (`types.ts:52-55`):

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

El modo INSERT registra el texto introducido para dot-repeat; el modo NORMAL contiene la submáquina de estados `CommandState`.

`CommandState` tiene 11 estados (`types.ts:62-75`), cubriendo idle, acumulación de count, operador pendiente, operatorCount, operatorFind, operatorTextObj, find, prefijo g, operatorG, replace e indent. El diagrama de estados está documentado en formato ASCII art al inicio de `types.ts` (`types.ts:1-26`):

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

La función `transition()` (`transitions.ts:55-73`) despacha al tipo `CommandState` actual hacia funciones manejadoras especializadas; cada función devuelve `{ next?: CommandState; execute?: () => void }`. Si `execute` existe, se ejecuta inmediatamente; si `next` existe, avanza el estado; si ninguno existe, vuelve a idle.

`PersistentState` (`types.ts:80-87`) persiste entre comandos: `lastChange` (grabación para dot-repeat), `lastFind` (repetición con `;`/`,`), `register` (portapapeles), `registerIsLinewise` (para juzgar el pegado a nivel de línea).

La multiplicación del count se implementa en `fromOperatorCount` (`transitions.ts`): `effectiveCount = state.count * motionCount`, soportando `3d2w` = eliminar 6 palabras. El count tiene un límite máximo de `MAX_VIM_COUNT = 10000` (`types.ts:96`), previniendo entradas maliciosas.

---

## 12.6 Análisis de Componentes Principales

### Componente de Entrada PromptInput

`src/components/PromptInput/PromptInput.tsx` (2.338 líneas) es uno de los componentes con mayor densidad funcional de Claude Code, asumiendo casi toda la lógica de interacción relacionada con la entrada.

**Escala de Props**: el tipo `Props` abarca aproximadamente 60 líneas, incluyendo valor de entrada, modo, estado de Vim, pila de stash, callbacks de submit, contexto de permisos, clientes MCP, selección de IDE, integración de voz, etc.

**Inyección de entrada externa**: `insertTextRef` expone la interfaz `{ insert, setInputWithCursor, cursorOffset }`, para que el reconocimiento de voz (STT) pueda insertar texto en la posición del cursor en lugar de reemplazar toda la entrada (`PromptInput.tsx:180-200`).

**Gestión del cursor**: rastrea independientemente el estado `cursorOffset`, distinguiendo los cambios internos (envueltos por `trackAndSetInput` en `onInputChange`) de las inyecciones externas (entrada de voz); los cambios externos mueven el cursor al final (`PromptInput.tsx:164-176`).

**Enrutamiento de modo**: según `isVimModeEnabled()` renderiza `<VimTextInput>` o `<TextInput>`; ambos comparten la interfaz `BaseTextInputProps`.

**Constantes del layout inferior** (`PromptInput.tsx`):
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
La altura máxima del slot de entrada inferior es el 50%; se reservan 5 líneas para el footer, el borde y los indicadores de estado.

### Interfaz de Configuración

`src/components/Settings/Config.tsx` (1.821 líneas) implementa la interfaz de configuración del comando `/config`.

**Navegación por búsqueda**: Config entra por defecto en modo búsqueda (`isSearchMode = true`); el usuario puede escribir directamente para filtrar elementos de configuración sin necesidad de desplazarse con teclas de dirección. `maxVisible = Math.max(5, paneCap - 10)` calcula dinámicamente el número de elementos visibles según la altura del terminal.

**Categorías de configuración**: el array `settingsItems: Setting[]` define uniformemente todos los ítems de configuración; cada ítem es de tipo `boolean` (interruptor), `enum` (selección enumerada) o `managedEnum` (componente personalizado de gestión).

**Seguimiento de cambios y restauración**: el estado `changes` registra todas las modificaciones; el ref `isDirty` se usa para la tecla Escape para determinar si es necesario escribir en disco o restaurar, evitando escrituras innecesarias al abrir y cerrar (`Config.tsx`).

**Sistema de submenús**: el estado `showSubmenu` controla la expansión de submenús como Theme, Model, TeammateModel, ExternalIncludes, OutputStyle, Language, EnableAutoUpdates; se comunica con el componente padre Settings mediante `setTabsHidden` para ocultar la cabecera de tabs.

**Arquitectura de configuración de múltiples fuentes**: las fuentes de configuración se dividen en `localSettings` (nivel de proyecto) y `userSettings` (nivel de usuario), leyendo y escribiendo mediante `getSettingsForSource`/`updateSettingsForSource`; al restaurar con Escape se usa una instantánea inicial para restaurar fuente por fuente, en lugar de leer la vista global fusionada (`Config.tsx`, los comentarios indican que esto es para soportar la semántica de eliminación de clave con `undefined`).

### Selector de Registros

`src/components/LogSelector.tsx` (1.574 líneas) implementa la interfaz de selección del historial de sesiones (comando `/resume`).

**Visualización en árbol**: muestra las sesiones bifurcadas (fork) en forma de árbol padre-hijo mediante `<TreeSelect<LogTreeNode>>`; la expansión/colapso se gestiona con `expandedGroupSessionIds`.

**Tres modos de vista**: `viewMode` puede ser `"list"` (lista por defecto), `"search"` (búsqueda) o `"preview"` (previsualización de sesión).

**Búsqueda difusa con Fuse.js**: usa matching difuso con `FUSE_THRESHOLD = 0.3`, combinado con `DATE_TIE_THRESHOLD_MS = 60 * 1000` (los resultados dentro de 1 minuto se ordenan por relevancia; de lo contrario, por tiempo).

**Previsualización de fragmentos de resultados de búsqueda**: `extractSnippet()` toma `SNIPPET_CONTEXT_CHARS = 50` caracteres antes y después de la posición de coincidencia, renderizando el contexto con `chalk.dim` y el término de búsqueda con color resaltado (`LogSelector.tsx`).

**Constantes de búsqueda profunda**:
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
(Actualmente `isDeepSearchEnabled = false`, función pendiente de activación)

**Optimización con React Compiler**: `import { c as _c } from "react/compiler-runtime"` al inicio del archivo indica que ha sido compilado por React Compiler; `_c(247)` asigna 247 slots de caché memo, y numerosas condiciones de memorización evitan el rerenderizado de subárboles.

---

## 12.7 Análisis de Decisiones de Diseño

### Por Qué un Renderizador Ink Personalizado en Lugar de Usar el Original

El Ink original serializa el árbol de React a una cadena y usa `log-update` para escritura incremental, un diseño orientado a herramientas CLI simples. Los requisitos de Claude Code superan los límites de este modelo:

1. **Selección con ratón**: requiere mapeo de coordenadas de celda a nivel de pantalla (hit-test); el modelo de cadenas no puede soportarlo.
2. **Búsqueda con resaltado**: requiere superponer celdas con colores invertidos sobre el fotograma ya renderizado; es imprescindible operar sobre una matriz de celdas.
3. **Desplazamiento hardware DECSTBM**: requiere consistencia entre los dos fotogramas anterior/siguiente en la región de desplazamiento; el diff de cadenas no puede simular el efecto de desplazamiento hardware del terminal.
4. **Posicionamiento del cursor para IME / a11y**: necesita conocer con precisión las coordenadas físicas en pantalla del cursor de entrada, para detener el cursor físico del terminal en la posición correcta (el texto preedit del método de entrada CJK se renderiza en la posición del cursor físico).
5. **Atomicidad de Alt-screen (BSU/ESU)**: necesita envolver erase + paint dentro de `\x1b[?2026h`/`\x1b[?2026l` para evitar que terminales como tmux rendericen estados intermedios.

### Consideraciones de Rendimiento del Doble Búfer

La salida al terminal es una operación intensiva en E/S; la llamada al sistema `write()` tiene un overhead fijo. La estrategia de diff con doble búfer reduce el volumen de escritura por fotograma de O(tamaño de pantalla) a O(área de cambios). Los comentarios en el código fuente muestran: los fotogramas solo-drain (solo DECSTBM scroll, sin commit de React) producen ~10 parches y ~200 bytes de salida (`ink.tsx:756`), mientras que un redibujado completo requeriría miles de bytes.

El flag `prevFrameContaminated` es la red de seguridad de corrección: cuando operaciones como el resaltado de selección, `resetFramesForAltScreen()` o `forceRedraw()` modifican el contenido del frontFrame, el siguiente fotograma debe hacer un diff completo en lugar de depender de la optimización de blit incremental (`ink.tsx:743`).

### Motivación para el Modo Vim

El núcleo de usuarios de Claude Code son usuarios intensivos de Vim: para ellos, usar atajos Vim en el área de prompts no es un adorno sino un requisito básico. La implementación de máquina de estados pura (sin dependencias, sin efectos secundarios) hace que la lógica Vim sea completamente testeable unitariamente, desacoplada de la capa de renderizado del terminal.

El comentario al inicio de `types.ts` revela la filosofía de diseño: **los tipos son documentación**. El union type de `CommandState` describe la máquina de estados de manera más precisa que cualquier comentario; la comprobación de exhaustividad de switch de TypeScript garantiza que cada estado sea manejado, y la omisión de un nuevo estado provoca un error en tiempo de compilación (`types.ts:1`).

---

## 12.8 Patrones Transferibles

Los siguientes patrones de diseño pueden trasladarse a otras aplicaciones de terminal o frameworks de UI:

**1. Búfer de Pantalla con Pool Compartido + ID Entero**
Almacenar los IDs enteros de caracteres/estilos usando `Int32Array` en lugar de cadenas u objetos. El Pool compartido entre fotogramas reduce el diff a comparaciones de enteros, eliminando completamente la presión sobre el GC. Aplicable a cualquier escenario de diff frecuente sobre una matriz rectangular.

**2. Los Tipos como Máquina de Estados**
Definir la máquina de estados con TypeScript union types, donde cada estado lleva exactamente los datos que necesita; las funciones de transición usan switch exhaustivo. El valor de retorno de doble clave `{next?, execute?}` de `transition()` + `TransitionResult` es extremadamente conciso y puede trasladarse directamente a otros escenarios de manejo de atajos.

**3. Diff Incremental Impulsado por Rectángulo de Daño**
Al renderizar, registrar el rectángulo realmente escrito (`damage`); el diff solo escanea esa área. Aplicable a cualquier sistema de búfer de fotogramas, no solo terminales; también se puede usar en UIs de juegos y pantallas embebidas.

**4. Doble Búfer + Red de Seguridad prevFrameContaminated**
Cuando ciertas operaciones no siguen la ruta de renderizado normal (modificando directamente el fotograma frontal), usar un flag booleano para forzar el diff completo en el siguiente fotograma, sin violar el invariante del doble búfer. Más eficiente que redibujar siempre todo; más correcto que permitir datos sucios.

**5. Retraso de Microtask en Renderizado**
El patrón `scheduleRender = throttle(() => queueMicrotask(onRender), 16)` garantiza que los efectos de layout de React se committen completamente antes del renderizado, evitando el retraso de un fotograma en la posición del cursor. También aplicable en otros escenarios de React + renderizador personalizado.

**6. Caché de Anchura de Línea (optimización específica para escenarios de streaming)**
En la salida en streaming, el cálculo de `stringWidth` para numerosas líneas sin cambios es un punto caliente. La estrategia simple de Map + limpiar cuando está lleno de `line-width-cache.ts` resuelve con ~20 líneas de código una reducción de ~50x en el overhead computacional (texto original del comentario); puede trasladarse directamente a cualquier escenario de renderizado de texto en streaming.

---

## 12.9 Índice de Código Fuente

| Archivo | Líneas | Responsabilidad principal |
|---|---|---|
| `src/ink/ink.tsx` | 1.722 | Clase Ink: gestión de fotogramas, planificación de renderizado, SIGCONT, traspaso a alt-screen, selección con ratón, búsqueda con resaltado |
| `src/ink/screen.ts` | 1.300+ | Tipo Screen, layout de celdas Int32Array, CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate: motor de diff prev/next, desplazamiento hardware DECSTBM |
| `src/ink/optimizer.ts` | ~100 | Optimizador de diff en un único pase |
| `src/ink/reconciler.ts` | 512 | Implementación de Custom Renderer anfitrión para React |
| `src/ink/render-node-to-output.ts` | 1.462 | Renderizado de nodos DOM a Screen |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | Caché LRU de anchura de línea (4.096 entradas) |
| `src/ink/output.ts` | ~300 | Renderizador Output, caché de caracteres (16.384 entradas) |
| `src/vim/types.ts` | ~150 | Definiciones de tipos VimState, CommandState, PersistentState y constantes |
| `src/vim/transitions.ts` | ~350 | Punto de entrada principal `transition()`, 11 funciones de transición de estado |
| `src/vim/motions.ts` | ~80 | Función pura `resolveMotion()` para movimiento del cursor |
| `src/vim/operators.ts` | ~500 | Funciones de ejecución delete/change/yank/x/r/~/J/paste/indent/openLine |
| `src/vim/textObjects.ts` | ~220 | Búsqueda de límites para objetos de texto como palabras/comillas/paréntesis |
| `src/screens/REPL.tsx` | 5.005 | Interfaz REPL principal, gestión de sesiones, flujo de mensajes, cambio entre modo pantalla completa/normal |
| `src/components/PromptInput/PromptInput.tsx` | 2.338 | Componente de entrada, enrutamiento modo Vim/Estándar, gestión del cursor, inyección STT |
| `src/components/Settings/Config.tsx` | 1.821 | Interfaz de configuración, navegación por búsqueda, lectura/escritura de configuración de múltiples fuentes, restauración con Escape |
| `src/components/LogSelector.tsx` | 1.574 | Selección del historial de sesiones, búsqueda Fuse.js, visualización en árbol de forks |

**Referencia rápida de constantes clave**:
- `FRAME_INTERVAL_MS = 16` (`constants.ts:1`)
- `MAX_CACHE_SIZE = 4096` (`line-width-cache.ts`)
- Límite de `charCache` `16384` (`output.ts:204`)
- `MAX_VIM_COUNT = 10000` (`vim/types.ts:96`)
- Layout de celda: 8 bytes/celda, `word0=charId`, `word1=styleId[31:17]|hyperlinkId[16:2]|width[1:0]` (`screen.ts:356`)
