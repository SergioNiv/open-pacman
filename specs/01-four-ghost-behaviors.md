# SPEC 01 — Cuatro fantasmas con comportamientos propios

> **Estado:** Aprovado
> **Depende de:** —
> **Fecha:** 2026-10-11
> **Objetivo:** Añadir 4 fantasmas con comportamientos distintos (cazador agresivo, emboscador, cooperativo y tímido), reutilizando la infraestructura existente de `game.js`.

## Alcance

**Dentro:**

- Expandir `GHOST_STARTS` de 2 a 4 entradas en `src/js/maze.js` (2 posiciones nuevas dentro de la pen).
- 4 `kind` de fantasma en `decideGhost` (`src/js/game.js`): `hunter` (cazador agresivo, existente), `ambush` (emboscador), `cooperator` (cooperativo), `shy` (tímido).
- Intercambiar dos colores en `GHOST_COLORS` (`src/js/render.js`) para que cada fantasma tenga su color clásico.
- Actualizar la línea de `GHOST_STARTS` en `AGENTS.md` (ahora 4 posiciones).

**Fuera de alcance (para specs futuros):**

- Fase scatter (retirada a esquinas cada N segundos) — decidida explícitamente, no entra.
- Power pellets / fantasmas asustados / comer fantasmas — no existen hoy y no se añaden.
- Pathfinding con BFS — se decide greedy por celda.
- Velocidades por nivel, más niveles, colisión fantasma-fantasma.

## Modelo de datos

```js
// src/js/maze.js — 4 entradas; el orden define el color en render.js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: 'hunter' },      // cazador agresivo (rojo)
  { x: 14, y: 14, kind: 'ambush' },      // emboscador (rosa)
  { x: 13, y: 13, kind: 'cooperator' },  // cooperativo (cian)
  { x: 14, y: 13, kind: 'shy' },         // tímido (naranja)
];
```

- El objeto fantasma no cambia: `{ x, y, dir, speed, kind }` (como hoy en `createGame`).
- Los objetivos se calculan al vuelo en `decideGhost`; no se persiste ningún estado nuevo.
- Constantes nuevas en `game.js`: `AMBUSH_AHEAD = 4`, `INKY_AHEAD = 2`, `SHY_DIST = 8`.

Comportamientos (todos deciden solo al estar alineados, y nunca dan la vuelta salvo callejón, igual que hoy):

| kind         | objetivo                                                                              | decisión                                                        |
| ------------ | ------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| `hunter`     | celda actual de Pacman                                                                | greedy Manhattan al objetivo (código existente, sin cambios)    |
| `ambush`     | celda de Pacman + `4 × DIRS[pacman.dir]`                                              | greedy Manhattan al objetivo                                    |
| `cooperator` | `T = P2 + (P2 − H)`; `P2` = celda de Pacman + `2 × DIRS[pacman.dir]`, `H` = celda del `hunter` | greedy Manhattan al objetivo                              |
| `shy`        | si `dist > SHY_DIST` → celda de Pacman; si no → —                                     | greedy si `dist > SHY_DIST`, dirección aleatoria legal si no    |

Convenciones:

- `dist` es Manhattan (misma métrica que el `hunter`).
- Los objetivos no se recortan: si salen de la grilla, greedy funciona igual con las diferencias de coordenadas.
- Empates en greedy: gana la primera opción mínima en el orden de iteración de `DIRS` (determinista, como el `hunter` actual).
- El `kind` `'random'` desaparece: su slot pasa a ser `'ambush'` y el fallback aleatorio vive dentro de `'shy'`.

## Plan de implementación

1. `src/js/maze.js`: expandir `GHOST_STARTS` a las 4 entradas del modelo de datos (la 2ª cambia de `kind: 'random'` a `'ambush'`). Verificación manual: abrir la página y ver 4 fantasmas dentro de la pen (los 2 `kind` nuevos caen en el default aleatorio mientras tanto).
2. `src/js/game.js`: añadir rama `ambush` en `decideGhost` (objetivo = celda de Pacman + `4 × DIRS[pacman.dir]`). Verificación manual: con Pacman moviéndose a la derecha, el rosa apunta a la celda por delante, no a la suya.
3. `src/js/game.js`: añadir rama `cooperator` en `decideGhost` (`T = P2 + (P2 − H)`). Verificación manual: el cian no sigue las rutas del rojo.
4. `src/js/game.js`: añadir rama `shy` en `decideGhost` (greedy si `dist > SHY_DIST`, aleatoria legal si no). Verificación manual: con Pacman cerca, el naranja deambula sin rumbo; lejos, persigue.
5. `src/js/render.js`: intercambiar `GHOST_COLORS[1]` y `[2]` (rosa → emboscador, cian → cooperativo). Verificación manual: rojo, rosa, cian y naranja en ese orden.
6. `AGENTS.md`: actualizar la línea "Starts are hardcoded" a las 4 posiciones de `GHOST_STARTS`. Solo doc, el juego no cambia.

## Criterios de aceptación

- [ ] La página carga sin errores en consola y se dibujan 4 fantasmas con 4 colores distintos (rojo, rosa, cian, naranja).
- [ ] Con Pacman caminando en línea recta por un corredor, el rojo termina a menos de 4 celdas de él (lo persigue de verdad).
- [ ] Con Pacman moviéndose a la derecha, el rosa apunta a la celda por delante de Pacman, no a la celda de Pacman.
- [ ] El cian toma decisiones que no coinciden sistemáticamente con las del rojo ni del rosa (sus rutas divergen).
- [ ] Con Pacman a ≤ 8 celdas (Manhattan), el naranja cambia de dirección de forma aleatoria; a > 8, persigue.
- [ ] Al chocar, se pierde una vida y los 4 fantasmas vuelven a sus 4 posiciones de inicio correctas.
- [ ] `MAZE_STR` sigue siendo 31 strings × 28 chars, simétrico, y la pen conserva su puerta en las celdas (13,12) y (14,12).

## Decisiones

- **Sí:** los 4 comportamientos clásicos (cazador, emboscador, cooperativo, tímido). Es el set canónico y da la diferencia de personalidad pedida.
- **Sí:** reutilizar `hunter` como cazador agresivo. Ya existe, es greedy hacia la celda de Pacman y esa rama no se toca.
- **Sí:** greedy por celda (Manhattan) para los cuatro. Consistente con la arquitectura actual (sin pathfinding), sin datos nuevos, y suficiente para la persecución agresiva.
- **No:** BFS a la celda objetivo. Más preciso, pero añade una función de pathfinding y estado; si el cazador se queda corto, se reevalúa en un spec futuro.
- **No:** fase scatter. Simplifica; si llega, entra con su timer global en otro spec.
- **Sí:** el slot del `random` pasa a ser `ambush` y el tímido se llama `shy` (con fallback aleatorio). Evita un `kind` que mienta sobre su comportamiento.
- **Sí:** las 2 posiciones nuevas van en (13,13) y (14,13), formando un 2×2 sobre la puerta, sin mover las existentes. Mantiene `resetPositions` intacto (ya itera por índice).
- **Sí:** colores clásicos por fantasma (intercambiar rosa y cian en `GHOST_COLORS`). Una línea y da identidad visual a cada personalidad.
- **No:** clamping de objetivos fuera de grilla. Greedy no lo necesita.

## Riesgos

| Riesgo                                                      | Mitigación                                                                                  |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| El greedy del cazador se queda corto en esquinas (mínimos locales) | Es el comportamiento aceptado en esta spec (ver Decisiones). Si molesta, spec futuro con BFS. |
| Los 4 fantasmas salen apilados por la misma puerta de la pen | No hay colisión fantasma-fantasma (como hoy); al alinearse, cada uno elige su propia dirección y se separan. |
| `resetPositions` desincronizado si cambia el orden de `GHOST_STARTS` | `createGame` y `resetPositions` usan el mismo índice; lo cubre el criterio de choque → reinicio. |

## Qué **NO** está en este spec

- Fase scatter (retirada a esquinas por timer).
- Power pellets y fantasmas asustados.
- Pathfinding (BFS).
- Velocidades por nivel / más niveles.
- Colisión fantasma-fantasma.

Cada una de esas, si llega, va en su propio spec.
