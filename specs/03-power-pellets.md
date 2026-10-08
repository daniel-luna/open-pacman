# SPEC 03 - Power pellets en las esquinas

**Estado:** Approved
**Dependencias:** SPEC 01, SPEC 02
**Fecha:** 2026-10-08
**Objetivo:** Añadir una power pellet en cada esquina que vuelva a Pacman invencible durante 8 segundos para comerse a los fantasmas, los cuales se pintan azul claro, se mueven aleatoriamente y recuperan su estado normal al acabar el tiempo.

## Alcance

**Incluido:**

- 4 power pellets en (1,1), (26,1), (1,29), (26,29) —hoy dots— dibujadas como bola más grande que un dot.
- Tile nuevo: código 4 (carácter `'o'` en `MAZE_STR`).
- Comer pellet: +50 puntos, la quita de `grid` y activa el poder 8 s (480 frames).
- Durante el poder: los 4 fantasmas (incluidos los de casa) azul claro `#4d4dff`; en los últimos 2 s parpadean a blanco; los `free` eligen dirección aleatoria.
- Comer fantasma: +200/400/800/1600 según combo (tope 1600); el fantasma se teletransporta a su celda de la casa, espera 90 frames y repite la salida guiada de SPEC 02.
- Otra pellet durante el poder reinicia el contador a 8 s y el combo a 0.
- Perder vida corta el poder (reset de temporizador y combo).
- Las 4 pellets cuentan en `dotsRemaining` (victoria = dots + 4 pellets).
- Velocidad de fantasmas sin cambios.
- Actualizar la línea de códigos de tile en AGENTS.md.

**No incluido:**

- Ojos de fantasma / regreso a casa con IA o pathfinding.
- Indicador de tiempo restante en HUD o efecto visual en Pacman.
- Sonido.
- Velocidad reducida para fantasmas asustados.

## Modelo de datos

```js
// maze.js: parseTile 'o' → 4; las 4 esquinas de MAZE_STR pasan de '.' a 'o'

// game.js
const POWER_FRAMES = 480;              // 8 s a 60 fps
const GHOST_SCORES = [200, 400, 800, 1600];
const RESPAWN_WAIT = 90;               // frames en casa tras ser comido (~1,5 s)

// en game (createGame):
powerFramesLeft: 0,
ghostCombo: 0,

// render.js
const FRIGHT_COLOR = '#4d4dff';        // radio de pellet: 6 (dot: 2,5)
```

Convenciones: mismos `DIRS`, `aligned()`, `canMove()`; `MAZE` pristina; `game.grid` solo muta al comer (tiles 2 y 4).

## Plan de implementación

1. **maze.js**: `'o'`→4 en `parseTile`, las 4 esquinas de `MAZE_STR` pasan a `'o'`, comentar el código en la cabecera; `createGame` cuenta `v===2||v===4`. Verificación: carga limpia en consola.
2. **render.js**: dibujar tile 4 como bola grande (radio 6, `DOT_COLOR`). Verificación: 4 bolas visibles en las esquinas.
3. **game.js — poder**: añadir `POWER_FRAMES`, `GHOST_SCORES`, `RESPAWN_WAIT` y los campos `powerFramesLeft`, `ghostCombo` en `createGame()`; comer tile 4 en `movePacman` (+50, `grid=0`, `dotsRemaining--`, `powerFramesLeft=POWER_FRAMES`, `ghostCombo=0`); descuento del temporizador en `update()`. Verificación: pellet suma 50 y `game.powerFramesLeft` corre desde la consola.
4. **game.js + render.js — modo asustado**: en `decideGhost`, si `powerFramesLeft>0` → dirección aleatoria entre `choices`; en `draw`, color `#4d4dff` para los fantasmas mientras dure el poder y parpadeo a blanco si `powerFramesLeft≤120`. Verificación: fantasmas claritos 8 s, parpadean los 2 s finales y vuelven a la normalidad.
5. **game.js — comer fantasmas**: rama en el bucle de colisión: con poder activo → puntos según combo (tope 1600), `ghostCombo++`, teleport del fantasma a `GHOST_STARTS[i]` con `phase:'waiting'`, `releaseIn: RESPAWN_WAIT`, `bobBase` actualizado; sin poder → lógica actual. Verificación: fantasma comido suma puntos y repite la cola de salida.
6. **game.js — reset**: `resetPositions` limpia `powerFramesLeft` y `ghostCombo`. Verificación: perder vida con poder activo → todo normal al renacer.
7. **AGENTS.md**: añadir "4 power pellet" a la línea de códigos de tile. Verificación: leer el texto.

## Criterios de aceptación

- [ ] Hay exactamente 4 power pellets en (1,1), (26,1), (1,29), (26,29), dibujadas con radio mayor que un dot.
- [ ] Comer una pellet suma 50 puntos y deja su celda en `grid` como 0.
- [ ] Durante 8 s los 4 fantasmas (incluidos los de casa) son azul claro `#4d4dff`.
- [ ] En los últimos 2 s el color parpadea a blanco sin cortar el poder antes de tiempo.
- [ ] Mientras dura el poder, los fantasmas `free` eligen dirección aleatoria.
- [ ] Al acabar los 8 s recuperan color y comportamiento de SPEC 01/02.
- [ ] Comer un fantasma suma 200/400/800/1600 según combo, tope 1600.
- [ ] El fantasma comido reaparece en su celda de la casa, espera ~1,5 s y repite la salida guiada.
- [ ] Comer otra pellet durante el poder reinicia el contador a 8 s.
- [ ] Perder una vida con poder activo corta el poder tras el reset.
- [ ] La victoria exige comer todos los dots más las 4 pellets.
- [ ] Velocidad de fantasmas intacta; `MAZE` intacta; `game.grid` solo muta al comer.
- [ ] Sin errores en consola al jugar ≥30 s con pellet, comida de fantasma y pérdida de vida.

## Decisiones tomadas y descartadas

- **Sí:** 50 por pellet y serie clásica 200/400/800/1600. **No:** 200 fijo; sin puntos nuevos.
- **Sí:** aleatorio clásico durante el poder. **No:** huir maximizando distancia (más literal con "huyen", pero se eligió el comportamiento arcade).
- **Sí:** teleport a casa con espera de 90 frames. **No:** ojos con pathfinding. **No:** desaparecer hasta el fin del poder.
- **Sí:** repetir pellet reinicia a 8 s y pone el combo a 0. **No:** acumular tiempo. **No:** ignorar el temporizador.
- **Sí:** perder vida corta el poder. **No:** continúa tras el reset.
- **Sí:** tile 4 + `'o'` en `MAZE_STR`. **No:** lista `POWER_PELLETS` aparte.
- **Sí:** las pellets cuentan en `dotsRemaining`.
- **Sí:** azul claro `#4d4dff` + parpadeo a blanco en los 2 s finales. **No:** blanco fijo. **No:** parpadeo continuo.
- **Sí:** el poder incluye los fantasmas en casa. **No:** solo los `free`.
- **Sí:** velocidad sin cambios. **No:** fantasmas más lentos durante el poder.
- **Sí:** tope de combo en 1600 tras el 4º fantasma, aunque reaparezcan.

## Riesgos identificados

| Riesgo | Mitigación |
| --- | --- |
| Colisión con poder activo resta vida por error | `if/else` explícito: comer fantasma solo si `powerFramesLeft>0` |
| Pellet no descuenta la victoria | Contar `v===2\|\|v===4` al crear y restar al comer el tile 4 |
| Parpadeo corta el poder a los 2 s | El parpadeo es solo color en render; el contador llega hasta 0 |
| Fantasmas `waiting`/`exiting` ignoran la IA asustada | Solo cambia `decideGhost`; el color aplica a los 4 vía render |
| Teleport a medio píxel rompe la salida guiada | Reusar enteros exactos de `GHOST_STARTS` (patrón de `resetPositions`) |
