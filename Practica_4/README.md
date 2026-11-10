# Práctica 4 - Amazon WareHouse

## Entender la practica

## Calibración
``` python
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

https://github.com/user-attachments/assets/dd037a1e-63e7-46b8-b337-020fba69b047

<img width="1418" height="1072" alt="imagen" src="https://github.com/user-attachments/assets/7b4985ba-1328-4650-bd19-85561b7e09b5" />

## Primer caso: Robot como un punto expandendo
La estanteria mide 2 metros de largo por 1 de ancho.
Al coger la estanteria tengo que librerar el nuevo mapa para que no crea que hay un obstaculo, ya que estamos moviendo el obstaculo que estamos detectando.


Cada vez que el robot planifica rehago el mapa de ocupación desde cero a partir del mapa original, y le aplico lo que ha cambiado en el almacén: las estanterías que he recogido las vacío y las que he dejado las marco como obstáculo. Como las zonas vaciadas se van acumulando en una lista no se pierden de una planificación a otra, y lo mismo con las patas de las que ya he dejado. Lo que cambia es cómo marco las estanterías dejadas según si el robot lleva una encima o no. Si va libre solo marco las patas porque puede pasar por debajo, y si va cargado marco la estantería entera porque no puede pasar por en medio. No necesito dos mapas guardados, simplemente construyo uno u otro según el estado. En el debug siempre dibujo solo las patas, así se ve limpio, aunque el bloque entero sí cuenta para planificar. Además la estantería la dejo girada 90 grados respecto a como la recojo, así que planifico ya con ese ángulo final y el robot se gira en el sitio antes de bajarla. Lo que no controlo es que ese giro en el sitio choque con algo, solo compruebo que la posición final es válida, y si dejo una estantería muy pegada a otra el margen de seguridad con el robot cargado puede hacer que no encuentre camino.

Para debbugear que zona liberaba y que dona ocupaba he usado showNunpy para dibujar en el mapa:
<img width="1045" height="421" alt="imagen" src="https://github.com/user-attachments/assets/9b487b0c-af2e-4a05-8959-94eeabd8c5b6" />


Más problemas:
Entender el planificador.
Darme cuenta de que le podia poner un angulo objetivo al planner para que llegue con ese objetivo en vez de que una vez llegar girar.
No es el camino más optimo nunca.
Problemas con el inflado de obstaculos
Problema ya que al principio usaba solo el x e y del planner y no el angulo
Problema de que como voy mirando más alante tambien cojo el angulo de mas alante por lo que a veces se me perdia, usar un VFF entre el angulo hasta el punto y el angulo que me da ese punto.
Otro problema es que miraba a puntos mas alante y el mas cercano y a veces cogia el punto anterior y entraba en un bucle por lo que por fuerza hago que coja puntos de indice superior

Ancho 1 largo 2

El planificador OMPL trabaja con posiciones (X, Y) y orientaciones (yaw). Por tanto, cada punto de la trayectoria tiene asociada una orientación.

El yaw representa la orientación que debe tener el robot en cada punto de la trayectoria. Va cambiando progresivamente desde la orientación inicial hasta alcanzar la orientación definida para el objetivo.

Sin embargo, esto no significa que la orientación de cada punto tenga que coincidir con la dirección hacia la que se encuentra el siguiente punto. Es decir, el robot puede avanzar en una dirección mientras mantiene una orientación diferente.

Por tanto, el yaw del planificador se puede entender como una transición de ángulos a lo largo de la trayectoria, cuyo valor final sí está condicionado por la orientación establecida en el objetivo.
## Segundo caso: Robot con la forma real y sin expandir
## ACKERMAN
Dubins State Space -> Problemas con el giro ya que el ackerman no puede girar sobre si mismo

Para el caso de ackerman, tambien he visto el OBB ya que con Dubins necesita OBB

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

**Marcha atrás.** El robot Ackermann puede retroceder, así que el planificador no se limita a caminos que solo van hacia delante. Usa un espacio de curvas que admite los dos sentidos, con el mismo radio de giro mínimo en ambos. Gracias a eso puede resolver situaciones que hacia delante serían imposibles o muy largas, como salir de un hueco estrecho o colocarse en la orientación justa para recoger una estantería sin tener que dar una vuelta enorme.

Un detalle importante es que el camino que devuelve el planificador solo guarda la posición y la orientación del robot en cada punto, no si en ese momento va hacia delante o hacia atrás. Ese sentido se deduce después: en cada paso se compara hacia dónde se desplaza el robot con hacia dónde está mirando. Si el desplazamiento va en la misma dirección que la orientación, avanza; si va en la contraria, retrocede.

Con esa información se parte la trayectoria en tramos de sentido constante. Cada vez que el sentido cambia hay un cambio de marcha, y ese punto es el final de un tramo y el principio del siguiente.

Cada tramo se sigue con la misma idea de siempre: elegir un punto del camino un poco por delante y calcular el arco que lleva hasta él. En un tramo de marcha atrás se hace el mismo cálculo pero imaginando que el robot mira hacia el lado contrario, y la velocidad lineal que se envía es negativa. La velocidad angular sale con el mismo cálculo, porque el giro del robot es el mismo mirado desde cualquiera de los dos lados.

Al llegar al final de un tramo intermedio, el robot se detiene un momento, como haría un coche al cambiar de marcha, y entonces empieza el siguiente tramo en el sentido contrario. Si es el último tramo, se comprueba que ha llegado a la posición y la orientación del objetivo; si se ha pasado sin llegar, se replanifica desde donde esté.
