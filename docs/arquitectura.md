# Arquitectura

## Objetivo

Implementar un juego de laberintos en Jack que sea facil de ejecutar, explicar y mantener. El foco esta en la logica del grid y en el comportamiento del enemigo, porque esos son los criterios con mayor peso en la rubrica (30% correctitud + 20% IA).

## Decisiones principales

- El juego es **por turnos**: cada movimiento valido del jugador dispara la actualizacion de la IA y un redibujado del estado. Esto evita problemas de temporizacion en Jack y mantiene el comportamiento determinista.
- El laberinto se guarda en un `Array` lineal. La posicion `(fila, columna)` se transforma con `fila * cols + columna`, expuesta por `Laberinto.getIndice`.
- El enemigo usa **BFS** para perseguir al jugador por caminos reales, respetando muros y bordes. Si la cola se desborda o no hay ruta, cae a un **respaldo voraz** por distancia Manhattan.
- Los modos cambian la dificultad con parametros simples: vidas, frecuencia de movimiento del enemigo y multiplicador de puntaje.
- El renderizado usa figuras basicas de `Screen` para que funcione en el emulador sin recursos externos. El HUD usa `Output` de texto.
- Toda clase con memoria dinamica (`Array` u objetos) expone `eliminar()` para liberar el heap antes de salir o cambiar de nivel.

## Mapa de clases (`src/`)

| Clase | Responsabilidad |
|---|---|
| `Main` | Punto de entrada. Construye y libera `Juego`. |
| `Juego` | Bucle principal, estados, vidas, puntaje, transiciones entre niveles. |
| `MenuPrincipal` | Pantalla inicial, navegacion por teclado, seleccion de modo. |
| `GestorNiveles` | Fabrica de `Laberinto` por nivel. Define muros, inicios y galletas. |
| `Laberinto` | Matriz del grid, `puedeEntrar`, almacenamiento de `Objetivo`, dibujado. |
| `Jugador` | Posicion + intento de movimiento validado. |
| `Enemigo` | BFS + respaldo voraz, dibujado, retroceder (modo lento). |
| `Objetivo` | Galleta: fila/columna con ciclo de vida independiente. |

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

Las galletas (`Objetivo`) se almacenan aparte en un arreglo dinamico `objetivos`. Cuando el jugador entra en una celda con galleta, esta se elimina por **desplazamiento de punteros** (no por nulificacion), evitando huecos en el arreglo y fugas de memoria.

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
        |       +--> Aumenta puntaje + recoge galletas en la celda
        |       +--> Si conteoObjetivos == 0 -> avanzarNivel
        |       +--> Enemigo.moverHacia (BFS + voraz)
        |               |
        |               +--> Si colisiona con jugador -> perderVida
        |               |
        |               +--> Render completo
        |
        +--> Sys.wait(5)  // estabilidad VM
```

## Decisiones de tamano

- Celdas de 16x16 px, dibujadas como cuadrados de 14x14 (deja 2 px de gap entre celdas).
- Origen del grid: `(10, 42)`. Deja la fila 0 de `Output` libre para el HUD.
- Limite practico: hasta 30 columnas (`10 + 29*16 + 13 = 487 < 511`) y hasta 12 filas (`42 + 11*16 + 13 = 231 < 255`).
- Niveles 1 y 2 son `10 x 12` y `10 x 16`; nivel 3 expande a `11 x 24` para aprovechar la pantalla.
