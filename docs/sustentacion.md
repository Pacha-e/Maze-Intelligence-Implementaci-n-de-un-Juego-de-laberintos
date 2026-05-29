# Guía de sustentación

## Idea central

El laberinto se representa como una matriz donde cada celda es camino o muro. Sobre algunas celdas se colocan galletas que el jugador debe recolectar para terminar el nivel. Por cada movimiento válido del jugador, el o los enemigos recalculan su siguiente paso usando una heurística de distancia Manhattan con filtro de muros.

## Puntos técnicos importantes

- `Laberinto` centraliza la matriz, los muros, las galletas y la validación de posiciones.
- `Jugador` solo conoce su posición y delega la validación al laberinto.
- `Enemigo` usa una heurística Manhattan que respeta los muros: mantiene una frontera de celdas alcanzables, en cada paso elige la más cercana al jugador y al expandir vecinos descarta los que `puedeEntrar` rechace. El método `moverEvitando(jug, lab, ef, ec)` permite pasar la posición del compañero (modo Experto) para marcarla como muro temporal. Si en 150 expansiones no llega al jugador, `pasoVoraz` da un paso adaptativo evaluando los 4 vecinos transitables.
- `Juego` controla estados, dificultad, vidas, puntaje y transiciones. Pre-aloca todos los objetos dinámicos una sola vez para evitar fragmentar el heap.
- `GestorNiveles` aísla la generación de los 9 mapas (3 modos x 3 niveles).
- Toda clase con memoria dinámica expone `eliminar()`. Los reusos intermedios pasan por `fijar(f, c)` sobre el objeto existente, sin reservar memoria nueva.
- El enemigo nunca se mueve al azar: cada decisión depende del estado actual del tablero.

## Cómo explicar la IA

El método clave es `Enemigo.moverEvitando(jug, lab, ef, ec)`. La idea en una frase: la frontera nunca contiene celdas inalcanzables, y entre las alcanzables siempre se elige la más cercana al jugador.

Pasos:

1. Limpia los visitados y guarda la posición actual del enemigo como nodo inicial.
2. Si hay compañero (modo Experto), marca su celda como visitada para que el segundo enemigo no la elija. El sentinela `(-1, -1)` significa que no hay compañero.
3. Mientras la frontera tenga nodos y no se hayan superado 150 expansiones:
   - Calcula la distancia Manhattan al jugador para cada nodo de la frontera y selecciona el menor. Lo intercambia al frente para procesarlo.
   - Si esa celda es la del jugador, recupera el primer paso registrado y mueve al enemigo en esa dirección.
   - Si no, expande sus 4 vecinos. Cada vecino pasa por `puedeEntrar` antes de entrar a la frontera.
4. Para cada celda que entra a la frontera, se guarda cuál fue su primer paso desde el origen: si el padre es el origen, el primer paso es el vecino mismo; si no, hereda el primer paso del padre. Así cuando la búsqueda llega al jugador, el enemigo sabe qué dirección tomar ahora.
5. Si la frontera se agota o se llega al tope sin alcanzar al jugador, entra el respaldo voraz: evalúa los 4 vecinos transitables del enemigo (excluyendo la celda del compañero en modo Experto) y elige el que minimice la distancia Manhattan.

## Por qué esta heurística

- La distancia Manhattan es la heurística natural en una grilla de 4 direcciones.
- Una BFS exhaustiva podría visitar las 403 celdas en el peor caso. La búsqueda guiada por la heurística suele llegar mucho antes.
- No requiere una cola FIFO: la frontera vive en arreglos pre-alocados y la selección del mejor nodo se hace con un swap al frente.
- Cuando no hay camino claro, el respaldo voraz garantiza un movimiento adaptativo en lugar de quedarse quieto.

## Ejemplo paso a paso

Mapa 4x4 donde `.` es camino, `#` es muro, `E` es enemigo y `J` es jugador:

```
E . . .
. # # .
. # . .
. . . J
```

Iteraciones (`d` es la distancia Manhattan al jugador en `(3,3)`):

1. Frontera: `[(0,0)]` con `d=6`. Se procesa `(0,0)`, no es el jugador. Se expanden sus vecinos: `(0,1) d=5` y `(1,0) d=5` entran. Las celdas `(-1,0)` y `(0,-1)` se rechazan por estar fuera del grid.
2. Frontera: `[(0,1), (1,0)]`. Empate en `d=5`, se toma el primero. Se expande `(0,2) d=4`. La celda `(1,1)` se rechaza por ser muro.
3. Frontera: `[(1,0) d=5, (0,2) d=4]`. El menor es `(0,2)`. Se expande `(0,3) d=3`.
4. Se procesa `(0,3)`. Se expande `(1,3) d=2`.
5. Se procesa `(1,3)`. Se expande `(2,3) d=1`.
6. Se procesa `(2,3)`. Se expande `(3,3) d=0`.
7. Se procesa `(3,3)`, que coincide con el jugador. El primer paso registrado es `(0,1)`.

Resultado: el enemigo se mueve de `(0,0)` a `(0,1)`, el primer paso del camino por arriba. La heurística llegó en 6 expansiones. Una BFS por niveles habría explorado también `(1,0)`, `(2,0)` y `(3,0)` antes de cerrar el camino.

## Mejoras sobre el alcance mínimo

- 9 mapas únicos con diseño progresivo, hasta 13x31 celdas.
- 3 modos de dificultad con vidas, frecuencia de enemigos y multiplicador de puntaje distintos.
- Sistema de galletas: el nivel termina cuando se recolectan todas.
- Modo Experto con dos enemigos que se evitan entre sí.
- Menú con navegación por flechas y selección directa por tecla.
- HUD permanente con vidas, nivel y puntaje.
- Pantallas de victoria y derrota con sprites grandes y puntaje final.
- Pool de memoria persistente: ningún `Memory.alloc` durante la partida.
- Validación defensiva en los setters de mapa para que un mapa mal definido nunca rompa el juego.
- Respaldo voraz en la IA cuando la heurística no alcanza al jugador.

## Casos de prueba manuales

| Caso | Resultado esperado |
|---|---|
| Moverse contra un muro | El jugador no cambia de celda. |
| Moverse fuera del borde | El jugador no cambia de celda. |
| Recolectar todas las galletas | Avanza al siguiente nivel. Si era el nivel 3, muestra victoria. |
| El enemigo alcanza al jugador | Se pierde una vida y se reinicia el nivel. Si era la última, muestra derrota. |
| Presionar `Q` en partida | Vuelve al menú principal. |
| Presionar `Q` en el menú | Muestra "GRACIAS POR JUGAR" y vuelve al menú con cualquier tecla. |
| Modo Entrenamiento | 5 vidas. El enemigo avanza 1 de cada 3 turnos. |
| Modo Normal | 3 vidas. El enemigo avanza cada paso del jugador. |
| Modo Experto | 1 vida. Dos enemigos simultáneos que se evitan entre sí. |
| Cambiar de modo varias veces seguidas | No se desborda el heap. |
| Jugador rodeado por muros | El enemigo no se congela: el respaldo voraz lo acerca al máximo posible. |
| Galleta accidentalmente sobre muro | El setter defensivo la descarta. El nivel termina con las galletas válidas. |
