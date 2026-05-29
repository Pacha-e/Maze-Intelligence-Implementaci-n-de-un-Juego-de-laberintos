# Arquitectura

## Idea general

El juego es por turnos. Cada movimiento válido del jugador dispara la actualización de los enemigos y un redibujado completo de la pantalla. Esto simplifica la temporización en Jack y hace que el comportamiento sea determinista.

El laberinto se guarda en un arreglo lineal de tamaño `filasMax * colsMax`. Para acceder a la celda `(fila, columna)` se usa el índice `fila * colsMax + columna`. El stride siempre es `colsMax`, la capacidad física, para que el mismo arreglo soporte mapas de distinto tamaño sin tener que reservar memoria nueva.

## Clases

| Clase | Responsabilidad |
|---|---|
| `Main` | Punto de entrada. Construye y libera `Juego`. |
| `Juego` | Bucle principal, estados, vidas, puntaje, transiciones entre niveles. |
| `MenuPrincipal` | Pantalla inicial, navegación por teclado, selección de modo. |
| `GestorNiveles` | Define los 9 mapas del juego: muros, posiciones iniciales y galletas. |
| `Laberinto` | Matriz del grid, validación `puedeEntrar`, pool de objetivos, dibujo de muros y galletas. |
| `Jugador` | Posición del héroe e intento de movimiento validado. |
| `Enemigo` | Heurística Manhattan con conciencia de muros y respaldo voraz. |
| `Objetivo` | Galleta del mapa: par fila/columna. |
| `Hud` | Panel superior con nivel, puntaje y corazones de vida. |
| `PantallaFinal` | Pantallas de victoria y derrota con sus sprites grandes. |

Cada clase tiene una responsabilidad clara. `Juego` orquesta el flujo y delega el dibujo al `Hud` y a `PantallaFinal`, así no mezcla lógica con presentación.

## Estados del juego

- `0`: menú inicial.
- `1`: partida activa.
- `2`: victoria.
- `3`: derrota.

Las transiciones solo ocurren desde `Juego`: elegir un modo, terminar el nivel 3, perder todas las vidas o presionar `Q` para volver al menú.

## Representación del mapa

El arreglo `celdas` guarda valores enteros: `0` es camino libre y `1` es muro. Las galletas se almacenan aparte en un arreglo de `Objetivo` con capacidad fija de 30 slots.

Cuando el jugador recoge una galleta se aplica swap-with-last: el slot recolectado se intercambia con el último activo y se decrementa el contador. Así el pool se mantiene compacto y sin huecos.

Toda validación de movimiento pasa por `Laberinto.puedeEntrar(f, c)`, que cubre tres casos:

- Coordenadas fuera del grid (`f < 0`, `c < 0`, `f >= filas`, `c >= cols`).
- Celdas marcadas como muro (`celdas[idx] = 1`).
- Coherencia: el jugador y los enemigos usan el mismo método.

## Flujo de un turno

1. `Juego.manejarCicloJuego` lee la tecla.
2. Si es una dirección, `Jugador.intentarMover` valida contra `Laberinto.puedeEntrar`.
3. Si el jugador se movió:
   - Se suma puntaje y se recogen las galletas que estén en la nueva celda.
   - Si no quedan galletas, avanza al siguiente nivel.
   - El enemigo (o los dos en Experto) calcula su paso.
   - En modo Entrenamiento, el enemigo retrocede si el conteo de movimientos no es múltiplo de 3.
   - Si hay colisión con el jugador, se pierde una vida.
   - Se redibuja el estado completo.

## Gestión de memoria

Todos los objetos dinámicos se reservan una sola vez al inicio del juego, con la capacidad máxima (13x31 = 403 celdas):

- `Laberinto` (con su arreglo de celdas y su pool de 30 objetivos)
- `Jugador`
- Los dos `Enemigo` (cada uno con sus 5 arreglos de búsqueda)
- `Hud`, `PantallaFinal`, `MenuPrincipal`, `GestorNiveles`

Cambiar de nivel o reiniciar tras perder una vida solo llama a `fijar(f, c)` sobre los objetos existentes. No se llama nunca a `Memory.alloc` ni `Memory.deAlloc` durante la partida.

Esto evita el desbordamiento del heap del JackOS que aparecía al cambiar de modo varias veces seguidas. Cada clase con memoria dinámica expone un método `eliminar()` que libera el heap al cerrar el juego.

## Validación defensiva

Los setters de mapa (`setInicio`, `setEnemigoInicio`, `setEnemigo2Inicio`, `setObjetivo`) descartan silenciosamente cualquier coordenada que no pase `puedeEntrar`. Es una segunda capa sobre la validación de movimiento que protege al juego contra mapas mal definidos: nunca se coloca una galleta dentro de un muro ni un actor empieza atrapado.

## Renderizado

El dibujado usa solo las primitivas del JackOS sobre la pantalla 512x256 del Hack: `drawRectangle`, `drawLine`, `drawCircle` y `drawPixel`. El HUD textual usa `Output.printString` y `Output.printInt`.

Las celdas son de 16x16 px y los sprites de 12x12 o 12x14, lo que deja un margen de 2 px entre celdas. El origen del grid es `(10, 42)`, dejando la fila 0 de `Output` libre para el HUD.

| Elemento | Tamaño | Construcción |
|---|---|---|
| Muro | 14x14 | Bloque negro con cruz blanca interna |
| Jugador | 12x12 | Cabeza con dos ojos cuadrados y sonrisa |
| Centinela | 12x14 | Domo + cuerpo + ondas inferiores + dos ojos |
| Cazador (Experto) | 12x16 | Centinela + cuerno superior + pupilas rasgadas |
| Cristal | 10x10 | Rombo de 4 puntas con centro lleno |
| Corazón (vida) | 9x9 | Dos jorobas + triángulo |
| Corona (victoria) | 60x30 | Tres picos con joyas |

Cada sprite se construye alternando `setColor(true)` y `setColor(false)` para simular huecos en la pantalla monocromática.

## Códigos de tecla

| Código | Tecla | Uso |
|---|---|---|
| 65 / 130 | A / izquierda | Mover izquierda |
| 87 / 131 | W / arriba | Mover arriba (y navegar menú) |
| 68 / 132 | D / derecha | Mover derecha |
| 83 / 133 | S / abajo | Mover abajo (y navegar menú) |
| 128 | Enter | Confirmar opción del menú |
| 81 / 113 | Q / q | Salir o volver al menú |
| 49..51 | 1, 2, 3 | Selección directa de modo |
