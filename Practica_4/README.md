# Práctica 4: Amazon Warehouse.

En esta práctica hay que programar un robot de almacén que recoge estanterías y las deja en otro sitio. Para ello se usa el planificador **OMPL**, que trabaja con posiciones `(x, y)` y orientaciones (`yaw`). Voy a repasar cómo lo he resuelto en tres escenarios, cada uno más realista que el anterior:

1. El robot como un **punto** con los obstáculos expandidos.
2. El robot con su **forma real** (rectángulo), sin expandir los obstáculos.
3. Un robot **Ackermann**, con restricciones de giro y marcha atrás.

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

Una vez tengo los puntos de referencia, los comparo con el mapa utilizando GIMP para obtener los píxeles correspondientes a cada coordenada. De esta forma puedo relacionar las coordenadas del mundo con las coordenadas de la imagen y obtener los datos necesarios para realizar la calibración. A partir de estos puntos calculo la transformación que permite pasar de un sistema de coordenadas a otro.

![Calibración](https://github.com/user-attachments/assets/7b4985ba-1328-4650-bd19-85561b7e09b5)

Así queda el recorrido de calibración, que utilizo para comprobar que la correspondencia entre ambos sistemas es correcta:

https://github.com/user-attachments/assets/dd037a1e-63e7-46b8-b337-020fba69b047

## 2. Primer caso: el robot como un punto (obstáculos expandidos)

Para el primer caso trato al robot como si fuese un punto y expando los obstáculos del mapa para tener en cuenta sus dimensiones. La estantería que tiene que transportar mide 2 metros de largo por 0,88 metros de ancho.

Cuando el robot recoge una estantería, tengo que liberar esa zona del mapa, ya que la estantería que lleva encima es precisamente el obstáculo que estaba detectando el planificador. Si no lo hiciese, el robot consideraría que tiene un obstáculo dentro de su propia posición y no podría planificar correctamente.

Para gestionar el mapa de ocupación, cada vez que el robot necesita planificar vuelvo a generarlo desde cero a partir del mapa original. Sobre este mapa aplico los cambios que se han producido durante la ejecución: las estanterías que he recogido se vacían y las que he dejado en otras posiciones se marcan como obstáculos. Las zonas que voy vaciando se guardan en una lista, de manera que se mantienen entre las distintas planificaciones y no vuelven a aparecer como obstáculos. Lo mismo hago con las patas de las estanterías que he ido dejando.

La forma de representar las estanterías que he dejado depende de si el robot lleva una encima o no. Cuando el robot está libre, únicamente marco las patas de las estanterías, ya que puede pasar por debajo de ellas. Sin embargo, cuando lleva una estantería, marco toda su superficie como obstáculo, porque ya no puede pasar por debajo. Por tanto, no necesito mantener dos mapas diferentes, sino que construyo el mapa correspondiente en función del estado del robot en cada planificación. En la representación de depuración dibujo siempre únicamente las patas para que el mapa se vea más limpio, aunque cuando el robot está cargado toda la superficie de la estantería sí se tiene en cuenta para comprobar las colisiones.

Además, cuando dejo una estantería la coloco girada 90 grados respecto a la orientación en la que la recojo. Por eso, directamente le doy al planificador la orientación final que quiero conseguir. De esta manera, el robot llega a la posición con la orientación adecuada y únicamente tiene que girar sobre sí mismo antes de bajar la estantería.

Hay algunas limitaciones en este planteamiento. Por ejemplo, no compruebo si durante ese giro sobre sí mismo el robot puede chocar con algún obstáculo, sino que únicamente compruebo que la posición y orientación finales sean válidas. También puede ocurrir que, si dejo una estantería demasiado cerca de otra, el margen de seguridad que utilizo cuando el robot está cargado impida encontrar un camino válido.

Para comprobar que estaba liberando y ocupando correctamente las zonas del mapa utilicé la función `showNumpy`, que me permite dibujar sobre la imagen del mapa y visualizar cómo se estaba modificando el mapa de ocupación.

### Entender el yaw del planificador

Una de las partes que más me costó entender fue el funcionamiento del yaw del planificador. OMPL trabaja con un espacio de estados SE2, por lo que cada estado de la trayectoria contiene tres valores: la posición X, la posición Y y la orientación del robot (yaw). Al principio utilizaba únicamente las coordenadas X e Y, sin tener en cuenta la orientación que devolvía el planificador.

El yaw representa la orientación que debe tener el robot en cada punto de la trayectoria. Esta orientación va cambiando desde la orientación inicial hasta alcanzar la orientación definida en el objetivo. Sin embargo, no tiene por qué coincidir con la dirección en la que se desplaza el robot hacia el siguiente punto. Es decir, el robot puede estar avanzando en una dirección mientras mantiene una orientación diferente. Por tanto, no hay que confundir la orientación del robot con la dirección de desplazamiento.

Durante el desarrollo me encontré con varios problemas relacionados con el planificador. Uno de los primeros fue entender cómo funcionaba y descubrir que podía establecer directamente una orientación objetivo. Esto me permitió conseguir que el robot llegase a su destino con la orientación deseada, en lugar de tener que realizar una planificación adicional para girar una vez alcanzada la posición.

Otro problema fue que los caminos que generaba el planificador no siempre eran los más óptimos. También tuve que ajustar el inflado de los obstáculos para evitar que el robot considerase inaccesibles zonas por las que realmente podía pasar.

En cuanto al seguimiento de la trayectoria, inicialmente utilizaba únicamente las coordenadas X e Y. Al empezar a tener en cuenta la orientación aparecieron otros problemas. Por ejemplo, cuando intentaba mirar puntos más adelantados de la trayectoria para conseguir un movimiento más suave, también utilizaba la orientación de esos puntos. Esto provocaba que, en determinadas situaciones, el robot se desviase o perdiese el camino. Para solucionarlo implementé un VFF (Virtual Force Field), que combina el ángulo hacia el punto objetivo con la orientación que proporciona el propio punto de la trayectoria. Así consigo tener en cuenta tanto hacia dónde debe desplazarse el robot como la orientación que debe ir adoptando.

También aparecieron bucles al seleccionar el punto más cercano de la trayectoria. Al buscar puntos adelantados, en ocasiones el algoritmo volvía a seleccionar un punto anterior al que ya había alcanzado, haciendo que el robot entrase en un bucle. La solución fue obligar a que la selección de puntos se realizase siempre sobre índices superiores al último punto considerado, evitando así retroceder dentro de la trayectoria.

## 3. Segundo caso: el robot con su forma real, sin expandir

En el segundo caso cambio la forma de representar al robot. En lugar de tratarlo como un punto y expandir los obstáculos, utilizo su forma y dimensiones reales para comprobar las colisiones. Para ello utilizo el State Validity Checker de OMPL, que me permite definir cuándo una posición del espacio de estados es válida y cuándo no lo es.

El planificador sigue trabajando en un espacio SE2, por lo que cada estado contiene la posición (x, y) y la orientación (yaw). Para comprobar si un estado es válido, defino una huella rectangular que representa al robot. Sus dimensiones cambian dependiendo de si está transportando una estantería o se encuentra libre.

A partir de la posición y orientación del robot calculo las cuatro esquinas del rectángulo. Después transformo esas coordenadas del mundo a coordenadas de píxeles del mapa y genero una máscara que representa la superficie ocupada por el robot utilizando `cv2.fillConvexPoly`. Finalmente, compruebo si alguno de los píxeles de esa máscara coincide con una zona ocupada del mapa.

Si alguna parte del rectángulo se encuentra sobre un obstáculo o fuera de los límites del mapa, considero que ese estado no es válido. Si toda la superficie está libre, el estado sí puede utilizarse durante la planificación. De esta manera, OMPL tiene en cuenta tanto el tamaño como la orientación del robot a la hora de generar la trayectoria.

Al utilizar este sistema ya no necesito expandir los obstáculos para tener en cuenta las dimensiones del robot, por lo que reduzco el engorde de las paredes a cero. Así evito tener que modificar el mapa original y realizo directamente la comprobación de colisiones con la geometría real del robot.

## 4. Robot Ackermann

Por último, para el robot Ackermann cambia el espacio de estados que utilizo. En este caso trabajo con Dubins State Space, ya que el robot tiene una limitación importante respecto al giro: no puede girar sobre sí mismo y necesita respetar un radio de giro mínimo.

### Marcha atrás

Sin embargo, el robot Ackermann también puede circular marcha atrás, por lo que no quiero limitar al planificador a generar trayectorias que únicamente permitan avanzar. Para ello utilizo un espacio de curvas que admite los dos sentidos de circulación, manteniendo el mismo radio de giro mínimo tanto hacia delante como hacia atrás.

Esto permite resolver situaciones que, si únicamente pudiese avanzar, serían imposibles o requerirían recorridos mucho más largos. Por ejemplo, salir de un hueco estrecho o colocarse con la orientación adecuada para recoger una estantería sin tener que dar una vuelta completa.

El camino que devuelve el planificador contiene la posición y la orientación de cada punto, pero no indica directamente si el robot debe avanzar o retroceder. Por tanto, tengo que deducir el sentido de la marcha a partir del propio movimiento de la trayectoria. Para ello comparo la dirección en la que se desplaza el robot entre dos puntos consecutivos con la orientación que tiene en ese momento.

Si el desplazamiento coincide con la dirección hacia la que está orientado el robot, debe avanzar. Si, por el contrario, el desplazamiento se produce en la dirección opuesta, debe retroceder.

Una vez calculado el sentido de circulación, divido la trayectoria en tramos en los que el robot mantiene siempre el mismo sentido. Cada vez que detecto un cambio entre avanzar y retroceder, considero que termina un tramo y comienza otro.

Para seguir cada uno de estos tramos utilizo una estrategia similar a la anterior: selecciono un punto algo más adelantado dentro de la trayectoria y calculo el arco necesario para llegar hasta él. La diferencia está en cómo interpreto ese punto dependiendo del sentido de circulación.

Cuando el robot tiene que retroceder, realizo el mismo cálculo que utilizaría para avanzar, pero considerando que el robot está mirando hacia el lado contrario. Además, envío una velocidad lineal negativa para indicar que debe desplazarse marcha atrás. La velocidad angular se calcula de la misma manera, ya que el giro necesario sigue siendo el mismo.

Cuando llega al final de un tramo intermedio en el que tiene que cambiar de sentido, el robot se detiene durante un instante y comienza el siguiente tramo en el sentido contrario, de forma similar a lo que haría un coche al cambiar de marcha.

Si se trata del último tramo, compruebo que el robot haya alcanzado tanto la posición como la orientación del objetivo. Si llega a sobrepasar el punto sin cumplir ambas condiciones, vuelvo a planificar desde su posición actual para obtener una nueva trayectoria.
