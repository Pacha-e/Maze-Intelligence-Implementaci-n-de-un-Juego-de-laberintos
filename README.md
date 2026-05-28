# Maze Intelligence

Juego de laberintos desarrollado en Jack para Nand2Tetris. El jugador debe llegar a la salida mientras un enemigo calcula una ruta por el laberinto y lo persigue.

## Controles

- `WASD` o flechas: mover jugador
- `R`: reiniciar partida
- `Q`: salir
- En el menu inicial:
  - `1`: Entrenamiento, 3 vidas y enemigo lento
  - `2`: Normal, 2 vidas y persecucion completa
  - `3`: Experto, 1 vida y persecucion agresiva

## Como ejecutar

### Opcion web

1. Abrir el Jack Compiler web: <https://nand2tetris.github.io/web-ide/compiler/>.
2. Cargar la carpeta `src` con el boton de carpeta.
3. Presionar `Compile`.
4. Presionar `Run` para pasar al VM Emulator.
5. Subir la velocidad del VM a `Fast`, presionar `Run` y luego `Enable Keyboard`.
6. Elegir un modo con `1`, `2` o `3`.

Captura de prueba: [`docs/vm-emulator-playtest.png`](docs/vm-emulator-playtest.png).

### Opcion local

1. Abrir la carpeta `src` con las herramientas de Nand2Tetris.
2. Compilar los archivos Jack:

```powershell
JackCompiler src
```

3. Abrir `src` en el VM Emulator y ejecutar `Main.main`.

El proyecto usa solo las clases estandar del sistema Jack: `Screen`, `Keyboard`, `Output`, `Array`, `Memory` y `Sys`.

## Estructura

- `Main.jack`: punto de entrada.
- `Game.jack`: ciclo principal, estados, puntaje, modos y vidas.
- `Maze.jack`: matriz del laberinto, colisiones y renderizado del mapa.
- `Player.jack`: movimiento validado del jugador.
- `Enemy.jack`: persecucion con busqueda BFS sobre el grid.

## Criterios cubiertos

- Correctitud logica: movimiento en grid con limites, muros y salida.
- IA del enemigo: busqueda BFS para encontrar la ruta corta hasta el jugador; si no hay ruta, usa una heuristica de respaldo.
- Estados: menu, juego, victoria y derrota.
- Validacion: todas las posiciones pasan por `Maze.canEnter`.
- Modularidad: clases separadas por responsabilidad.
- Renderizado: muros, salida, jugador y enemigo diferenciados en pantalla.
