# Maze Intelligence

**Cripta del Centinela** — Juego de laberintos desarrollado en Jack para Nand2Tetris. Encarnas a un arqueologo-digital que se infiltra en una cripta-circuito para robar los cristales de datos. Uno o dos centinelas-fantasma, guiados por una IA con **heuristica Manhattan y conciencia de muros**, te persiguen por los caminos validos del mapa. Estetica arcade-retro 1-bit, inspirada en Pac-Man y los clasicos monocromaticos del ZX Spectrum y el Game Boy.

## Controles

- `W` `A` `S` `D` o flechas: mover al jugador.
- `Q` durante la partida: regresar al menu principal.
- En el menu inicial:
  - `1`: Modo Entrenamiento.
  - `2`: Modo Normal.
  - `3`: Modo Experto.
  - Flechas Arriba/Abajo + Enter para navegar el menu.
  - `Q` desde el menu: muestra la pantalla de despedida.

## Modos de dificultad

| Modo | Vidas | Enemigos | Comportamiento | Multiplicador puntaje |
|---|---|---|---|---|
| 1 - Entrenamiento | 5 | 1 (lento, 1 de cada 3 turnos) | persecucion intermitente | x1 |
| 2 - Normal | 3 | 1 (paso a paso) | persecucion estandar | x2 |
| 3 - Experto | 1 | 2 (simultaneos, evitan colisionar entre si) | pinza tactica | x3 |

Cada modo recorre 3 niveles propios (9 mapas unicos en total) con dimensiones que crecen hasta 13x31 celdas para aprovechar toda la pantalla del VM Emulator.

## Como ejecutar

### Opcion web (recomendada)

1. Abrir el Jack Compiler web: <https://nand2tetris.github.io/web-ide/compiler/>.
2. Cargar la carpeta `src` con el boton de carpeta.
3. Presionar `Compile`.
4. Presionar `Run` para pasar al VM Emulator.
5. Subir la velocidad del VM a `Fast`, presionar `Run` y luego `Enable Keyboard`.
6. Elegir un modo con `1`, `2`, `3` o con las flechas + Enter.

### Opcion local

1. Abrir la carpeta `src` con las herramientas de Nand2Tetris.
2. Compilar los archivos Jack:

```powershell
JackCompiler src
```

3. Abrir `src` en el VM Emulator y ejecutar `Main.main`.

El proyecto usa solo las clases estandar del sistema Jack: `Screen`, `Keyboard`, `Output`, `Array`, `Memory` y `Sys`.

## Estructura del codigo (`src/`)

- `Main.jack`: punto de entrada. Instancia `Juego`, ejecuta el bucle y libera memoria.
- `Juego.jack`: orquestador. Maneja estados (menu/partida/victoria/derrota), modo de dificultad, vidas, puntaje y transiciones entre niveles.
- `MenuPrincipal.jack`: interfaz retro del menu inicial con navegacion por flechas y seleccion directa por tecla.
- `GestorNiveles.jack`: fabrica de mapas. Construye los 9 layouts (3 modos x 3 niveles) con muros, inicios y galletas.
- `Laberinto.jack`: matriz del mapa. Centraliza muros, validacion de movimientos (`puedeEntrar`), almacenamiento de galletas y renderizado del grid. Pre-aloca un pool de `Objetivo` para evitar fragmentacion del heap.
- `Jugador.jack`: posicion del heroe y movimiento validado contra el laberinto.
- `Enemigo.jack`: IA con heuristica Manhattan y conciencia de muros. En cada paso elige de su frontera la celda con menor distancia Manhattan al jugador y solo expande vecinos transitables (filtrados por `Laberinto.puedeEntrar`). Incluye un respaldo voraz que da un paso adaptativo cuando la heuristica no alcanza al jugador en 150 expansiones. En modo Experto, el segundo enemigo recibe la posicion del primero y la marca como visitada para no fusionarse.
- `Objetivo.jack`: representa una galleta (par fila/columna), reutilizable via el pool de `Laberinto`.

## Identidad visual

El juego usa exclusivamente las primitivas graficas del JackOS (`drawRectangle`, `drawLine`, `drawCircle`, `drawPixel`) sobre la pantalla monocromatica 512x256 del Hack. La identidad visual se construye con sprites compuestos pixel-art:

| Elemento | Forma | Tamano | Identidad |
|---|---|---|---|
| Jugador | Cabeza con ojos cuadrados y sonrisa | 12x12 px | Curiosidad, agilidad — el arqueologo |
| Centinela (modo 1, 2 y 3) | Domo redondeado + ondas inferiores + ojos | 12x14 px | Patrullero clasico estilo Pac-Man ghost |
| Cazador (modo Experto, 2do enemigo) | Centinela + cuerno superior + pupilas rasgadas | 12x16 px | Especialista, diferenciado del centinela comun |
| Cristal (objetivo) | Rombo de 4 puntas con centro lleno | 10x10 px | Reliquia valiosa, lectura inmediata |
| Muro | Bloque solido 14x14 con cruz interna blanca | 14x14 px | Bloque-circuito, textura sin perder legibilidad |
| Corazon (vida) | Pixel-art clasico de dos jorobas + triangulo | 9x9 px | Indicador universal en el HUD |
| Corona (victoria) | Tres picos con joyas blancas | 60x30 px | Trofeo grande en la pantalla final |

### Pantallas

- **Menu principal**: marco triple con caja de titulo, mision, opciones con corchetes `>>> [ N ] MODO <<<` y sprites de jugador/enemigo en las cuatro esquinas.
- **Partida**: HUD superior con nivel + puntaje a la izquierda y corazones de vida a la derecha; grid de muros, cristales, jugador y centinela(s).
- **Victoria**: marco doble concentrico (efecto trofeo) + corona gigante + puntaje final.
- **Derrota**: marco solido (efecto lapida) + sprite gigante del centinela cazandote + puntaje final.

### Referencias de diseno

- [The Art of 1 Bit Game Development](https://bandurart.com/the-art-of-1-bit-game-development/) — guias de composicion 1-bit y uso de patrones.
- [Pac-Man Ghost Gang](https://pixeljoint.com/pixelart/84514.htm) — anatomia de sprites tipo fantasma y diferenciacion por detalles.
- [Sprite techniques en Jack/Nand2Tetris](https://github.com/encse/nand2tetris/blob/master/pong/Screen.jack) — primitivas Screen y patrones erase-then-redraw.

## Decisiones tecnicas clave

- **Pre-alocacion total**: `Juego` instancia `Laberinto`, `Jugador` y los dos `Enemigo` una sola vez al arrancar, con la capacidad maxima (13x31 = 403 celdas). Los cambios de nivel y reinicios tras perder vida solo reposicionan los objetos via `fijar()`, sin llamar a `Memory.alloc` / `Memory.deAlloc`. Esto evita el desbordamiento del heap del JackOS reportado al cambiar de modo varias veces.
- **Pool de objetivos**: `Laberinto` reserva un arreglo de 30 `Objetivo` al construirse. `setObjetivo` reposiciona un slot libre del pool; `removerObjetivo` aplica swap-with-last para mantener el arreglo compacto sin huecos.
- **Movimiento por turnos**: cada movimiento valido del jugador dispara la actualizacion de la IA y un redibujado del estado. Esto evita problemas de temporizacion y mantiene el comportamiento determinista.
- **Validacion centralizada y defensiva**: toda posicion candidata pasa por `Laberinto.puedeEntrar(f, c)` (bordes + muros), tanto para el jugador como para la IA. Adicionalmente, los setters de mapa (`setObjetivo`, `setInicio`, `setEnemigoInicio`, `setEnemigo2Inicio`) descartan silenciosamente coordenadas invalidas como capa redundante para que un mapa mal definido nunca produzca galletas inalcanzables o actores atrapados.

## Criterios cubiertos (segun rubrica)

- **Correctitud logica:** movimiento valido en el grid, deteccion de colisiones contra muros y bordes, captura del jugador, recoleccion de galletas, fin de nivel al recolectar todas, fin de juego al perder todas las vidas.
- **IA del enemigo:** heuristica informada por distancia Manhattan combinada con conciencia de muros. La frontera nunca recibe celdas con muro ni fuera de bordes (filtro previo via `puedeEntrar`), y entre las que quedan se elige siempre la mas cercana al jugador. Esto produce persecucion que rodea obstaculos sin recorrer toda la grilla. En modo Experto, el segundo enemigo marca la posicion del primero como muro temporal para no fusionarse, formando una pinza tactica.
- **Manejo de estados:** menu (0), partida (1), victoria (2), derrota (3). Transiciones controladas exclusivamente desde `Juego`.
- **Validacion de movimientos:** toda posicion pasa por `Laberinto.puedeEntrar`, que cubre bordes, muros y posiciones invalidas.
- **Modularidad:** ocho clases con responsabilidades claras y bajo acoplamiento.
- **Clean Code:** nombres en espanol, metodos cortos, sin redundancia, gestion explicita del heap (pool persistente + `eliminar()` al cerrar).
- **Renderizado:** muros, galletas, jugador y enemigos diferenciados por forma y tamano; HUD con vidas, nivel y puntaje.
