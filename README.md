# Maze Intelligence

Juego de laberintos desarrollado en Jack para Nand2Tetris. El jugador debe recolectar todas las galletas de cada nivel mientras un enemigo controlado por IA (BFS sobre el grid) lo persigue por los caminos validos del mapa.

## Controles

- `W` `A` `S` `D` o flechas: mover al jugador.
- `Q` durante la partida: regresar al menu principal.
- En el menu inicial:
  - `1`: Modo Entrenamiento, 3 vidas y enemigo lento.
  - `2`: Modo Normal, 2 vidas y persecucion estandar.
  - `3`: Modo Experto, 1 vida y persecucion agresiva (doble movimiento).
  - Flechas Arriba/Abajo + Enter para navegar el menu.
  - `Q` desde el menu: cierra el juego.

## Como ejecutar

### Opcion web (recomendada)

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

## Estructura del codigo (`src/`)

- `Main.jack`: punto de entrada. Instancia `Juego` y arranca el ciclo principal.
- `Juego.jack`: orquestador. Maneja estados (menu/partida/victoria/derrota), modo de dificultad, vidas, puntaje y transiciones entre niveles.
- `MenuPrincipal.jack`: interfaz retro del menu inicial con navegacion por flechas y seleccion directa por tecla.
- `GestorNiveles.jack`: fabrica de mapas. Construye los `Laberinto` de cada nivel (1, 2 y 3) con muros, inicios y galletas.
- `Laberinto.jack`: matriz del mapa. Centraliza muros, validacion de movimientos (`puedeEntrar`), almacenamiento de galletas y renderizado del grid.
- `Jugador.jack`: posicion del heroe y movimiento validado contra el laberinto.
- `Enemigo.jack`: IA. Implementa BFS sobre el grid con respaldo voraz (distancia Manhattan) cuando no hay ruta.
- `Objetivo.jack`: representa una galleta (par fila/columna con su propio ciclo de vida en heap).

## Criterios cubiertos (segun rubrica)

- **Correctitud logica:** movimiento valido en el grid, deteccion de colisiones contra muros y bordes, captura del jugador, recoleccion de galletas, fin de nivel al recolectar todas, fin de juego al perder todas las vidas.
- **IA del enemigo:** BFS para hallar el primer paso de la ruta mas corta hasta el jugador respetando muros; respaldo heuristico voraz por distancia Manhattan cuando la cola se llena o no hay ruta.
- **Manejo de estados:** menu (0), partida (1), victoria (2), derrota (3). Transiciones controladas exclusivamente desde `Juego`.
- **Validacion de movimientos:** toda posicion pasa por `Laberinto.puedeEntrar`, que cubre bordes, muros y posiciones invalidas.
- **Modularidad:** ocho clases con responsabilidades claras y bajo acoplamiento.
- **Clean Code:** nombres en espanol, metodos cortos, sin redundancia, todos los objetos liberan memoria con `eliminar()`.
- **Renderizado:** muros, galletas, jugador y enemigo diferenciados por forma y tamano; HUD con vidas, nivel y puntaje.
