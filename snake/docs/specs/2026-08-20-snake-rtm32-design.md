# Snake para RTM32/STX4 — diseño

Fecha: 2026-08-20

## Contexto

Juego Snake corriendo nativo en el simulador STX4/RTM32 de este repo.
Se ensambla con `rtm32.asm-1.1.0/` a formato `.mdbg`, se carga
en el simulador con `load <f>.mdbg exact` por el debugger telnet (puerto
4444) y se juega/observa por la consola UART (puerto 5555).

## Hallazgos empíricos que fundamentan el diseño

Verificados en esta sesión (ver investigación previa):

- **Input de teclado es posible**, leyendo con `LBU`/`LHU` desde la misma
  dirección de escritura de la UART (`0xFFFFFF00`, truco
  `addi $t2, $zero, -0x100`). El registro es **single-shot**: lee 0 salvo
  en el/los poll(s) inmediatamente después de llegar un byte nuevo
  (sobrevive al menos ~12ms de ventana de polling, no queda latcheado
  indefinidamente).
- **Requiere arreglar infraestructura primero**: `docker-entrypoint.sh`
  levanta `socat` con `-u` (unidireccional, PTY→TCP only), lo que descarta
  todo lo que el jugador tipee. Hay que sacar el `-u` para que sea
  bidireccional. Sin este fix no hay input, sin importar qué tan bien esté
  el código del juego.
- **Sin timer ni interrupciones funcionando** (`TRAP` corrompe `$vbr`,
  `CFS`/`CTS` no-op). El tempo del juego es un delay loop de espera activa.
- **Velocidad medida: ~111.000 instr/seg**, pero esa medición se hizo con
  logging TRACE verbose activo — probablemente más rápido en ejecución
  normal (sin flags de trace). La constante de delay final se calibra en
  fase de implementación, corriendo bajo el mismo entrypoint que usará el
  jugador.
- **Sin instrucción de RNG.** Se genera con un LCG a mano (`MUL` + `ADDI`).
- `examine bb <addr> <n>` no funciona tal cual está documentado
  (`Invalid option 'bb'`) — usar `dump hex <start> <end>` para inspeccionar
  memoria a mano durante debugging.
- Pipeline `.rtm` → `aarch64-macos-rtm32.asm` → `.mdbg` →
  `load exact` → `continue` reconfirmado funcionando. Gotcha: **no** correr
  `reset` antes de `load` (deja PC en `0xF0000000`, no mapeado, double
  fault) — cargar directo tras conectar.

## Decisiones de diseño

| Decisión | Elegido | Por qué |
|---|---|---|
| Bordes | Paredes matan | Snake clásico, colisión más simple (fuera de rango = game over) |
| Controles | WASD | Un byte ASCII por tecla, lectura directa y confiable con `LBU`. Flechas mandan secuencias de 3 bytes (`ESC [ A`), frágil con polling single-shot. |
| Tablero | 16×12 (192 celdas) | Entra cómodo en cualquier terminal, pocos bytes por frame, margen de sobra dado el emulador lento |
| Redibujado | Incremental | Cada tick solo escribe lo que cambió (borra cola vieja, dibuja cabeza nueva, comida si aparece). ~20-40 bytes/frame en vez de ~200+ de un full redraw. Menos costo de CPU emulada y sin parpadeo. |
| RNG comida | LCG con seed fija, avanza 1 paso por tick | Simple, determinístico (reproducible para debug), sigue viéndose random dentro de una partida. Avanza cada tick (no solo al spawnear) para no correlacionar visualmente con el momento exacto en que se come la fruta. |
| Game over | Mensaje "GAME OVER" + largo final (score) impreso + `trap 1` (halt) | Consistente con el idioma ya usado en `hola_mundo.rtm`/`criba.rtm`. Para jugar de nuevo se recarga el `.mdbg`. |

## Estado en RAM (`.data`, sin direcciones hardcodeadas — igual que `criba.rtm`)

- `grid: .pad 1 192` — occupancy grid 16×12, 1 byte/celda: `0`=vacío,
  `1`=snake, `2`=comida. Única fuente de verdad para colisión (O(1), no se
  recorre el body).
- `body: .pad 1 384` — buffer circular de coordenadas `(x,y)` de la
  serpiente, 192 slots × 2 bytes.
- `head_idx`, `tail_idx`, `len` — words en `.data`.
- `dir` — word, `0`=up `1`=down `2`=left `3`=right.
- `rng_state` — word, estado del LCG, seed inicial fija (`0x2A2A2A2A`).
- `food_x`, `food_y` — bytes.
- Stack: `$sp` inicializado en `0xFFF0` (memoria de 64K, sobra margen).

## Flujo principal

```
init:
  grid[*] = 0
  snake inicial: len=3, centro del tablero, dir=right
  marcar snake en grid, dibujar snake inicial completo (único full-redraw)
  spawn comida (LCG hasta encontrar celda libre en grid)
  dibujar comida

loop:
  tecla = LBU 0xFFFFFF00                          // poll single-shot
  si tecla in {w,a,s,d} y no es reversa 180°       // ej. venir de dir=right, ignorar 'a'
    -> dir = tecla mapeada

  nueva_cabeza = head + delta(dir)

  si nueva_cabeza fuera de [0,16)x[0,12) -> GAME_OVER
  si grid[nueva_cabeza] == 1 (snake)     -> GAME_OVER

  si grid[nueva_cabeza] == 2 (comida):
      len++                                        // no se borra la cola: crece
      spawn nueva comida (LCG + retry si celda ocupada)
  sino:
      celda_vieja = body[tail_idx]
      grid[celda_vieja] = 0
      borrar celda_vieja en pantalla (ANSI move-cursor + ' ')
      tail_idx = (tail_idx + 1) % 192

  body[head_idx] = nueva_cabeza
  grid[nueva_cabeza] = 1
  head_idx = (head_idx + 1) % 192
  dibujar nueva_cabeza en pantalla (ANSI move-cursor + '#')

  rng_state = LCG_step(rng_state)                  // 1 paso por tick, siempre

  delay(N instrucciones)                           // constante a calibrar
  j loop

GAME_OVER:
  ANSI move-cursor a fila de mensaje
  imprimir "GAME OVER" (rutina tipo imprimir_asciiz de hola_mundo.rtm)
  imprimir len final como score (rutina print32u de criba.rtm, reusada tal cual)
  trap 1
```

## Rutinas reutilizadas de los ejemplos existentes

- `imprimir_asciiz` (de `hola_mundo.rtm`) — para "GAME OVER" y cualquier
  string fija.
- `print32u` (de `criba.rtm`) — para imprimir el score final.
- Ambas se copian/adaptan tal cual, ya están verificadas funcionando.

## Fuera de alcance (YAGNI)

- Pausa/resume, niveles de dificultad, colores, sonido (no hay).
- Persistencia de high score entre partidas.
- Soporte de flechas de teclado (ver tabla de decisiones — riesgo de
  pérdida de bytes con polling single-shot).
- Reinicio automático tras game over (recomendado explícitamente en contra
  durante el brainstorming: recargar el `.mdbg` es aceptable).

## Prerrequisito de infraestructura (bloqueante, va antes que el código del juego)

Editar `docker-entrypoint.sh`: sacar el flag `-u` del comando `socat` que
bridgea el PTY de la UART a `TCP-LISTEN:5555`, para que quede bidireccional
y el input del jugador llegue a la CPU. Verificar con un probe corto (igual
al usado en la investigación) antes de invertir tiempo en el juego
completo.

## Calibración de tempo (paso explícito de implementación, no placeholder)

1. Medir instr/seg real bajo el entrypoint sin flags de trace (loop de N
   iteraciones conocido, timestamps de wall-clock alrededor de
   `continue`→`trap`).
2. Elegir constante de delay loop para un tick de ~150-200ms en base a esa
   medición.
3. Ajustar a mano jugando — si se siente muy rápido/lento, tocar la
   constante.
