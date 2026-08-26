# Snake — borde visual del tablero

**Fecha:** 2026-08-26
**Contexto:** `rtm32.asm-1.1.0/snake.rtm` ya funciona end-to-end (ver
`2026-08-20-snake-rtm32-design.md`). Falta indicar visualmente el límite del
tablero — hoy el jugador solo se entera de que chocó cuando aparece
`GAME OVER`, sin ninguna marca en pantalla de dónde termina el área jugable.

## Objetivo

Dibujar un borde rectangular alrededor del tablero de 16×12, visible desde
que arranca cada partida (incluyendo auto-restart tras game over).

## Diseño

### 1. Offset del playfield en `draw_cell`

`draw_cell` mapea coords de juego (x,y, 0-based) a coords de terminal
(1-based) sumando 1 a cada eje. Se cambia esa suma a 2. El playfield pasa de
ocupar filas 1-12 / columnas 1-16 de terminal a filas 2-13 / columnas 2-17,
dejando un anillo libre alrededor (fila 1, fila 14, columna 1, columna 18)
para el borde.

La lógica de grid y colisión (`game_loop`, límites 0-15 / 0-11) no cambia —
solo cambia dónde se dibuja cada celda en pantalla.

### 2. Rutina `draw_border`

Nueva subrutina, llamada una vez al principio de `game_start` (antes de
dibujar la serpiente inicial), usando `draw_cell` con coordenadas de juego
fuera de rango (que el offset +2 lleva a coords válidas de terminal ≥1):

- Esquinas `+` en (-1,-1), (16,-1), (-1,12), (16,12)
- Borde superior/inferior `-`: y=-1 e y=12, x=0..15
- Bordes laterales `|`: x=-1 y x=16, y=0..11

Se redibuja en cada `game_start`, incluido el auto-restart post game-over
(no hay que tocar `game_over`/`wait_key`/`clear_screen` para eso — el flujo
ya vuelve a `game_start` completo).

### 3. Reposicionar mensaje de `GAME OVER`

`game_over` hoy hardcodea fila 14 columna 1 para el texto (`addi $a0, $zero,
14` seguido del código ANSI de cursor). Con el shift, fila 14 pasa a ser el
borde inferior — el mensaje lo pisaría. Se cambia esa constante a fila 16
(deja fila 15 en blanco como separador). El mensaje de replay sigue
imprimiéndose secuencialmente después vía `\r\n`, sin coordenada propia — no
necesita cambio.

### 4. Fuera de alcance

- `show_menu` (pantalla de título) no lleva borde — solo aparece durante el
  juego.
- No se toca el tamaño del tablero (sigue 16×12 celdas jugables) ni la
  lógica de colisión/spawn de comida.

## Testing

Manual, vía el mismo flujo de `README.md` (`nc` en modo raw): confirmar que
el borde aparece completo al iniciar partida, se redibuja igual tras
game-over/restart, que la serpiente y la comida siguen apareciendo en la
posición correcta dentro del borde (no corridas), y que el mensaje
`GAME OVER` no se solapa con el borde inferior.
