# raygame

Tetris de terminal a 30 fps, escrito en [raylang](https://github.com/ray-language/raylang). Es la app de **latencia dura**: bucle de juego con reloj de frame + reloj de gravedad independientes, input sin bloqueo (`io.read_timeout` hasta el próximo frame), redibujado diferencial por línea (mover una pieza repinta ≤ 4 líneas — hay test), colores ANSI por pieza, niveles que aceleran, y high score persistente.

```text
$ raygame            # ←→ mover · ↑ rotar · ↓ bajar · espacio soltar · p pausa · r reiniciar · q salir
$ raygame --bench    # los números de latencia (ver abajo)
```

## Los números que esta app vino a medir

`raygame --bench` en Apple Silicon:

| Métrica | Nativo | VM |
|---|---|---|
| Frame completo (lógica + layout + diff), 1000 frames | **21 µs/frame** | 92 µs/frame |
| `time.sleep(33)` × 60 — media real | **33 ms** | 33 ms |
| `time.sleep(33)` × 60 — peor caso | **35 ms** | 35 ms |

Lectura: el render jamás es el cuello (0.06% del presupuesto de 33 ms), y
desde raylang M119 **`time.sleep` es preciso** (~1 ms de desvío; este mismo
benchmark medía 39–43 ms cuando el hallazgo se anotó — la causa era el
`nanosleep` de macOS, y el runtime pasó a `poll(2)`). El bucle sigue
planificando por instante absoluto (`next_frame += 33`), que es el patrón
sin deriva que el MANUAL de raylang documenta.

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

1. **[RESUELTO — raylang M119]** `time.sleep` se pasa +6–10 ms: el sleep es
   preciso vía `poll(2)` — este mismo `--bench` pasó de 39/43 ms a 33/35 ms
   sin tocar la app. El reloj absoluto sigue siendo el patrón para pacing
   sin deriva (ahora documentado en el MANUAL de raylang).
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
