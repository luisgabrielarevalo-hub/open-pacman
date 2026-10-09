# SPEC 01 — Cuatro fantasmas con personalidades y salida secuencial

> **Status:** Aprobado
> **Depends on:** —
> **Date:** 2026-10-09
> **Objective:** El juego tiene cuatro fantasmas con comportamientos distintos y salida secuencial de la casa cada 2 segundos.

## Scope

**In:**

- Cuatro fantasmas activos con un `kind` distinto cada uno.
- Un fantasma agresivo que siempre elige el camino óptimo hacia Pacman.
- Salida secuencial de la casa a los 0, 2, 4 y 6 segundos de empezar (o de perder vida).
- Espera visible dentro de la casa (rebote vertical) hasta el turno de salida.
- Color propio por fantasma (rojo, rosa, cian, naranja).

**Out of scope (for future specs):**

- Modo asustado / comer fantasmas (power pellets).
- Niveles de velocidad por nivel o por fantasma.
- Sonido o animaciones nuevas más allá del rebote y los colores.
- Cambios en puntuación, vidas o colisión.

## Data model

```js
// Ampliación en src/js/maze.js
const GHOST_STARTS = [
  // { x, y, kind, color, exitDelay } — exitDelay en ticks de update (~60/s)
  // kind: 'chaser' | 'ambusher' | 'flanker' | 'random'
];

// Estado por fantasma en src/js/game.js (vía createGame)
const ghost = {
  x: 13, y: 14, dir: 'up', speed: 0.1, // GHOST_SPEED sin cambios
  kind: 'chaser',
  color: '#ff0000',
  exitAt: 0,      // tick de update en que puede salir
  waiting: true,  // true mientras rebota dentro de la casa
};
```

Convenciones:

- Coordenadas en celdas fraccionarias, origen arriba-izquierda (como el resto del juego).
- Un tick = una llamada a `update(game)` (≈1/60 s). Salidas en ticks 0, 120, 240, 360.
- Objetivos de caza (celda entera, distancia Manhattan como `hunter` actual):
  - `chaser` → posición de Pacman.
  - `ambusher` → Pacman + 4 celdas en su dirección actual.
  - `flanker` (corte lateral) → Pacman + 4 celdas en perpendicular a su dirección, hacia el lado donde está el fantasma (si Pacman va en horizontal, el objetivo se desplaza en vertical, y viceversa).
  - `random` → reutiliza el comportamiento aleatorio actual sin cambios.

## Implementation plan

1. Ampliar `GHOST_STARTS` en `src/js/maze.js` a 4 entradas dentro de la casa (fila 14), cada una con `kind`, `color` y orden de salida. Prueba: `createGame().ghosts.length === 4`.
2. Extender el objeto fantasma en `createGame()` (`src/js/game.js`) con `exitAt` (0/120/240/360) y `waiting:true`. Prueba: recargar y ver 4 fantasmas en la casa.
3. Implementar rebote vertical en la casa mientras `waiting` y subida por la puerta (tile 3, que ya es transitable para fantasmas) al llegar su turno. Prueba: al empezar, el 1.º sale de inmediato y los demás rebotan.
4. Implementar objetivos `ambusher` (+4 por delante) y `flanker` (corte lateral ±4 en perpendicular) en `decideGhost()`, manteniendo `chaser` (Manhattan directo, sin aleatoriedad) y `random` intactos. Prueba: cada `kind` elige su objetivo en una posición de prueba.
5. Hacer que `resetPositions()` restaure posiciones y reinicie los retardos de salida. Prueba: perder una vida recoloca a los 4 en la casa con la misma secuencia.
6. Conectar color por `kind` en `src/js/render.js` (reutilizar `GHOST_COLORS`: rojo `chaser`, rosa `ambusher`, cian `flanker`, naranja `random`). Prueba: los 4 fantasmas se ven de distinto color.

## Acceptance criteria

- [ ] La partida crea exactamente 4 fantasmas con `kind` distintos.
- [ ] El fantasma rojo (`chaser`) reduce siempre la distancia Manhattan a Pacman cuando tiene opciones (misma velocidad 0.1 que los demás).
- [ ] El fantasma rosa (`ambusher`) apunta a Pacman + 4 celdas en su dirección.
- [ ] El fantasma cian (`flanker`) apunta al corte lateral (perpendicular ±4).
- [ ] El fantasma naranja (`random`) se mueve de forma aleatoria como antes.
- [ ] Las salidas ocurren en orden a ~0, ~2, ~4 y ~6 segundos tras empezar.
- [ ] Ningún fantasma fuera de turno abandona la casa; los que esperan rebotan en vertical dentro de ella.
- [ ] Tras perder una vida, los 4 vuelven a la casa y la secuencia de salida se reinicia.
- [ ] Cada fantasma se dibuja con su color propio.
- [ ] La colisión sigue quitando 1 vida con radio <0.5 y no hay errores en consola.

## Decisions

- **Sí:** personalidades estilo arcade simplificado (persigue / embosca +4 / corte lateral / aleatorio). Cubre "cada uno con su forma de actuar" sin inventar IA compleja.
- **No:** duplicar 2 `hunter` + 2 `random`. No daría cuatro formas de actuar distintas.
- **Sí:** agresivo = camino óptimo siempre, misma velocidad 0.1. Decisión explícita del usuario; evita desbalancear con más velocidad.
- **Sí:** salida por temporizador 0/2/4/6 s con rebote vertical. Simple, determinista y verificable; alternativa por dots comidos descartada por ser más difícil de verificar.
- **Sí:** reiniciar la secuencia al perder vida. Comportamiento predecible y fácil de probar.
- **Sí:** 4 colores arcade (rojo, rosa, cian, naranja) reutilizando `GHOST_COLORS`. El array ya existe con esos 4 valores.
- **No:** power pellets / comer fantasmas. Va en otro spec si llega.

## What is **not** in this spec

- Modo asustado y comer fantasmas.
- Velocidades distintas por fantasma o por nivel.
- Sonido, HUD nuevo o animaciones extra.
- Cambios en puntuación, vidas o radio de colisión.
