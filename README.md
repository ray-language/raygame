# raygame

Tetris de terminal a 30 fps, escrito en [raylang](https://github.com/roberto-ayala/raylang). Es la app de **latencia dura**: bucle de juego con reloj de frame + reloj de gravedad independientes, input sin bloqueo (`io.read_timeout` hasta el próximo frame), redibujado diferencial por línea (mover una pieza repinta ≤ 4 líneas — hay test), colores ANSI por pieza, niveles que aceleran, y high score persistente.

```text
$ raygame            # ←→ mover · ↑ rotar · ↓ bajar · espacio soltar · p pausa · r reiniciar · q salir
$ raygame --bench    # los números de latencia (ver abajo)
```

## Los números que esta app vino a medir

`raygame --bench` en Apple Silicon:

| Métrica | Nativo | VM |
|---|---|---|
| Frame completo (lógica + layout + diff), 1000 frames | **21 µs/frame** | 92 µs/frame |
| `time.sleep(33)` × 60 — media real | **39 ms** | 40 ms |
| `time.sleep(33)` × 60 — peor caso | **43 ms** | 44 ms |

Lectura: el render jamás es el cuello (0.06% del presupuesto de 33 ms), pero
**`time.sleep` se pasa sistemáticamente +6–10 ms en ambos motores** — dormir
el presupuesto entero daría ~25 fps reales, no 30. El bucle de raygame lo
compensa planificando por instante absoluto (`next_frame += 33` con clamp),
así que el juego va suave; el dato queda para IDEAS (§72): la precisión del
timer del scheduler no alcanza para *frame pacing* ingenuo.

## Cómo está hecho

- **Lógica pura y sin reloj** (`src/tetris.ray`): 7 piezas × 4 rotaciones,
  colisión, lock, line clear con colapso, puntuación 100/300/500/800×nivel,
  gravedad por nivel (800 ms → 100 ms). Determinista con `random.seed` —
  los 7 tests la ejercitan sin terminal ni tiempo.
- **Dos relojes en un bucle**: el frame (33 ms, repinta si algo cambió) y la
  gravedad (según nivel); `io.read_timeout(64, hasta_el_frame)` es la espera
  única — la disciplina de raytop llevada a cadencia de juego.
- Frames de líneas fijas + diff absoluto (`ESC[n;1H` + `ESC[2K`), colores
  dentro de la línea. Verificado bajo pty real: entra/sale limpio del
  alt-screen y responde a las teclas.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Tetris completo: 7 piezas, rotación, line clear, niveles, hard drop | ✅ |
| 30 fps con input sin bloqueo + diff mínimo (≤4 líneas por movimiento) | ✅ |
| Pausa, reinicio, high score persistente | ✅ |
| `--bench` (coste de frame + jitter de sleep) | ✅ |
| Binario nativo (jugado bajo pty) | ✅ |
| Tests (reglas puras + shape del frame + diff) | ✅ 7 |
| Wall kicks (SRS), ghost piece, sonido | 📋 v2 |

## Hallazgos de dogfood

Anotados en `raylang/IDEAS.md` §72:

1. **`time.sleep` se pasa +6–10 ms consistentemente** (ambos motores): la
   predicción del catálogo ("precisión de sleep, jitter de frame") con número.
   Para juegos/pacing hace falta o un sleep más fino o documentar que el
   patrón correcto es reloj absoluto + `read_timeout` (que sí funciona).
2. El patrón tecla-o-plazo aguanta 30 fps sin despeinarse (0.02–0.09 ms de
   frame): la duda del catálogo queda cerrada — el terminal nunca es el
   cuello, el timer sí puede serlo.

## Desarrollo

```sh
ray test
ray run src/main.ray --bench
ray build --native src/main.ray -o raygame --release
```

Estructura: `src/main.ray` · `tetris.ray` (reglas puras) · `screen.ray`
(frame + diff) · `app.ray` (bucle + bench).
