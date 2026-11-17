# Práctica 4: Amazon Warehouse. Planificar con OMPL y mover estanterías

En esta práctica hay que programar un robot de almacén que recoge estanterías y las deja en otro sitio. Para ello se usa el planificador **OMPL**, que trabaja con posiciones `(x, y)` y orientaciones (`yaw`). Voy a repasar cómo lo he resuelto en tres escenarios, cada uno más realista que el anterior:

1. El robot como un **punto** con los obstáculos expandidos.
2. El robot con su **forma real** (rectángulo), sin expandir los obstáculos.
3. Un robot **Ackermann**, con restricciones de giro y marcha atrás.

---

## 1. Calibración

Antes de planificar nada necesitaba saber cómo se mueve realmente el robot. Escribí un programa de calibración que recorre varios puntos, avanza una distancia fija, gira 90° y muestra la posición medida en cada parada. Así comparo lo que ordeno con lo que ocurre.

```python
import HAL
import Frequency
import math
import time


VELOCIDAD = 0.15
VELOCIDAD_GIRO = 0.3

DISTANCIA = 2
TOLERANCIA = 0.02

GIRO = - math.pi / 2
TOLERANCIA_GIRO = 0.03

TIEMPO_ESPERA = 4


def obtener_pose():

    pose = HAL.getPose3d()

    return pose.x, pose.y, pose.yaw


def mostrar_posicion(nombre):

    x, y, yaw = obtener_pose()

    HAL.setV(0)
    HAL.setW(0)

    print("--------------------------------")
    print(nombre)
    print("X:", round(x, 4), "metros")
    print("Y:", round(y, 4), "metros")
    print("Esperando 4 segundos...")
    print("--------------------------------")

    time.sleep(TIEMPO_ESPERA)


def avanzar(distancia):

    x0, y0, yaw0 = obtener_pose()

    while True:

        x, y, yaw = obtener_pose()

        recorrido = math.sqrt(
            (x - x0)**2 + (y - y0)**2
        )

        if recorrido >= distancia - TOLERANCIA:
            break

        HAL.setV(VELOCIDAD)
        HAL.setW(0)

        Frequency.tick()

    HAL.setV(0)
    HAL.setW(0)


def girar(angulo):

    _, _, yaw0 = obtener_pose()

    objetivo = yaw0 + angulo

    while objetivo > math.pi:
        objetivo -= 2 * math.pi

    while objetivo < -math.pi:
        objetivo += 2 * math.pi

    while True:

        _, _, yaw = obtener_pose()

        error = objetivo - yaw

        while error > math.pi:
            error -= 2 * math.pi

        while error < -math.pi:
            error += 2 * math.pi

        if abs(error) < TOLERANCIA_GIRO:
            break

        HAL.setV(0)
        HAL.setW(
            VELOCIDAD_GIRO if error > 0 else -VELOCIDAD_GIRO
        )

        Frequency.tick()

    HAL.setV(0)
    HAL.setW(0)


# --------------------------------
# CALIBRACION
# --------------------------------

HAL.setV(0)
HAL.setW(0)

mostrar_posicion("PUNTO 1")

avanzar(DISTANCIA)

mostrar_posicion("PUNTO 2")

girar(GIRO)
avanzar(DISTANCIA)

mostrar_posicion("PUNTO 3")

girar(GIRO)
avanzar(DISTANCIA)

mostrar_posicion("PUNTO 4")

avanzar(DISTANCIA)

mostrar_posicion("PUNTO 5")

girar(GIRO)
avanzar(DISTANCIA)

mostrar_posicion("PUNTO 6")

HAL.setV(0)
HAL.setW(0)

print("CALIBRACION TERMINADA")

while True:

    HAL.setV(0)
    HAL.setW(0)

    Frequency.tick()
```

Así queda el recorrido de calibración:

https://github.com/user-attachments/assets/dd037a1e-63e7-46b8-b337-020fba69b047

![Calibración](https://github.com/user-attachments/assets/7b4985ba-1328-4650-bd19-85561b7e09b5)

---

## 2. Primer caso: el robot como un punto (obstáculos expandidos)

La estantería mide **2 metros de largo por 1 de ancho**.

Al coger la estantería hay que liberar el mapa para que el planificador no crea que hay un obstáculo: la estantería que llevo encima es justo el obstáculo que estaba detectando.

### Cómo gestiono el mapa de ocupación

Cada vez que el robot planifica, **rehago el mapa de ocupación desde cero** a partir del mapa original y le aplico lo que ha cambiado en el almacén:

- Las estanterías que he recogido las **vacío**.
- Las que he dejado las **marco como obstáculo**.

Las zonas vaciadas se van acumulando en una lista, así que no se pierden de una planificación a otra. Lo mismo ocurre con las patas de las estanterías que ya he dejado.

Lo que cambia es cómo marco las estanterías dejadas según si el robot lleva una encima o no:

- **Robot libre:** solo marco las patas, porque puede pasar por debajo.
- **Robot cargado:** marco la estantería entera, porque no puede pasar por en medio.

No necesito guardar dos mapas: construyo uno u otro según el estado. En el debug dibujo siempre solo las patas para que se vea limpio, aunque el bloque entero sí cuenta para planificar.

Además, la estantería la dejo **girada 90°** respecto a cómo la recojo, así que planifico ya con ese ángulo final y el robot gira en el sitio antes de bajarla.

**Limitaciones conocidas:**

- No controlo que ese giro en el sitio choque con algo; solo compruebo que la posición final es válida.
- Si dejo una estantería muy pegada a otra, el margen de seguridad con el robot cargado puede hacer que no encuentre camino.

Para depurar qué zona liberaba y cuál ocupaba usé `showNumpy` para dibujar sobre el mapa:

![Depuración del mapa de ocupación](https://github.com/user-attachments/assets/9b487b0c-af2e-4a05-8959-94eeabd8c5b6)

### Entender el yaw del planificador

El planificador OMPL trabaja con posiciones `(X, Y)` y orientaciones (`yaw`), así que cada punto de la trayectoria lleva asociada una orientación.

Ese yaw es la orientación que debe tener el robot en cada punto y va cambiando progresivamente desde la orientación inicial hasta la definida en el objetivo. Pero eso **no significa** que la orientación de cada punto coincida con la dirección hacia el siguiente: el robot puede avanzar en una dirección mientras mantiene una orientación distinta.

Por tanto, el yaw del planificador se entiende como una **transición de ángulos** a lo largo de la trayectoria, cuyo valor final sí está condicionado por la orientación del objetivo.

### Problemas que me he encontrado

- **Entender el planificador.**
- **Darme cuenta de que podía darle un ángulo objetivo al planner**, para que llegue ya con esa orientación en vez de llegar y luego girar.
- **El camino nunca es el más óptimo.**
- **Inflado de obstáculos.**
- **Usar solo `x` e `y`** del planner al principio y no el ángulo.
- **Mirar puntos más adelante** hace que también tome el ángulo de más adelante, y a veces el robot se perdía. La solución fue usar un **VFF** (campo de fuerzas virtual) que combina el ángulo hasta el punto con el ángulo que da ese punto.
- **Bucles con el punto más cercano:** al mirar puntos adelantados y el más cercano, a veces cogía el punto anterior y entraba en un bucle. La solución fue forzar que siempre coja puntos de **índice superior**.

---

## 3. Segundo caso: el robot con su forma real, sin expandir

Para que el planificador tenga en cuenta el tamaño y la forma del robot se utiliza una función de comprobación de colisiones, el **State Validity Checker** de OMPL.

El planificador trabaja en un espacio de estados **SE2**: cada estado es la posición `(x, y)` y la orientación (`yaw`). En lugar de pasarle a OMPL una figura geométrica, compruebo si el robot puede ocupar cada posición que va generando el planificador:

1. Defino una **huella rectangular** con las dimensiones del robot, que cambian según lleve estantería o no.
2. A partir de la posición y la orientación calculo las **cuatro esquinas** del rectángulo y las paso de coordenadas del mundo a píxeles del mapa.
3. Genero una **máscara** del rectángulo con `cv2.fillConvexPoly`.
4. Compruebo si algún píxel de la máscara coincide con una zona ocupada.

Si alguna parte del rectángulo cae dentro de un obstáculo o fuera de los límites del mapa, la posición **no es válida**; en caso contrario, sí lo es. Así OMPL planifica con las dimensiones reales del robot en lugar de tratarlo como un punto.

Por último, reduzco el engorde de las paredes a **0**, porque ya no hace falta inflar el mapa.

---

## 4. Robot Ackermann

Con un robot Ackermann cambia el espacio de estados: uso **Dubins State Space**. El problema es el giro, ya que un Ackermann **no puede girar sobre sí mismo**.

Dubins necesita comprobar colisiones con un rectángulo orientado (**OBB**), así que implementé esa comprobación:

```python
"""
Comprobacion de colision con un rectangulo orientado (OBB).

El robot es un rectangulo de largo_robot x ancho_robot (mas el margen
de seguridad) centrado en (x, y) y girado yaw + giro_huella. Cada pixel
ocupado se trata como el cuadrado que realmente es (de lado 1 pixel,
centrado en su indice) y se comprueba su solape exacto con el
rectangulo mediante el teorema del eje separador (SAT).

No se redondean las esquinas del rectangulo a pixeles ni se usa ninguna
circunferencia, asi que no hay falsos positivos por exceso de
conservadurismo ni falsos negativos por redondeo.
"""
```

### Marcha atrás

El Ackermann puede retroceder, así que el planificador no se limita a caminos que solo van hacia delante. Usa un espacio de curvas que admite **los dos sentidos**, con el mismo radio de giro mínimo en ambos. Gracias a eso puede resolver situaciones que hacia delante serían imposibles o muy largas, como salir de un hueco estrecho o colocarse en la orientación justa para recoger una estantería sin dar una vuelta enorme.

**Deducir el sentido de la marcha.** El camino que devuelve el planificador solo guarda posición y orientación en cada punto, no si el robot va hacia delante o hacia atrás. Ese sentido se deduce después: en cada paso se compara hacia dónde se desplaza el robot con hacia dónde mira.

- Si el desplazamiento va en la misma dirección que la orientación, **avanza**.
- Si va en la contraria, **retrocede**.

**Tramos de sentido constante.** Con esa información se parte la trayectoria en tramos. Cada vez que el sentido cambia hay un **cambio de marcha**, y ese punto es el final de un tramo y el principio del siguiente.

**Seguimiento de cada tramo.** Se sigue con la misma idea de siempre: elegir un punto del camino un poco por delante y calcular el arco que lleva hasta él.

- En un tramo de **marcha atrás** se hace el mismo cálculo imaginando que el robot mira hacia el lado contrario, y la **velocidad lineal enviada es negativa**.
- La **velocidad angular** sale con el mismo cálculo, porque el giro del robot es el mismo mirado desde cualquiera de los dos lados.

**Cambios de marcha.** Al llegar al final de un tramo intermedio, el robot se detiene un momento, como haría un coche al cambiar de marcha, y empieza el siguiente tramo en el sentido contrario. Si es el último tramo, se comprueba que ha llegado a la posición y orientación del objetivo; si se ha pasado sin llegar, **se replanifica desde donde esté**.

---

## Conclusiones

- Rehacer el mapa de ocupación en cada planificación es más simple y robusto que mantener varios mapas.
- Darle al planner un **ángulo objetivo** evita giros innecesarios al llegar.
- Un *State Validity Checker* con la huella real del robot permite planificar sin inflar obstáculos.
- En Ackermann, Dubins + OBB + marcha atrás permiten maniobras que hacia delante serían inviables, a costa de gestionar los cambios de marcha.
