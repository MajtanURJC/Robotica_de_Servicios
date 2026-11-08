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
Dubins State Space -> Problemas con el giro ya que no coge el radio de giro y al girar choca con cosas

Para el caso de eliminar los margenes de seguridad, tambien he visto el OBB ya que con Dubins mira el radio pero muchas veces no ocupa es eradio y necesito meterla por espacios mucho menores al radio y que pueda girar un poco, porque bloquea todo el giro con Dubins aunque quepa
