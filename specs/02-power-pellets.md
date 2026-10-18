# SPEC 02 — Cuatro power pellets con modo asustado y comer fantasmas

> **Status:** Aprobado
> **Depends on:** SPEC 01
> **Date:** 2026-10-18
> **Objective:** El tile 'o' introduce 4 power pellets en las esquinas arcade que activan 6 segundos de modo asustado, durante el cual Pacman puede comer fantasma por 200/400/800/1600.

## Scope

**In:**

- Nuevo tile `'o'` → código `4` en `parseTile` (`src/js/maze.js`), colocado en `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)` sustituyendo 4 dots existentes.
- Comer pellet: +50 puntos, temporizador de 360 ticks (6 s) en `game.pelletTimer` y racha de comidos `game.chain = 0`.
- Fantasmas asustados mientras `pelletTimer > 0`: velocidad `0.05`, dirección aleatoria (ignoran su `kind`), color azul `#2121ff` con parpadeo azul/blanco en los últimos 120 ticks.
- Comer fantasma asustado: `score += 200 * 2^chain` (200, 400, 800, 1600), `chain++`, y el fantasma vuelve a su posición de `GHOST_STARTS` como normal (peligroso, no comible) saliendo de la casa a los 60 ticks (`exitAt = game.tick + 60`).
- Chocar con un fantasma no asustado sigue costando 1 vida (radio < 0.5, sin cambios).
- Los 4 pellets cuentan para ganar vía `dotsRemaining` (se les hace `--` al comer).

**Out of scope (for future specs):**

- Modo "ojos" (retorno visual del fantasma corriendo a la casa).
- Duración del modo asustado variable por nivel.
- Fruta bonus y puntos extra al completar el laberinto.
- Sonido, HUD nuevo o animaciones adicionales.
- Cambios en vidas, radio de colisión o velocidades fuera del modo asustado.

## Data model

```js
// src/js/maze.js — nuevo código de tile
//   'o' → parseTile devuelve 4 (pellet). El resto sin cambios (1, 2, 3, 0).

// src/js/game.js — ampliación del estado de createGame()
game.pelletTimer = 0;  // ticks restantes de modo asustado; 0 = inactivo
game.chain = 0;        // racha de fantasmas comidos en el modo actual (0..3)

// fantasma (objeto de createGame/GHOST_STARTS.map)
ghost.frightened = false;  // true solo mientras el pellet activo lo asuste
ghost.speed = 0.1;         // pasa a 0.05 si frightened, 0.1 al revivir o al fin del modo
```

Convenciones:

- El modo asustado se mide con `game.tick` (decisión de SPEC 01): 360 ticks ≈ 6 s, parpadeo cuando `pelletTimer <= 120`.
- Al comer pellet, `frightened = true` para los 4 fantasmas (incluidos los revividos, como el arcade).
- El fantasma comido conserva su `kind`/`color`; solo cambia su estado de vuelta.

## Implementation plan

1. En `src/js/maze.js`: añadir `'o'` a `parseTile` (código 4) y cambiar los 4 dots de las esquinas arcade por `o`. Prueba: `createGame().grid` tiene exactamente 4 celdas con valor 4 y `dotsRemaining` las cuenta.
2. En `src/js/game.js` (`createGame`): añadir `pelletTimer: 0`, `chain: 0` y `frightened: false` por fantasma; en `movePacman`, comer tile 4 → `score += 50`, `pelletTimer = 360`, `chain = 0`, `frightened = true` en los 4 fantasmas. Prueba: pasar por un pellet suma 50 y activa el modo.
3. En `update()` (`src/js/game.js`): decrementar `pelletTimer`; al llegar a 0, restaurar `speed = 0.1` y `frightened = false`. Mientras dure, fantasmas asustados a `0.05`. Prueba: tras 6 s los fantasmas vuelven a la normalidad.
4. En `moveGhost`/`decideGhost` (`src/js/game.js`): si `g.frightened`, elegir dirección aleatoria en vez de la personalidad. Prueba: durante 6 s ningún fantasma usa `ghostTarget`.
5. En `update()` (colisión): si `g.frightened`, comerlo: `score += 200 * 2^chain`, `chain++`, `g.frightened = false`, `g.speed = 0.1`, teletransporte a `GHOST_STARTS[i]` con `waiting = true` y `exitAt = game.tick + 60`; si no, perder vida como hoy. Prueba: comidos seguidos rinden 200/400/800/1600 y el fantasma reaparece en la casa.
6. En `src/js/render.js`: dibujar el tile 4 como disco grande y parpadeante (radio ~7, on/off con `frame`); fantasmas asustados en azul `#2121ff`, parpadeando a `#ffffff` los últimos 120 ticks. Prueba: se ven 4 pellets grandes y fantasmas azules 6 s.
7. En `resetPositions()` (`src/js/game.js`): al perder vida, `pelletTimer = 0`, `chain = 0`, `frightened = false` y velocidades a `0.1` en todos. Prueba: perder vida limpia el modo asustado de golpe.

## Acceptance criteria

- [ ] El grid inicial tiene exactamente 4 tiles de código 4 en `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`.
- [ ] Comer un pellet suma 50 puntos al score.
- [ ] Comer un pellet activa 360 ticks (≈6 s) de modo asustado en los 4 fantasmas.
- [ ] Durante el modo, los fantasmas van a velocidad 0.05 y eligen dirección aleatoria (diferente de su comportamiento normal).
- [ ] Comiendo fantasmas seguidos en el mismo modo: 200, luego 400, luego 800, luego 1600.
- [ ] Un fantasma comido reaparece en su casilla de la casa, sale ~1 s después y ya no es comible hasta el siguiente pellet.
- [ ] Chocar con un fantasma no asustado sigue quitando 1 vida con radio < 0.5.
- [ ] Al terminar el modo (o al perder vida) los fantasmas vuelven a velocidad 0.1 y su comportamiento normal.
- [ ] Los fantasmas parpadean azul/blanco durante los últimos ~2 s del modo.
- [ ] Comer otro pellet reinicia el temporizador a 360 ticks y la racha a 200.
- [ ] Los 4 pellets cuentan en `dotsRemaining`; la partida se gana solo habiéndolos comido también.
- [ ] La partida carga sin errores en consola.

## Decisions

- **Sí:** tile `'o'` → código 4 en `maze.js` como primera clase del grid. Escalable y el render ya lee `game.grid`.
- **No:** lista de coordenadas aparte manteniendo dots. Estado duplicado y seguimiento manual.
- **Sí:** posiciones arcade `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`. Confirmado por el usuario; solo hay que reescribir 4 caracteres de `MAZE_STR`.
- **Sí:** +50 por pellet y cadena arcade 200/400/800/1600. Elegido sobre 200 fijo (más simple) por fidelidad arcade, confirmado por el usuario.
- **Sí:** racha reiniciada al comer otro pellet. Descrito y confirmado en la fase de preguntas.
- **Sí:** 6 s medidos en ticks `game.tick` con parpadeo final. Determinista y verificable; coincide con la convención de SPEC 01.
- **Sí:** movimiento aleatorio simple + mitad de velocidad (0.05). Elegido sobre "huir de Pacman": huir hace comer fantasmas automático y rompe el reto.
- **No:** modo "ojos". Más código (navegación de retorno, dibujo extra); el regreso inmediato a la casa es aceptable para este alcance.
- **No:** el fantasma comido sigue comible tras revivir — vuelve como normal (peligroso) salvo alimento nuevo. Evita encadenar puntos duplicados y coincide con el arcade.
- **Sí:** perder vida termina el modo asustado de golpe. Evita estados raros tras `resetPositions()` (decisión del usuario).
- **Sí:** los pellets cuentan en el único `dotsRemaining`. Evita un contador paralelo para la victoria.

## What is **not** in this spec

- Modo "ojos" de retorno a la casa.
- Duración de asustado por nivel o aceleración progresiva.
- Fruta bonus, sonido, HUD o animaciones nuevas.
- Cambios en vidas, radio de colisión o velocidades base (PACMAN_SPEED 0.125 / GHOST_SPEED 0.1).
