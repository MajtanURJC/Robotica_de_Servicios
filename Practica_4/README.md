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

### Caso 1: Robot como un punto y expansión de obstáculos

En este primer caso, simplificamos el robot considerándolo como un punto. Para tener en cuenta sus dimensiones, en lugar de comprobar toda su geometría, expandimos los obstáculos del mapa. De esta forma, si el punto que representa al robot puede pasar por una zona libre del mapa expandido, entendemos que el robot completo también puede hacerlo.

La expansión depende de si el robot lleva una estantería o no. Cuando está libre, utilizamos un tamaño de 0,30 m y, cuando está cargado, 0,70 m. A estas dimensiones les añadimos un margen de seguridad de 0,20 m. Por tanto, los obstáculos se expanden 0,50 m cuando el robot está libre y 0,90 m cuando lleva una estantería.

Para realizar esta expansión utilizamos operaciones morfológicas de OpenCV, concretamente una dilatación con un kernel elíptico. La escala de la transformación entre el mapa y el mundo nos permite calcular cuántos píxeles corresponden a la distancia de expansión en metros.

También tenemos que actualizar el mapa conforme el robot va recogiendo y depositando estanterías. Para ello, partimos siempre del mapa original y aplicamos los cambios acumulados. Cuando recogemos una estantería, liberamos su zona para que deje de aparecer como un obstáculo. Cuando depositamos una, la volvemos a representar en el mapa.

En este último caso distinguimos entre las dos situaciones del robot: si está libre, representamos únicamente las esquinas de la estantería, que corresponden a las patas y permiten que el robot pueda pasar por debajo. En cambio, si lleva otra estantería, representamos la estantería depositada como un obstáculo completo.

Hay un detalle importante en la actualización del mapa: después de expandir los obstáculos, volvemos a liberar las zonas de recogida. Esto es necesario porque la expansión de las estanterías cercanas podría llegar a cubrir parte de la zona a la que queremos acceder.

Para la planificación utilizamos OMPL con un espacio de estados SE2, aunque en este caso la comprobación de colisiones solo tiene en cuenta las coordenadas X e Y, ya que el robot está reducido a un punto. También establecemos una orientación objetivo diferente según esté cargado o libre.

El planificador utilizado es RRTConnect. Realizamos cinco intentos de planificación, dedicando un máximo de 2 segundos a cada uno. De las trayectorias encontradas seleccionamos la de menor longitud y posteriormente interpolamos sus puntos para obtener un recorrido más detallado que pueda seguir el robot.

Una de las principales ventajas de este método es que resulta relativamente sencillo de implementar y permite tener en cuenta el tamaño del robot mediante la expansión de obstáculos. Sin embargo, tiene una limitación importante: la expansión es una aproximación de la geometría real. Al utilizar una expansión circular, no representa exactamente la forma rectangular de la estantería ni tiene en cuenta su orientación. Esto puede hacer que se bloqueen pasos por los que realmente podría pasar el robot o, dependiendo de la aproximación utilizada, que la representación no se ajuste completamente a su geometría.

### VIDEO

https://youtu.be/-Xi1o5X3_xc

### Caso 2: Robot con huella rectangular

En este segundo caso mejoramos la representación del robot. En lugar de considerarlo un punto y expandir los obstáculos, utilizamos directamente su huella rectangular para comprobar si puede ocupar una determinada posición y orientación.

Definimos unas dimensiones distintas dependiendo de si el robot está libre o transporta una estantería. Cuando está libre, utilizamos un rectángulo de 0,30 × 0,30 m. Cuando está cargado, las dimensiones pasan a ser 1 × 2 m. Además, añadimos un margen de seguridad de 0,30 m a cada lado de la huella, aumentando así el espacio que debe quedar libre alrededor del robot.

La principal diferencia respecto al caso anterior es que ahora la comprobación de colisiones depende también de la orientación del robot. Para ello, utilizamos el yaw de cada estado del planificador y calculamos las cuatro esquinas del rectángulo en coordenadas del mundo. Después transformamos esas esquinas a píxeles mediante la calibración del mapa.

Con las cuatro esquinas construimos un polígono que representa la huella del robot. Mediante `cv2.fillConvexPoly` generamos una máscara de los píxeles que ocupa dicho polígono y comprobamos si alguno de ellos coincide con un obstáculo del mapa. También verificamos que la huella completa permanezca dentro de los límites de la imagen. Si alguna parte del rectángulo sale del mapa o se solapa con un obstáculo, esa posición se considera no válida.

Para implementar esta comprobación utilizamos un `State Validity Checker` de OMPL. De esta forma, cada vez que el planificador evalúa un estado, se comprueba si la huella rectangular del robot puede ocupar esa posición con la orientación correspondiente. Como ya tenemos en cuenta las dimensiones reales del robot en esta comprobación, no necesitamos expandir previamente los obstáculos.

También tenemos que considerar la orientación de la estantería que transporta el robot. Para ello, guardamos el ángulo con el que se recogió y calculamos la diferencia respecto a la orientación deseada de la estantería. Esta diferencia se utiliza para orientar correctamente la huella rectangular durante la planificación. Así, aunque el robot cambie de orientación, podemos seguir representando correctamente la posición de la estantería que lleva.

Al igual que en el caso anterior, el mapa se actualiza cada vez que se recoge o deposita una estantería. Las zonas de recogida se liberan y las estanterías depositadas se incorporan como nuevos obstáculos. Además, cuando el robot está libre, las estanterías depositadas se representan mediante sus patas, mientras que cuando está cargado se consideran obstáculos completos.

Para planificar seguimos utilizando RRTConnect, con cinco intentos de hasta 2 segundos cada uno, seleccionando la trayectoria de menor longitud entre las encontradas. La diferencia está en que ahora cada estado debe superar una comprobación de colisión mucho más completa, ya que se tiene en cuenta tanto la posición como la orientación y las dimensiones del robot.

La principal ventaja de este método es que representa de una forma más fiel la geometría del robot y de la estantería que transporta. Esto permite aprovechar mejor los espacios disponibles y comprobar si el robot puede atravesar zonas estrechas teniendo en cuenta su orientación. Sin embargo, también aumenta el coste computacional, porque en cada comprobación hay que calcular las esquinas del rectángulo, generar su máscara y comprobar los píxeles ocupados.

En definitiva, el primer caso utiliza una aproximación más sencilla mediante la expansión de obstáculos, mientras que el segundo comprueba directamente la geometría rectangular del robot. El segundo método ofrece una representación más precisa, aunque requiere más cálculos durante la planificación.

### VIDEO

https://youtu.be/blhjDPClez0

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

Si se trata del último tramo, compruebo que el robot haya alcanzado tanto la posición como la orientación del objetivo. Si llega a sobrepasar el punto sin cumplir ambas condiciones, vuelvo a planificar desde su posición actual para obtener una nueva trayectoria

### VIDEO 

https://youtu.be/Sw9CspkspCI

### Diferencias entre SE2 y Reeds-Shepp

Se han utilizado dos espacios de planificación diferentes para generar las trayectorias del robot:

**SE2 (`SE2StateSpace`):**

Este planificador trabaja con tres variables: posición X, posición Y y orientación (yaw). Sin embargo, no tiene en cuenta las restricciones cinemáticas de un vehículo Ackermann ni su radio mínimo de giro. Por tanto, puede generar trayectorias en las que la orientación del robot no sea coherente con la dirección de desplazamiento entre puntos consecutivos. Aunque las posiciones y orientaciones sean válidas geométricamente, la trayectoria puede resultar difícil de ejecutar por el robot.

**Reeds-Shepp (`ReedsSheppStateSpace`):**

Este planificador tiene en cuenta las restricciones cinemáticas de un vehículo Ackermann, incluyendo su radio mínimo de giro. Genera trayectorias formadas por tramos de avance y retroceso, cuyas orientaciones están relacionadas con el movimiento del vehículo. De esta forma, los cambios de orientación siguen curvas compatibles con las capacidades de giro del robot.

**Diferencia principal:**

Mientras que SE2 planifica posiciones y orientaciones sin imponer restricciones sobre cómo debe desplazarse el robot entre ellas, Reeds-Shepp genera trayectorias que respetan la geometría de movimiento de un vehículo Ackermann. Esto permite obtener recorridos más adecuados para su ejecución real, especialmente cuando es necesario realizar maniobras marcha atrás.


