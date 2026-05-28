# Arquitectura

## Objetivo

Implementar un juego de laberintos en Jack que sea facil de ejecutar, explicar y mantener. El foco esta en la logica del grid y en el comportamiento del enemigo, porque esos son los criterios con mayor peso en la rubrica (30% correctitud + 20% IA).

## Decisiones principales

- El juego es **por turnos**: cada movimiento valido del jugador dispara la actualizacion de la IA y un redibujado del estado. Esto evita problemas de temporizacion en Jack y mantiene el comportamiento determinista.
- El laberinto se guarda en un `Array` lineal. La posicion `(fila, columna)` se transforma con `fila * colsMax + columna`, expuesta por `Laberinto.getIndice`. El stride es `colsMax` (capacidad fisica), no `cols` (dimension logica del nivel actual), para que el mismo arreglo soporte mapas de distintos tamanos sin realocar.
- El enemigo usa una **heuristica de distancia Manhattan combinada con conciencia de muros**: mantiene una frontera de celdas candidatas y en cada iteracion elige la celda con menor distancia Manhattan al jugador. La conciencia de muros viene del filtro previo `Laberinto.puedeEntrar` antes de agregar cualquier vecino: las celdas con muro o fuera del grid nunca entran en la frontera. Asi el enemigo rodea obstaculos sin reconstruir el camino completo y sin recorrer toda la grilla. Tope de 150 expansiones por turno para mantener fluidez en la VM.
- **Respaldo voraz** (`Enemigo.pasoVoraz`): si la heuristica agota las 150 expansiones sin alcanzar al jugador (mapa patologico o jugador rodeado), el enemigo evalua sus 4 vecinos transitables y se mueve al que minimice la distancia Manhattan. Asi el comportamiento es siempre adaptativo: nunca se queda "congelado" frente al jugador.
- **Validacion defensiva** en `Laberinto.setObjetivo / setInicio / setEnemigoInicio / setEnemigo2Inicio`: cada setter rechaza silenciosamente coordenadas en muro o fuera de borde via `puedeEntrar`. Esto garantiza que un mapa mal definido nunca produzca una galleta inalcanzable que congele el nivel ni un actor que arranque atrapado.
- En modo Experto entran dos enemigos simultaneos; el segundo marca la posicion del primero como muro temporal antes de planear su paso, evitando que ambos se fusionen y formando una pinza tactica.
- Los modos cambian la dificultad con parametros simples: vidas, frecuencia de movimiento del enemigo, cantidad de enemigos y multiplicador de puntaje.
- El renderizado usa figuras basicas de `Screen` para que funcione en el emulador sin recursos externos. El HUD usa `Output` de texto.
- **Gestion de heap por pool persistente**: todas las instancias dinamicas (`Laberinto`, `Jugador`, los dos `Enemigo`, los 30 `Objetivo`) se alocan una sola vez al iniciar `Juego`, con capacidad maxima (13x31). Los cambios de nivel y reinicios tras perder vida solo llaman `fijar(f, c)` sobre los objetos existentes, sin disparar `Memory.alloc` / `Memory.deAlloc`. Esto evita la fragmentacion del heap del JackOS que producia un overflow al cambiar de modo varias veces.
- Cada clase con memoria dinamica expone `eliminar()` para liberar todo el heap al cerrar el juego (camino final `Main.main` -> `Juego.eliminar`).

## Mapa de clases (`src/`)

| Clase | Responsabilidad |
|---|---|
| `Main` | Punto de entrada. Construye y libera `Juego`. |
| `Juego` | Bucle principal, estados, vidas, puntaje, transiciones entre niveles. |
| `MenuPrincipal` | Pantalla inicial, navegacion por teclado, seleccion de modo. |
| `GestorNiveles` | Fabrica de mapas. Define 9 layouts unicos (3 modos x 3 niveles): muros, posiciones de inicio y galletas. |
| `Laberinto` | Matriz del grid, `puedeEntrar`, pool de `Objetivo`, dibujado. |
| `Jugador` | Posicion + intento de movimiento validado. |
| `Enemigo` | Heuristica Manhattan + conciencia de muros (filtra por `puedeEntrar`) + fallback voraz cuando la heuristica no alcanza al jugador + variante `moverEvitando` para el modo Experto. |
| `Objetivo` | Galleta: fila/columna reutilizable via pool en `Laberinto`. |

## Estados del juego

- `0`: menu inicial.
- `1`: partida activa.
- `2`: victoria.
- `3`: derrota.

Las transiciones solo ocurren desde `Juego`: elegir modo, llegar a la meta del nivel, perder todas las vidas, reiniciar o salir.

## Representacion del mapa

Valores del arreglo `celdas`:

- `0`: camino libre.
- `1`: muro.

Las galletas (`Objetivo`) se almacenan aparte en un arreglo dinamico `objetivos` de capacidad fija (30 slots). Cuando el jugador entra en una celda con galleta, se elimina por **swap-with-last**: se intercambia el slot recolectado con el ultimo activo y se decrementa el conteo. Esto preserva el pool sin nulificar punteros y sin huecos en el arreglo.

Toda validacion de movimiento pasa por `Laberinto.puedeEntrar(f, c)`, lo que centraliza:

- Bordes (`f < 0`, `c < 0`, `f >= filas`, `c >= cols`).
- Muros (`celdas[idx] = 1`).
- Coherencia entre jugador y enemigo (ambos llaman al mismo metodo).

## Flujo de un turno

```
Tecla -> Juego.manejarCicloJuego
        |
        +--> Jugador.intentarMover (consulta Laberinto.puedeEntrar)
        |       movido = true?
        |       |
        |       +--> Aumenta puntaje + recoge galletas en la celda (swap-with-last)
        |       +--> Si conteoObjetivos == 0 -> avanzarNivel
        |       +--> Enemigo.moverHacia (heuristica Manhattan + puedeEntrar)
        |       +--> Si modo == 3 -> Enemigo2.moverEvitando (misma heuristica
        |                            con posicion de Enemigo1 como muro temporal)
        |       +--> Modo Entrenamiento: si !esMultiplo(movs, 3) -> Enemigo.retroceder
        |       +--> Si colisiona con jugador -> perderVida
        |       +--> Render completo
        |
        +--> Sys.wait(5)  // estabilidad VM
```

## Decisiones de tamano

- Celdas de 16x16 px, dibujadas como cuadrados de 14x14 (deja 2 px de gap entre celdas).
- Origen del grid: `(10, 42)`. Deja la fila 0 de `Output` libre para el HUD.
- Limite practico: hasta 31 columnas (`10 + 30*16 + 13 = 503 < 511`) y hasta 13 filas (`42 + 12*16 + 13 = 247 < 255`).
- Niveles: se usan configuraciones de `11x27`, `13x29` y `13x31` para maximizar la inmersion visual en la pantalla de la VM de Nand2Tetris. El pool fisico siempre es 13x31 = 403 celdas.

## Identidad visual (tema "Cripta del Centinela")

El juego se construye exclusivamente con las primitivas del JackOS sobre la pantalla 1-bit del Hack. La identidad visual se logra con sprites compuestos:

- **Jugador** (`Jugador.dibujar`): cara de 12x12 con dos ojos cuadrados y sonrisa central. Identidad: arqueologo agil y curioso.
- **Enemigo centinela** (`Enemigo.dibujar`, variante 1): domo redondeado superior (`drawCircle`) + cuerpo rectangular + ondas inferiores (lineas blancas que recortan el cuerpo) + dos ojos cuadrados con pupila centrada. Inspirado en los fantasmas clasicos de Pac-Man.
- **Enemigo cazador** (`Enemigo.dibujar`, variante 2 — solo en modo Experto): igual al centinela pero con cuerno superior (`drawLine`) y pupilas rasgadas (rectangulos verticales). Permite distinguirlo del primero de un vistazo.
- **Cristal/objetivo** (`Laberinto.dibujar`): rombo de 4 puntas con centro lleno, mucho mas legible y "valioso" que el cuadradito plano.
- **Muro** (`Laberinto.dibujar`): bloque solido 14x14 con una cruz interna blanca que evoca un circuito impreso. La cruz da textura sin sacrificar legibilidad.
- **Corazon de vida** (`Juego.dibujarCorazon`): pixel-art clasico de dos jorobas + triangulo descendente, 9x9 px, ubicado en la franja del HUD.
- **Corona** (`Juego.dibujarCoronaGrande`): tres picos con joyas blancas, 60x30 px, pantalla de victoria.
- **Sprite gigante del enemigo** (`Juego.dibujarEnemigoGrande`): 50x60 px, pantalla de derrota; conserva la anatomia del centinela en escala.

Decisiones de composicion:

- Las primitivas elegidas son siempre las mas baratas posibles (`drawRectangle` y `drawLine`) para preservar fluidez en la VM. `drawCircle` se usa solo donde aporta caracter (domo del enemigo, sprite grande).
- El contraste se obtiene por reglas 1-bit: cada figura se dibuja en negro y se recortan los detalles internos en blanco con `setColor(false)`. Asi se simulan "huecos" sin tener acceso a un canal alfa.
- Los sprites del menu (mini-jugador y mini-enemigo en las esquinas) reusan la misma anatomia que los sprites de juego para reforzar la identidad: lo que ves en el menu es lo que vas a ver en partida.

## Codigos de tecla (Keyboard.keyPressed)

| Codigo | Tecla | Uso |
|---|---|---|
| 65 / 130 | A / flecha izquierda | mover izquierda |
| 87 / 131 | W / flecha arriba | mover arriba (y navegar menu) |
| 68 / 132 | D / flecha derecha | mover derecha |
| 83 / 133 | S / flecha abajo | mover abajo (y navegar menu) |
| 128 | Enter | confirmar opcion de menu |
| 81 / 113 | Q / q | salir / volver al menu |
| 49..51 | 1, 2, 3 | seleccion directa de modo |
