# Guia de sustentacion

## Idea central

El juego representa el laberinto como una matriz. Cada celda puede ser camino o muro; ademas, sobre algunas celdas se colocan galletas (objetivos) que el jugador debe recolectar. Por cada movimiento valido del jugador, el o los enemigos recalculan su siguiente paso usando una heuristica de distancia Manhattan combinada con conciencia de muros (filtro previo por `Laberinto.puedeEntrar`).

## Puntos tecnicos importantes

- `Laberinto` centraliza la matriz, los muros, las galletas (pool de 30 `Objetivo`) y la validacion de posiciones (`puedeEntrar`).
- `Jugador` solo conoce su posicion y delega la validacion al laberinto.
- `Enemigo` usa una **heuristica Manhattan con conciencia de muros**. Mantiene una frontera de celdas candidatas, en cada iteracion elige la celda con menor distancia Manhattan al jugador, y al expandir vecinos descarta los que `Laberinto.puedeEntrar` rechace (muros y bordes). El metodo `moverEvitando(jug, lab, ef, ec)` permite pasar la posicion del companero (modo Experto) para marcarla como muro temporal y evitar fusiones. Si la heuristica agota 150 expansiones sin alcanzar al jugador, `pasoVoraz` da un paso adaptativo evaluando los 4 vecinos transitables.
- `Juego` controla estados, dificultad, vidas, puntaje y transiciones. Pre-aloca **una sola vez** `Laberinto`, `Jugador` y los dos `Enemigo` para evitar la fragmentacion del heap del JackOS entre niveles.
- `GestorNiveles` aisla la generacion de mapas: 9 layouts (3 modos x 3 niveles), cada uno con dimensiones, muros, posiciones de inicio y galletas distintas.
- Toda clase con memoria dinamica expone `eliminar()` para liberar el heap al cerrar; los reusos intermedios pasan por `fijar(f, c)` sobre el objeto existente, sin alocar.
- El juego **evita movimiento aleatorio**: el enemigo siempre decide con base en el estado actual del tablero, lo que cumple el criterio de "estrategia clara y eficiente" de la rubrica.

## Como explicar la IA (heuristica Manhattan con conciencia de muros)

El metodo clave es `Enemigo.moverEvitando(jug, lab, ef, ec)` (`moverHacia` es un wrapper que pasa `-1, -1` cuando no hay companero). Idea en una frase: **la frontera nunca contiene celdas inalcanzables, y entre las alcanzables siempre se elige la mas cercana al jugador segun distancia Manhattan**.

Pasos:

1. Limpia la marca de visitados y guarda la posicion del enemigo en la frontera.
2. Si hay companero (modo Experto), marca su celda como visitada para que el segundo enemigo no la elija (sentinela `(-1, -1)` significa "sin companero").
3. Mientras la frontera no este vacia (y hasta un tope de 150 expansiones para mantener la fluidez del VM):
   - **Heuristica Manhattan**: recorre la frontera, calcula `|f - fJugador| + |c - cJugador|` para cada celda y selecciona la de menor valor. La intercambia con el frente para procesarla.
   - Si esa celda coincide con el jugador, recupera el primer paso registrado y mueve al enemigo a esa direccion.
   - Si no, expande sus 4 vecinos.
4. **Conciencia de muros**: cada vecino candidato pasa por `Laberinto.puedeEntrar(f, c)`. Esto rechaza muros (`celdas[idx] = 1`) y posiciones fuera de bordes. Solo los vecinos transitables y no visitados entran en la frontera. Por construccion, la frontera nunca contiene una celda a la que el enemigo no pueda llegar.
5. Para cada celda agregada, guarda **cual fue el primer paso** desde el origen: si el padre es el origen, el primer paso es el vecino mismo; si no, hereda el primer paso del padre. Esto permite que cuando la heuristica llega al jugador, el enemigo sepa que direccion tomar AHORA, no el camino completo.
6. Si la frontera se vacia o se agota el tope sin alcanzar al jugador, entra en juego el **respaldo voraz** (`pasoVoraz`): evalua los 4 vecinos transitables (filtrados por `puedeEntrar` y, en modo Experto, excluyendo la celda del companero) y elige el que minimice la distancia Manhattan al jugador. Asi el enemigo siempre intenta acercarse, incluso cuando la heuristica selectiva no encuentra ruta en el tope de expansiones.

### Por que esta heuristica y no una busqueda exhaustiva

- **Cumple el enunciado**: la rubrica pide una IA con "estrategia clara y eficiente, no aleatoria". La distancia Manhattan es la heuristica admisible natural en una grilla con movimientos en 4 direcciones, y el filtro por muros garantiza movimientos legales.
- **Espacio acotado**: una BFS exhaustiva visitaria en peor caso todas las celdas alcanzables (hasta 403 en el mapa maximo). La heuristica selectiva explora primero los nodos prometedores y suele llegar al jugador con muchas menos expansiones.
- **Simplicidad en Jack**: no requiere una cola FIFO con desplazamiento de cabeza. La frontera vive en arreglos pre-alocados de tamano fijo y la seleccion del mejor nodo se hace con un swap al frente.
- **Robustez visual**: cuando no hay camino claro en 150 expansiones (mapas patologicos o jugador rodeado), el enemigo se queda quieto en lugar de hacer un movimiento ilegal o aleatorio.

### Diagrama del algoritmo

```
                +----------------------+
                |  Inicio: enemigo en  |
                |       (fe, ce)       |
                +----------+-----------+
                           |
                           v
                +--------------------------+
                |  Frontera = [(fe, ce)]   |
                |  primerPaso[origen]      |
                |       = (fe, ce)         |
                |  Marca visitado          |
                |  Si hay companero (ef,ec)|
                |     marca visitado(ef,ec)|
                +--------------------------+
                           |
                           v
              +------------+---------------+
              | Frontera vacia o tope=150? |
              +-----+------------------+---+
                    | NO               | SI
                    v                  v
        +---------------------+   +-------------------+
        |  Elige nodo i con   |   |  No mover (manten |
        |  min(Manhattan ->   |   |  posicion actual) |
        |    jugador)         |   +-------------------+
        |  Swap i <-> frente  |
        |  Pop x = frente     |
        |                     |
        |  x == jugador?      |
        |   SI -> mover a     |
        |        primerPaso[x]|
        |   NO ->             |
        |     por cada vecino |
        |     valido v:       |
        |       si no visit:  |
        |         encolar v   |
        |         marcar      |
        |         primerPaso[v]:
        |           si padre==|
        |           origen-> v|
        |           si no -> p|
        +---------+-----------+
                  |
                  +-----> volver al check
```

### Ejemplo paso a paso

Considere un laberinto 4x4 donde `.` es camino, `#` es muro, `E` es enemigo y `J` es jugador:

```
E . . .
. # # .
. # . .
. . . J
```

Iteraciones (`d` = distancia Manhattan al jugador `(3,3)`):

1. Frontera: `[(0,0)]` con `d=6`. Pop `(0,0)`, no es `J`. Expande sus vecinos pasando `puedeEntrar`: `(0,1) d=5` y `(1,0) d=5` entran. Las celdas `(-1,0)` y `(0,-1)` son rechazadas por estar fuera del grid.
2. Frontera: `[(0,1), (1,0)]`. Empate en `d=5`; toma el primero por orden. Pop `(0,1)`. Expande `(0,2) d=4`. La celda `(1,1)` seria valida geometricamente pero `puedeEntrar` la rechaza por ser muro.
3. Frontera: `[(1,0) d=5, (0,2) d=4]`. Min = `(0,2)`. Swap al frente, pop. Expande `(0,3) d=3`.
4. Min = `(0,3) d=3`. Pop. Expande `(1,3) d=2`.
5. Min = `(1,3) d=2`. Pop. Expande `(2,3) d=1`.
6. Min = `(2,3) d=1`. Pop. Expande `(3,3) d=0`.
7. Min = `(3,3) d=0`. Pop. **Coincide con el jugador.** Primer paso registrado: `(0,1)`.

Resultado: el enemigo se mueve de `(0,0)` a `(0,1)`, primer paso de la ruta por arriba. La heuristica llego al jugador en 6 expansiones; una busqueda exhaustiva por niveles habria explorado tambien `(1,0)`, `(2,0)` y `(3,0)` (4 expansiones extra) antes de cerrar el camino, y la conciencia de muros mantuvo a `(1,1)` y `(2,1)` fuera de la frontera todo el tiempo.

## Mejoras agregadas sobre el alcance minimo

- Nueve mapas unicos (3 por cada modo) con diseno progresivo inspirado en Pac-Man y dimensiones de hasta 13x31 celdas.
- Tres modos de dificultad con parametros independientes:
  - **Entrenamiento**: 5 vidas, enemigo solo avanza 1 de cada 3 turnos (retrocede en los otros), multiplicador x1.
  - **Normal**: 3 vidas, enemigo 1:1, multiplicador x2.
  - **Experto**: 1 vida, **dos enemigos simultaneos** que se evitan entre si para formar una pinza tactica, multiplicador x3.
- Sistema de galletas: el nivel termina solo cuando se recolectan todas.
- Menu retro con navegacion por flechas y seleccion directa.
- HUD permanente con vidas, nivel y puntaje.
- Pantallas de victoria y derrota con puntaje final.
- **Gestion estricta de memoria**: pool persistente (Laberinto, Jugador, los dos Enemigo y 30 Objetivo se pre-alocan una sola vez) + `eliminar()` al cerrar. Esto eliminó el desbordamiento del heap del JackOS que aparecia al cambiar de modo varias veces.
- **Validacion defensiva en setters de mapa**: `Laberinto.setObjetivo / setInicio / setEnemigoInicio / setEnemigo2Inicio` rechazan silenciosamente coordenadas que caigan en muro o fuera del grid (via `puedeEntrar`). Es una capa redundante sobre la validacion de movimiento que blinda al juego contra mapas mal definidos: nunca se coloca una galleta inalcanzable, ni un actor empieza atrapado.
- **Respaldo voraz en la IA** (`Enemigo.pasoVoraz`): si la heuristica selectiva no encuentra al jugador en 150 expansiones, el enemigo evalua sus 4 vecinos transitables y se mueve al de menor distancia Manhattan. Cumple con el criterio "adaptativo e inteligente" de la rubrica.

## Preguntas anticipadas de sustentacion

### "La pantalla de Nand2Tetris es muy limitada, &iquest;como lograron tanto detalle visual?"

La premisa de la pregunta es enga&ntilde;osa y conviene aclararla primero antes de explicar la tecnica.

**1. La pantalla del Hack no es de "pocos pixeles".** Es de **512x256 pixeles monocromaticos** = 131,072 pixeles individuales. Lo que hace que la mayoria de los proyectos de Jack se vean limitados no es la resolucion de hardware, sino que usan exclusivamente `Output.printString` / `Output.printChar` para todo, y la consola de texto del JackOS esta fija en **64 columnas x 23 filas** (caracteres tipograficos de 8x11 px). Eso reduce el espacio expresivo de 131,072 pixeles a apenas 1,472 celdas de texto.

**2. Nuestra estrategia: separar `Output` y `Screen` por capas.**

| Capa | Clase Jack | Para que sirve |
|---|---|---|
| Texto (HUD) | `Output.printString` / `printInt` | Solo el nivel y el puntaje en el HUD superior |
| Graficos (todo lo demas) | `Screen.drawRectangle`, `drawLine`, `drawCircle`, `drawPixel` | Muros, jugador, enemigos, cristales, corazones, corona, marcos, etc. |

Al usar `Screen.*` accedemos al control pixel-perfect de los 131,072 pixeles del framebuffer Hack (`0x4000`-`0x5FFF` en el mapa de memoria). El JackOS expone solo cuatro primitivas, pero combinadas componen cualquier figura.

**3. Tecnica de composicion 1-bit por capas (cada sprite es un mini-collage).**

El truco mas importante: como la pantalla es 1-bit (sin canal alfa, sin transparencia), simulamos "huecos" alternando `Screen.setColor(true)` (negro) y `Screen.setColor(false)` (blanco) sobre la misma region. Cada sprite se construye en 3 pasadas:

```
1. setColor(true)  -> drawRectangle / drawCircle del contorno y cuerpo
2. setColor(false) -> drawRectangle interior (la "cara" o "ojos" blancos)
3. setColor(true)  -> drawRectangle / drawLine para detalles finos
                     (pupilas, sonrisa, cuerno, cruz interna del muro)
```

Ejemplo concreto del sprite del jugador (`Jugador.dibujar`, 12x12 px) en celda 16x16:

```jack
// 1. Cabeza negra
do Screen.setColor(true);
do Screen.drawRectangle(x + 2, y + 2, x + 13, y + 13);
// 2. Cara blanca (recorte interior)
do Screen.setColor(false);
do Screen.drawRectangle(x + 4, y + 4, x + 11, y + 11);
// 3. Detalles negros: ojos + sonrisa
do Screen.setColor(true);
do Screen.drawRectangle(x + 5, y + 5, x + 6, y + 7);
do Screen.drawRectangle(x + 9, y + 5, x + 10, y + 7);
do Screen.drawLine(x + 6, y + 10, x + 9, y + 10);
do Screen.drawPixel(x + 5, y + 9);
do Screen.drawPixel(x + 10, y + 9);
```

Total: 6 primitivas para una cara expresiva. Eso es **cientos de veces mas barato** que iterar `drawPixel` celda por celda.

**4. Eleccion de primitivas por costo.** No todas las primitivas del JackOS cuestan lo mismo en ciclos VM:

- `drawRectangle` es la mas barata por pixel cubierto (escribe palabras de 16 bits enteras directo al framebuffer cuando el rectangulo esta alineado).
- `drawLine` es proporcional a la longitud.
- `drawCircle` requiere `r` iteraciones del algoritmo de Bresenham; costoso para radios grandes.

Por eso usamos `drawCircle` **solo** donde aporta caracter (el domo del enemigo, el sprite gigante de derrota) y `drawRectangle`/`drawLine` para todo lo demas. La escena de partida completa (mapa 13x31 + 2 enemigos + jugador + cristales + HUD) cabe en ~250-400 primitivas por frame, suficientemente rapido en modo Fast de la VM.

**5. Coordenadas calculadas, no hardcodeadas.** Cada celda del grid se posiciona con `Laberinto.celdaX(c)` y `Laberinto.celdaY(f)` (origen `(10, 42)`, paso 16 px). Eso deja **fila 0 de Output (los 8x11 px superiores) libre** para el HUD textual, y reserva los 214 pixeles restantes para el grid. Los sprites se dibujan en coordenadas relativas a `(celdaX, celdaY)`, asi cualquier ajuste del origen propaga sin cambiar codigo.

**6. Pool de memoria pre-alocada.** Dibujar 250+ primitivas por frame **NO** asigna ni libera memoria del heap — todos los sprites son metodos sobre objetos pre-alocados una sola vez (ver seccion de gestion estricta de memoria). Eso evita la fragmentacion que tipicamente ralentiza la VM despues de varios niveles.

**Resumen de la respuesta corta para sustentacion**: la pantalla Hack tiene 131,072 pixeles individuales (no "unos cuantos"); el limite de "pocos caracteres" solo aplica a `Output.*`. Nosotros separamos las capas: `Output` para HUD numerico y `Screen` para todo el arte. Cada sprite es un collage 1-bit de 3 pasadas (cuerpo negro, recorte blanco, detalles negros), construido solo con `drawRectangle`, `drawLine`, `drawCircle` y `drawPixel`. La eleccion de primitivas se hace por costo en ciclos VM (rectangulos primero, circulos solo donde aportan) y la coherencia visual viene del tema unificado "Cripta del Centinela".

---

## Casos de prueba manuales

| Caso | Resultado esperado |
|---|---|
| Intentar moverse contra un muro | El jugador no cambia de celda. |
| Intentar salir del borde | El jugador no cambia de celda. |
| Recolectar todas las galletas del nivel | Avanza al siguiente nivel; si era el 3, muestra victoria. |
| Dejar que el enemigo alcance al jugador | Resta una vida y reinicia el nivel; si era la ultima, muestra derrota. |
| Presionar `Q` en partida | Regresa al menu principal sin perder progreso (el modo y nivel se reinician al elegir nuevo modo). |
| Presionar `Q` en el menu | Muestra "GRACIAS POR JUGAR" y vuelve al menu al presionar cualquier tecla. |
| Modo Entrenamiento | 5 vidas, enemigo retrocede 2 de cada 3 turnos (avance neto cada 3 pasos). |
| Modo Normal | 3 vidas, enemigo avanza cada paso del jugador. |
| Modo Experto | 1 vida, dos enemigos avanzan a la vez y se evitan entre si. |
| Cambiar de modo varias veces seguidas | No se desborda el heap (pool persistente). |
| Jugador encerrado por muros (jugador queda en isla sin acceso) | El enemigo no se congela: el respaldo voraz lo acerca al maximo posible. |
| Mapa con galleta accidentalmente sobre muro | El setter defensivo descarta la galleta; el nivel termina correctamente al recolectar las validas. |
| Pantalla de victoria vs derrota | Marcos visualmente diferentes (doble marco concentrico vs bloque solido) para identificacion inmediata. |
