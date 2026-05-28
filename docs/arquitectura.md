# Arquitectura

## Objetivo

Implementar un juego de laberintos en Jack que sea facil de ejecutar, explicar y mantener. El foco esta en la logica del grid y en el comportamiento del enemigo, porque esos son los criterios con mayor peso en la rubrica.

## Decisiones principales

- El juego es por turnos: cada movimiento valido del jugador actualiza la IA y vuelve a dibujar el estado. Esto reduce errores de temporizacion en Jack.
- El laberinto se guarda en un `Array` lineal. La posicion `(fila, columna)` se transforma con `fila * columnas + columna`.
- El enemigo usa BFS para perseguir al jugador por caminos reales, respetando muros y limites.
- Los modos cambian dificultad con parametros simples: vidas, frecuencia de movimiento y puntaje.
- El renderizado usa figuras basicas de `Screen` para que funcione en el emulador sin recursos externos.

## Estados del juego

- `0`: menu inicial.
- `1`: partida activa.
- `2`: victoria.
- `3`: derrota.

Las transiciones solo ocurren desde `Game`: elegir modo, llegar a la meta, perder vidas, reiniciar o salir.

## Representacion del mapa

Valores del arreglo:

- `0`: camino libre.
- `1`: muro.
- `2`: salida.

Toda validacion de movimiento pasa por `Maze.canEnter`, lo que centraliza bordes, muros y posiciones invalidas.
