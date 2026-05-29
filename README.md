# Maze Intelligence

Juego de laberintos escrito en Jack para Nand2Tetris. El jugador debe recolectar los cristales del mapa mientras uno o dos centinelas lo persiguen.

Proyecto final de Organización de Computadores, Universidad EAFIT, 2026.

## Controles

- `W` `A` `S` `D` o flechas: mover al jugador.
- `Q` durante la partida: volver al menú.
- En el menú: `1`, `2`, `3` para elegir modo, o flechas + Enter.

## Modos

| Modo | Vidas | Enemigos | Multiplicador |
|---|---|---|---|
| Entrenamiento | 5 | 1 (avanza cada 3 turnos) | x1 |
| Normal | 3 | 1 | x2 |
| Experto | 1 | 2 (se evitan entre sí) | x3 |

Cada modo tiene 3 niveles distintos. En total son 9 mapas, con dimensiones de hasta 13x31 celdas.

## Cómo ejecutar

### En el navegador

1. Abrir <https://nand2tetris.github.io/web-ide/compiler/>.
2. Cargar la carpeta `src`.
3. Presionar **Compile**.
4. Pasar al VM Emulator, subir la velocidad a **Fast** y presionar **Run**.
5. Habilitar el teclado con **Enable Keyboard**.

### En local

1. Compilar con `JackCompiler src`.
2. Abrir la carpeta en el VM Emulator y ejecutar `Main.main`.

El proyecto usa únicamente las clases del sistema Jack: `Screen`, `Keyboard`, `Output`, `Array`, `Memory` y `Sys`.

## Estructura del código

```
src/
├── Main.jack            Punto de entrada
├── Juego.jack           Bucle principal, estados, transiciones
├── MenuPrincipal.jack   Pantalla inicial y selección de modo
├── GestorNiveles.jack   Generador de los 9 mapas
├── Laberinto.jack       Matriz del grid, muros y pool de objetivos
├── Jugador.jack         Posición y movimiento del héroe
├── Enemigo.jack         IA de persecución
├── Objetivo.jack        Galleta del laberinto
├── Hud.jack             Panel superior con nivel, puntaje y vidas
└── PantallaFinal.jack   Pantallas de victoria y derrota
```

## Cómo funciona la IA

El enemigo no se mueve al azar. En cada turno explora una frontera de celdas alcanzables y elige siempre la más cercana al jugador según distancia Manhattan. Al expandir vecinos solo acepta los que pasen la validación de muros y bordes (`Laberinto.puedeEntrar`), por lo que rodea los obstáculos sin recorrer todo el mapa.

Si en 150 expansiones no logra trazar el camino completo (jugador muy lejos o rodeado), el enemigo cae a un respaldo voraz: evalúa sus 4 vecinos transitables y se mueve al que mejore más la distancia. Así nunca se queda quieto frente al jugador.

En modo Experto entran dos enemigos. El segundo recibe la posición del primero como muro temporal antes de planear su paso, lo que evita que se fusionen y forma una pinza táctica.

## Documentación adicional

- `docs/arquitectura.md`: decisiones de diseño y mapa de clases.
- `docs/sustentacion.md`: guía para la sustentación oral y casos de prueba.
