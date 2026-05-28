# Guia de sustentacion

## Idea central

El juego representa el laberinto como una matriz. Cada celda puede ser camino o muro; ademas, sobre algunas celdas se colocan galletas (objetivos) que el jugador debe recolectar. Por cada movimiento valido del jugador, el enemigo recalcula con BFS la ruta mas corta hasta el y avanza un paso.

## Puntos tecnicos importantes

- `Laberinto` centraliza la matriz, los muros, las galletas y la validacion de posiciones (`puedeEntrar`).
- `Jugador` solo conoce su posicion y delega la validacion al laberinto.
- `Enemigo` usa **BFS** (busqueda por amplitud) para encontrar el primer paso de la ruta mas corta hacia el jugador. Si la cola se desborda o el jugador queda inalcanzable, cae a un movimiento voraz por distancia Manhattan.
- `Juego` controla estados, dificultad, vidas, puntaje y transiciones.
- `GestorNiveles` aisla la generacion de mapas: tres niveles, cada uno con dimensiones, muros, inicios y galletas distintos.
- Toda clase con memoria dinamica expone `eliminar()` para evitar fugas en el heap.
- El juego **evita movimiento aleatorio**: el enemigo siempre decide con base en el estado actual del tablero, lo que cumple el criterio de "estrategia clara y eficiente" de la rubrica.

## Como explicar la IA (BFS)

El metodo clave es `Enemigo.moverHacia(jugador, laberinto)`:

1. Inicializa una cola con la posicion actual del enemigo.
2. Marca esa celda como visitada.
3. Mientras la cola tenga elementos y no haya hallado al jugador:
   - Extrae la celda al frente de la cola.
   - Si coincide con la posicion del jugador, registra el indice y termina.
   - Si no, agrega los vecinos validos (arriba, abajo, izquierda, derecha) que sean transitables y no visitados.
4. Para cada celda agregada, guarda **cual fue el primer paso** desde el origen (no la celda misma): si el padre es el origen, el primer paso es el vecino; si no, hereda el primer paso del padre.
5. Cuando halla al jugador, mueve al enemigo a ese primer paso.

### Diagrama del algoritmo

```
                +----------------------+
                |  Inicio: enemigo en  |
                |       (fe, ce)       |
                +----------+-----------+
                           |
                           v
                +----------------------+
                |  Cola = [(fe, ce)]   |
                |  primerPaso[origen]  |
                |       = (fe, ce)     |
                |  Marca visitado      |
                +----------+-----------+
                           |
                           v
              +------------+------------+
              |  Cola vacia o hallado?  |
              +-----+--------------+----+
                    | NO           | SI
                    v              v
        +---------------------+   +----------------+
        |  Extrae frente x    |   |  Si hallado:   |
        |                     |   |   mover a      |
        |  x == jugador?      |   |   primerPaso[x]|
        |   SI -> hallado=x   |   |  Si NO:        |
        |   NO ->             |   |   voraz por    |
        |     por cada vecino |   |   Manhattan    |
        |     valido v:       |   +----------------+
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

Iteraciones de BFS (la cola guarda `(fila, col, primerPaso)`):

1. Cola: `[(0,0,(0,0))]`. Visita `(0,0)`. No es `J`. Agrega `(0,1,(0,1))` y `(1,0,(1,0))`.
2. Cola: `[(0,1,(0,1)), (1,0,(1,0))]`. Visita `(0,1)`. Agrega `(0,2,(0,1))`.
3. Visita `(1,0)`. Agrega `(2,0,(1,0))`.
4. Visita `(0,2)`. Agrega `(0,3,(0,1))`.
5. Visita `(2,0)`. Agrega `(3,0,(1,0))`.
6. Visita `(0,3)`. Agrega `(1,3,(0,1))`.
7. Visita `(3,0)`. Agrega `(3,1,(1,0))`.
8. Visita `(1,3)`. Agrega `(2,3,(0,1))`.
9. Visita `(3,1)`. Agrega `(3,2,(1,0))`.
10. Visita `(2,3)`. Agrega `(3,3,(0,1))`.
11. Visita `(3,2)`. Es `(3,3)`? No, sigue.
12. Visita `(3,3)`. **Es el jugador.** Primer paso registrado: `(0,1)`.

Resultado: el enemigo se mueve de `(0,0)` a `(0,1)`, que es el primer paso de la ruta optima por arriba.

### Respaldo voraz

Si la cola supera la capacidad (`filas * cols`) o el BFS termina sin hallar al jugador (por ejemplo, jugador rodeado de muros), el enemigo evalua las cuatro direcciones y elige la que minimice la distancia Manhattan al jugador. Esto garantiza que el enemigo siempre haga algo razonable, incluso en mapas patologicos.

## Mejoras agregadas sobre el alcance minimo

- Tres niveles con mapas distintos en forma y tamano (10x12, 10x16 y 11x24).
- Tres modos de dificultad con parametros independientes (vidas, velocidad del enemigo, multiplicador de puntaje).
- Sistema de galletas: el nivel termina solo cuando se recolectan todas, no por llegar a una salida fija.
- Menu retro con navegacion por flechas y seleccion directa.
- HUD permanente con vidas, nivel y puntaje.
- Pantallas de victoria y derrota.
- Gestion estricta de memoria: cada clase con datos en heap libera con `eliminar()`.

## Casos de prueba manuales

| Caso | Resultado esperado |
|---|---|
| Intentar moverse contra un muro | El jugador no cambia de celda. |
| Intentar salir del borde | El jugador no cambia de celda. |
| Recolectar todas las galletas del nivel | Avanza al siguiente nivel; si era el 3, muestra victoria. |
| Dejar que el enemigo alcance al jugador | Resta una vida y reinicia el nivel; si era la ultima, muestra derrota. |
| Presionar `Q` en partida | Regresa al menu principal. |
| Presionar `Q` en el menu | Cierra el juego. |
| Modo Entrenamiento | 3 vidas, enemigo se mueve la mitad de los turnos. |
| Modo Experto | 1 vida, enemigo hace doble movimiento cada 3 turnos del jugador. |
