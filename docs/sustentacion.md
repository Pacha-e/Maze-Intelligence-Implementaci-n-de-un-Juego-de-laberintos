# Guia de sustentacion

## Idea central

El juego representa el laberinto como una matriz. Cada celda puede ser camino, muro o salida. El jugador solo puede moverse a celdas validas y el enemigo recalcula una ruta hacia el jugador despues de cada movimiento.

## Puntos tecnicos importantes

- `Maze` centraliza la matriz, los muros, la salida y la validacion de posiciones.
- `Player` solo conoce su posicion y delega la validacion al laberinto.
- `Enemy` usa BFS, una busqueda por amplitud, para encontrar el primer paso de la ruta mas corta hacia el jugador.
- `Game` controla estados, dificultad, vidas, puntaje y transiciones.
- El juego evita movimiento aleatorio: el enemigo siempre decide con base en el estado actual del tablero.

## Como explicar la IA

1. El enemigo empieza desde su posicion actual.
2. Marca la celda inicial como visitada.
3. Explora vecinos validos en cuatro direcciones: arriba, abajo, izquierda y derecha.
4. Guarda para cada celda cual fue el primer paso desde el enemigo.
5. Cuando encuentra al jugador, se mueve a ese primer paso.

Esta estrategia respeta muros y bordes, por eso es mas robusta que comparar solo la distancia Manhattan.

## Mejoras agregadas

- Tres niveles con mapas diferentes.
- Tres modos de dificultad.
- Vidas segun el modo.
- Puntaje por movimiento y por completar niveles.
- Pantallas de menu, victoria y derrota.

## Casos de prueba manuales

- Intentar moverse contra un muro: el jugador no cambia de celda.
- Intentar salir del borde: el jugador no cambia de celda.
- Llegar a la salida: avanza de nivel o muestra victoria en el nivel 3.
- Dejar que el enemigo alcance al jugador: resta una vida o muestra derrota.
- Presionar `R`: reinicia la partida con las vidas del modo actual.
- Presionar `Q`: sale del juego.
