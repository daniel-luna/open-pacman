# SPEC 02 − Salida fluida de la casa de los fantasmas

**Estado:** Approved
**Dependencias:** SPEC 01
**Fecha:** 2026-10-07
**Objetivo:** Hacer fluida la salida de los cuatro fantasmas desde su casa: cada uno espera su turno, sale guiado en línea recta por la puerta y, una vez fuera, no reentra.

## Alcance

**Incluido:**
- Liberación escalonada clásica: Blinky sale ya, Pinky a los 2 s, Inky a los 4 s y Clyde a los 6 s desde el inicio de la partida (0/120/240/360 frames a 60 fps).
- Ruta de salida guionizada: el fantasma va primero a la columna de su puerta (13 o 14), sube en línea recta por esa columna (celdas 13→12 puerta→11) y, alineado en fila 11, pasa a `free` y su IA normal de SPEC 01 toma el control.
- Bobbing vertical en su celda mientras espera su turno (`waiting`).
- Puerta deja de ser transitable para un fantasma una vez liberado (no reentra).
- Al perder vida, `resetPositions` reinicia la cola: los cuatro vuelven a la casa y repiten los mismos retardos de salida.
- La transición a salida guiada exige estar alineado a celda exacta (sin medio-pixel).

**No incluido:**
- Modo “asustado” / comer fantasmas (ya excluido en SPEC 01).
- Animaciones de ojos o sprites por estado.
- Cambios de velocidad (`GHOST_SPEED`) o de comportamiento por `kind` ya definido.
- Cambios en `src/js/render.js`: el bobbing es movimiento de posición, se dibuja igual.

## Modelo de datos

Nuevos campos por fantasma en `createGame()`:

```js
// en maze.js → destino de GHOST_STARTS o derivado: columna de salida por kind
const EXIT_COLS = { blinky: 13, pinky: 13, inky: 14, clyde: 14 };
```

```js
// en game.js
const RELEASE_DELAYS = { blinky: 0, pinky: 120, inky: 240, clyde: 360 }; // frames

// cada fantasma:
g = {
  ...,
  phase: 'waiting',   // 'waiting' | 'exiting' | 'free'
  releaseIn: RELEASE_DELAYS[ kind ],
  bobBase: 14,        // celda y de reposo para el bobbing (origen GHOST_STARTS)
};
```

Convenciones: mismos `DIRS`, `aligned()`, `canMove()`. Durante `exiting` el fantasma no llama a `decideGhost()`, avanza por una secuencia de direcciones fija. `releaseIn` solo descuenta mientras `phase === 'waiting'`.

## Plan de implementación

1. **game.js — estado**: añadir `RELEASE_DELAYS`, `EXIT_COLS` y los campos `phase`, `releaseIn`, `bobBase` al mapeo de fantasmas en `createGame()`. Verificación: abre la partida, consola limpia.
2. **game.js — espera**: en `moveGhost`, si `phase === 'waiting'`, no decidir; descuenta `releaseIn` (si llega a 0 → `phase = 'exiting'`) y aplica bobbing: `g.y = bobBase + sin(frame * T) * 0.4`, `g.x` fija a su celda. Verificación: fantasmas se balancean dentro de la casa.
3. **game.js — salida guiada**: si `phase === 'exiting'`, snap a celda entera y seguir la secuencia [ girar a `EXIT_COLS[kind]` (si aplica), `up` hasta y=13, `up` puerta y=12, `up` hasta y=11 ]; al llegar a (col, 11) → `phase = 'free'` y `bobBase` se ignora. Verificación: cada fantasma sale recto por su puerta sin oscilar.
4. **game.js — no reentrada**: `isWall`/`canMove` reciben una bandera `throughDoor` (true durante `exiting`); con `throughDoor=false`, la celda 3 bloquea también a los fantasmas (una vez `free`, la puerta es muro). Verificación: tras salir, ningún fantasma vuelve a colarse.
5. **game.js — reset**: `resetPositions` restaura `phase='waiting'`, `releaseIn = RELEASE_DELAYS[kind]` y `bobBase` desde `GHOST_STARTS`. Verificación: al perder vida todos vuelven y repiten la cola.
6. **Verificación final**: jugar 30 s y perder una vida; observar salidas escalonadas, bobbing, no reentrada y que Blinky/Pinky/Inky/Clyde siguen con su `kind` y color de SPEC 01.

## Criterios de aceptación

- [ ] Blinky sale de la casa en los primeros frames; Pinky a ±2 s, Inky a ±4 s y Clyde a ±6 s tras iniciar la partida.
- [ ] Cada fantasma sale en línea recta por la columna 13 o 14, sin oscilar dentro de la casa.
- [ ] Mientras espera, el fantasma se balancea verticalmente en su celda y no se desplaza de la casa.
- [ ] Tras salir, ningún fantasma vuelve a entrar por la puerta (ni por decisión de IA ni atravesándola).
- [ ] Al perder una vida, los cuatro regresan a sus posiciones y repiten la cola con los mismos retardos.
- [ ] Una vez `free`, cada fantasma conserva su `kind`, color y comportamiento de SPEC 01.
- [ ] Sin errores en consola al jugar ≥30 s y tras perder una vida.
- [ ] `MAZE` intacta; `game.grid` solo se muta al comer dots.

## Decisiones tomadas y descartadas

- **Sí:** ruta guionizada + escalonado. Una sola IA dentro de la casa oscila y da una salida fea.
- **Sí:** retardos clásicos 0/2/4/6 s. Ritmo arcade; da tiempo a ver cada salida. Descartado 0/1/2/3 s (demasiado simultáneo) y “todos a la vez” (pierde la parte fluida).
- **Sí:** bobbing vertical en espera. Descartado “quieto” (casa muerta) e “IA normal en casa”.
- **Sí:** la puerta bloquea a fantasmas liberados. Descartado reentrada (comportamiento actual, raíz de la salida poco fluida).
- **Sí:** reset reinicia la cola completa. Descartado “salir ya liberados” (inconsistente).
- **Sí:** solo toca `src/js/game.js`. Descartado tocar `render.js`/`maze.js` más allá de una opción para `EXIT_COLS`.

## Riesgos identificados

| Riesgo                                                   | Mitigación                                                        |
| -------------------------------------------------------- | ----------------------------------------------------------------- |
| Transición bobbing → salida deja el fantasma a medio pixel | Exigir snap a celda entera al pasar a `exiting`.                 |
| Bloquear la puerta rompe la propia salida guiada          | `throughDoor=true` durante `exiting`; solo `free` queda bloqueado. |
| `resetPositions` olvida los nuevos campos → cola no se repite | Restaurar `phase`, `releaseIn` y `bobBase` desde `GHOST_STARTS`. |