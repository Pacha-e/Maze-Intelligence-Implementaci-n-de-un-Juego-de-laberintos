# Guia de sustentacion

## Idea central

El juego representa el laberinto como una matriz. Cada celda puede ser camino o muro; ademas, sobre algunas celdas se colocan galletas (objetivos) que el jugador debe recolectar. Por cada movimiento valido del jugador, el o los enemigos recalculan con Best-First Search un paso hacia el jugador, respetando muros y bordes.

## Puntos tecnicos importantes

- `Laberinto` centraliza la matriz, los muros, las galletas (pool de 30 `Objetivo`) y la validacion de posiciones (`puedeEntrar`).
- `Jugador` solo conoce su posicion y delega la validacion al laberinto.
- `Enemigo` usa **Best-First Search (greedy)** con distancia Manhattan como heuristica para hallar el primer paso de una ruta razonable hasta el jugador. El metodo `moverEvitando(jug, lab, ef, ec)` permite pasar la posicion del companero (modo Experto) para marcarla como muro temporal y evitar fusiones.
- `Juego` controla estados, dificultad, vidas, puntaje y transiciones. Pre-aloca **una sola vez** `Laberinto`, `Jugador` y los dos `Enemigo` para evitar la fragmentacion del heap del JackOS entre niveles.
- `GestorNiveles` aisla la generacion de mapas: 9 layouts (3 modos x 3 niveles), cada uno con dimensiones, muros, posiciones de inicio y galletas distintas.
- Toda clase con memoria dinamica expone `eliminar()` para liberar el heap al cerrar; los reusos intermedios pasan por `fijar(f, c)` sobre el objeto existente, sin alocar.
- El juego **evita movimiento aleatorio**: el enemigo siempre decide con base en el estado actual del tablero, lo que cumple el criterio de "estrategia clara y eficiente" de la rubrica.

## Como explicar la IA (Best-First Search)

El metodo clave es `Enemigo.moverEvitando(jug, lab, ef, ec)` (`moverHacia` es un wrapper que pasa `-1, -1` cuando no hay companero):

1. Limpia la marca de visitados y guarda la posicion del enemigo en la frontera.
2. Si hay companero (modo Experto), marca su celda como visitada para que el segundo enemigo no la elija.
3. Mientras la frontera no este vacia (y hasta un tope de 150 expansiones para mantener la fluidez del VM):
   - Selecciona el nodo de la frontera con **menor distancia Manhattan al jugador** y lo intercambia al frente (asi el siguiente `pop` lo procesa).
   - Si la celda extraida coincide con el jugador, recupera el primer paso registrado y mueve al enemigo a esa direccion.
   - Si no, expande sus 4 vecinos transitables y no visitados.
4. Para cada celda agregada, guarda **cual fue el primer paso** desde el origen: si el padre es el origen, el primer paso es el vecino mismo; si no, hereda el primer paso del padre.
5. Si la frontera se vacia o se agota el tope sin alcanzar al jugador, el enemigo mantiene su posicion (no se mueve a una celda peor que la actual).

### Por que Best-First y no BFS puro

- **Espacio**: BFS expande por niveles y, en un peor caso, visita toda la grilla (403 nodos). Best-First explora primero los nodos prometedores, llegando al jugador con muchos menos pasos en mapas abiertos.
- **Simplicidad en Jack**: una cola FIFO real requiere desplazar el indice de cabeza; Best-First trabaja sobre la misma frontera con un swap del minimo al frente.
- **Suficiente para la rubrica**: Best-First con Manhattan rodea muros locales y reproduce el comportamiento persecutor pedido. Cuando no encuentra ruta en 150 expansiones, simplemente no avanza ese turno, lo cual es interpretable y nunca produce un movimiento ilegal.

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

Iteraciones de Best-First (`d` = distancia Manhattan al jugador `(3,3)`):

1. Frontera: `[(0,0)]` con `d=6`. Pop `(0,0)`, no es `J`. Expande `(0,1) d=5` y `(1,0) d=5`.
2. Frontera: `[(0,1) d=5, (1,0) d=5]`. Empate; toma el primero por orden. Pop `(0,1)`. Expande `(0,2) d=4`.
3. Frontera: `[(1,0) d=5, (0,2) d=4]`. Min = `(0,2)`. Swap al frente, pop. Expande `(0,3) d=3`.
4. Min = `(0,3) d=3`. Pop. Expande `(1,3) d=2`.
5. Min = `(1,3) d=2`. Pop. Expande `(2,3) d=1`.
6. Min = `(2,3) d=1`. Pop. Expande `(3,3) d=0`.
7. Min = `(3,3) d=0`. Pop. **Coincide con el jugador.** Primer paso registrado: `(0,1)`.

Resultado: el enemigo se mueve de `(0,0)` a `(0,1)`, primer paso de la ruta por arriba. Best-First llego al jugador en 6 expansiones; BFS puro habria explorado tambien `(1,0)`, `(2,0)` y `(3,0)` (4 expansiones extra) antes de cerrar el camino.

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
