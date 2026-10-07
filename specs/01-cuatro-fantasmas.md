# SPEC 01 - Cuatro fantasmas con comportamientos diferenciados

**Estado:** Approved
**Dependencias:** Ninguna
**Fecha:** 2026-10-07
**Objetivo:** Añadir 4 fantasmas con nombres clásicos, cada uno con comportamiento propio (uno agresivo que persigue posición actual), con colores distintos y manteniendo la colisión actual.

## Alcance

**Incluido:**
- Pasar de 2 a 4 fantasmas con nombres clásicos: Blinky, Pinky, Inky, Clyde.
- Asignar comportamientos diferenciados:
  - Blinky: agresivo, persigue la posición actual de Pacman (objetivo = celda de Pacman, sin lookahead).
  - Pinky: emboscada (apunta a donde irá Pacman; objetivo ~2 celdas por delante en dirección actual de Pacman).
  - Inky: patrulla/ida-vuelta (oscila entre dos puntos fijos dentro/fuera del área de casa de fantasmas o sigue un circuito corto).
  - Clyde: aleatorio (comportamiento tipo `random` actual).
- Colores distintos para cada fantasma en renderizado.
- Mantener colisión actual (tocar fantasma = perder vida, reset posiciones).
- Mantener velocidades actuales (`GHOST_SPEED`), alineación en celda, túnel, bloqueo por pared (solo pared), puerta solo bloquea a Pacman.
- Inicialización en casa de fantasmas con 4 posiciones.

**No incluido:**
- Modo “asustado” / comer fantasmas.
- Fantasmas que salen de la casa con temporizador (todos salen juntos o desde sus posiciones iniciales).
- Lookahead complejo para Blinky (solo posición actual).
- Animaciones/ojos adicionales por estado.

## Modelo de datos

Nuevos campos/valores:
- `ghost.kind` pasa a identificar nombre: `'blinky' | 'pinky' | 'inky' | 'clyde'`.
- `ghost.targetX`, `ghost.targetY` (opcionales, calculados al decidir) o simplemente lógica por kind.
- `GHOST_STARTS` pasa de 2 a 4 entradas con `{x,y,kind:'...'}` (nombres clásicos).
- Colores: mapeo por kind en `render.js` (Blinky rojo, Pinky rosa, Inky cian, Clyde naranja).

Posiciones iniciales (dentro del área de casa de fantasmas, fila 14 contexto pen; usar celdas transitables dentro de pen): 
- Blinky: (12,14)
- Pinky: (13,14)
- Inky: (14,14)
- Clyde: (15,14)

(Siempre dentro del recinto de casa; si alguna queda fuera por geometría exacta, ajustar a las 4 celdas centrales contiguas del pen.)

## Plan de implementación

1. **maze.js**: Actualizar `GHOST_STARTS` a 4 entradas con nombres clásicos y sus posiciones iniciales. Mantener `TUNNEL_ROW`, `MAZE`, etc. sin cambios.
2. **game.js**: 
   - Mantener `GHOST_SPEED` y lógica de movimiento/alineación.
   - `createGame()` mapea `GHOST_STARTS` a 4 fantasmas con `kind`, `dir:'up'`, `speed`.
   - Reescribir `decideGhost(game,g)` con lógica por `kind`:
     - `blinky` (agresivo): objetivo = posición redondeada actual de Pacman `(px,py)`. Elegir dirección (no opuesta) con menor distancia Manhattan a objetivo.
     - `pinky` (emboscada): objetivo = 2 celdas por delante de Pacman en su dirección actual. Si dirección es inválida, caer a posición actual. Usar Manhattan.
     - `inky` (patrulla): definir dos waypoints fijos (p. ej. esquina superior-izquierda del área pen y esquina superior-derecha del área pen) y alternar entre ellos cuando llegue/alinee. Si atascado, caer a aleatorio válido (excluyendo opuesta).
     - `clyde` (aleatorio): comportamiento igual a `random` actual (elegir entre opciones válidas, excluir opuesta; si ninguna, 180).
   - Asegurar que cuando `choices` vacío, permita giro 180 (ya existe). No cambiar bloqueo (solo pared).
3. **render.js**:
   - Reemplazar `GHOST_COLORS = ['#ff0000','#00ffff','#ffb8ff','#ffb852']` por mapeo por `kind`: `blinky:'#ff0000'`, `pinky:'#ffb8ff'`, `inky:'#00ffff'`, `clyde:'#ffb852'`.
   - `draw()` pasa color según `g.kind` (con fallback).
4. **index.html / carga**: Sin cambios (orden actual válido).
5. **Verificación**: Abrir `src/index.html`, comprobar 4 fantasmas con colores distintos, comprobar Blinky va hacia Pacman, Pinky apunta por delante, Inky va/alternates entre waypoints, Clyde errático. Colisión sigue funcionando. No reaparecen dots.

## Criterios de aceptación

- [ ] Existen exactamente 4 fantasmas con nombres `blinky`, `pinky`, `inky`, `clyde`.
- [ ] Cada fantasma tiene color distinto visible.
- [ ] Blinky elige ruta hacia posición actual de Pacman (distancia Manhattan mínima entre opciones válidas, sin mirar atrás salvo callejón).
- [ ] Pinky apunta a ~2 celdas adelante de dirección actual de Pacman.
- [ ] Inky alterna entre al menos 2 waypoints definidos (patrulla detectable).
- [ ] Clyde se mueve aleatoriamente entre opciones válidas (comportamiento no determinista hacia Pacman).
- [ ] Colisión con cualquier fantasma decrementa vidas/reset como antes.
- [ ] `game.grid` mutado solo al comer dots; `MAZE` intacto.
- [ ] Sin errores en consola al cargar/reiniciar.

## Decisiones tomadas y descartadas

- **Nombres clásicos**: Elegidos para claridad y tradición (Blinky/Pinky/Inky/Clyde).
- **Blinky agresivo = posición actual**: Requerido explícitamente (sin lookahead). Más simple y directo.
- **Pinky ambush 2 pasos adelante**: Aproxima comportamiento clásico sin complejidad excesiva.
- **Inky patrulla**: Usa ida-vuelta entre waypoints (clásico “errático”/patrullero). Evita simular pareja con Blinky.
- **Clyde aleatorio**: Coincide con `random` existente.
- **Mantener colisión**: Evita introducir modo asustado ahora (fuera de alcance).
- **Descartado: lookahead para Blinky**: Contradice “posición actual”.
- **Descartado: modo frightened**: Requiere sprites/estados adicionales; pospuesto.

## Riesgos identificados

- **Waypoints de Inky**: Deben caer dentro de área transitables (evitar quedar atrapado permanentemente). Solución: elegir celdas alineadas, validar rutas o caer a aleatorio si bloqueado.
- **Pinky mira fuera del mapa**: Al calcular objetivo, clamp a bordes o si celda es inválida, usar posición actual de Pacman.
- **Inicialización 4 fantasmas**: Asegurar `GHOST_STARTS` tiene 4 entradas y `createGame` no asume 2.
