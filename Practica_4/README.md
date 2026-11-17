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

Para el robot Ackermann cambia bastante el planteamiento, ya que tiene unas limitaciones de movimiento que no tenía el robot anterior. No puede girar sobre sí mismo y necesita respetar un radio de giro mínimo. Por ello utilizo Reeds-Shepp State Space, que permite generar trayectorias teniendo en cuenta tanto los movimientos hacia delante como hacia atrás.

El radio de giro mínimo que utilizo es de 1,4 metros. Esto permite al planificador generar trayectorias que se adaptan mejor al movimiento real del robot y resolver situaciones en las que sería complicado llegar utilizando únicamente la marcha hacia delante.

También tengo que tener en cuenta que el robot cambia de dimensiones dependiendo de si lleva una estantería o no. Por eso utilizo una huella rectangular para comprobar las colisiones, añadiendo además un margen de seguridad de 0,07 metros. De esta manera no planifico únicamente teniendo en cuenta las dimensiones exactas del robot, sino que dejo una pequeña separación respecto a los obstáculos para evitar que pase demasiado cerca de ellos.

El mapa de ocupación se sigue actualizando como en los casos anteriores, liberando las zonas de las estanterías recogidas y marcando las que se han dejado. Además, compruebo las colisiones teniendo en cuenta la orientación del robot y de la estantería que transporta.

Para mejorar la llegada a las estanterías también implementé unos tramos de aproximación. En lugar de hacer que el planificador llegue directamente al punto de recogida o depósito, busco una posición desde la que pueda realizar una aproximación recta hasta el objetivo. Compruebo diferentes distancias y sentidos para encontrar un tramo que esté libre de obstáculos. Si no encuentro ninguno válido, permito que el planificador intente llegar directamente al objetivo.

El planificador genera una trayectoria con posición y orientación en cada punto, pero no indica directamente si el robot tiene que avanzar o retroceder. Para solucionarlo comparo el desplazamiento entre dos puntos consecutivos con la orientación del robot. Si el desplazamiento coincide con la dirección hacia la que está orientado, debe avanzar; si se produce en la dirección contraria, debe retroceder.

Una vez calculado el sentido, divido la trayectoria en diferentes tramos. Cada vez que cambia el sentido de circulación comienza un tramo nuevo. De esta forma puedo seguir cada parte de la trayectoria manteniendo el sentido correspondiente y detener el robot cuando tenga que cambiar de marcha.

Una de las mayores dificultades que he tenido ha sido conseguir que el robot siga correctamente la trayectoria, especialmente por el control del ángulo. Al principio el seguimiento no era suficientemente preciso y el robot no corregía bien su orientación respecto a los puntos que tenía que alcanzar.

Para solucionarlo introduje una ganancia angular, de manera que la velocidad de giro depende del error entre la orientación actual del robot y la orientación deseada que devuelve el planificador. Cuanto mayor es el error, mayor es la corrección angular, hasta alcanzar el límite de velocidad angular establecido.

Sin embargo, al aumentar la ganancia angular aparecía otro problema: las sobreoscilaciones. El robot corregía demasiado su orientación, se pasaba del ángulo deseado y después tenía que volver a corregir en sentido contrario. Esto provocaba movimientos poco suaves y hacía que le costase estabilizarse sobre la trayectoria. Tuve que ajustar la ganancia y limitar la velocidad angular para conseguir un seguimiento más estable.

También tuve que modificar la velocidad lineal en función del error angular. Cuando el robot está bastante desalineado, reduzco su velocidad para evitar que siga avanzando mientras intenta corregir su orientación. Además, aplico una reducción de velocidad al acercarse al objetivo, de forma que no llegue demasiado rápido y pueda detenerse con mayor precisión.

Otro aspecto importante ha sido añadir un espacio de seguridad alrededor del robot. No basta con comprobar que sus dimensiones exactas no coincidan con un obstáculo, ya que en la práctica interesa mantener cierta separación. Para ello aumento las dimensiones de la huella rectangular mediante un margen de seguridad. Esto hace que el planificador busque recorridos algo más conservadores y evita que el robot pase demasiado cerca de las estanterías y las paredes.

Por último, uno de los problemas que más dificultades me ha dado ha sido el tiempo de ejecución de la simulación. Al no trabajar con un tiempo de simulación que avance de forma controlada, muchas veces los tiempos de planificación y ejecución no se comportaban como esperaba. Esto afectaba especialmente a los deadlines que utilizaba para limitar el tiempo de planificación.

En ocasiones el planificador no disponía realmente del tiempo que yo esperaba, o las comprobaciones entre intentos no se realizaban en las condiciones previstas. Como consecuencia, podía comparar trayectorias que no se habían calculado en condiciones equivalentes o descartar intentos antes de tiempo. Esto hacía que los resultados fuesen bastante variables y que, aunque el código fuese el mismo, unas ejecuciones funcionasen correctamente y otras no.

Para intentar mejorar esto dividí el tiempo total de planificación entre varios intentos. En cada uno utilizo RRTstar para buscar una trayectoria y, si encuentra una, compruebo que sea válida y que llegue correctamente a la posición y orientación de aproximación. Guardo las trayectorias válidas y finalmente selecciono la de menor longitud. De esta manera intento no quedarme con la primera solución que encuentre el planificador, sino comparar varias alternativas.

Aun así, la falta de un tiempo de simulación controlado ha seguido siendo una limitación importante durante las pruebas, ya que los deadlines y los tiempos de ejecución no siempre se comportaban de forma consistente.

En definitiva, la parte más complicada del Ackermann no ha sido únicamente conseguir que el planificador encuentre un camino, sino lograr que el robot sea capaz de seguirlo correctamente. He tenido que ajustar la ganancia angular, controlar las sobreoscilaciones, adaptar la velocidad lineal al error de orientación, añadir un margen de seguridad y gestionar los cambios entre marcha hacia delante y marcha atrás. A todo esto se han sumado los problemas de temporización de la simulación, que han hecho que las pruebas no siempre diesen resultados reproducibles.

Cuando llega al final de un tramo intermedio en el que tiene que cambiar de sentido, el robot se detiene durante un instante y comienza el siguiente tramo en el sentido contrario, de forma similar a lo que haría un coche al cambiar de marcha.

Si se trata del último tramo, compruebo que el robot haya alcanzado tanto la posición como la orientación del objetivo. Si llega a sobrepasar el punto sin cumplir ambas condiciones, vuelvo a planificar desde su posición actual para obtener una nueva trayectoria.
