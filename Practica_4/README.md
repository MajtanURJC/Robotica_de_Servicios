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
<img width="1418" height="1072" alt="imagen" src="https://github.com/user-attachments/assets/7b4985ba-1328-4650-bd19-85561b7e09b5" />
