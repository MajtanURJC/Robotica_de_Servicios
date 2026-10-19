# Autoparking - Práctica 3

## Aclaraciones iniciales
Como no le vi sentido hacerlo con el sensor primero y pasar al LIDAR, por lo que he empezado la práctica directamente con el LIDAR.

## LIDAR
Primero y lo más importante le dedique bastante tiempo a entender el LIDAR, lo que devolvía y como detectar huecos con el y que angulo es cada lado.

Lo primero es ver que devuelve, y con un sencillo codigo se puede ver:

```python
import HAL
import Frequency

lidar = HAL.getLidarData()
print("Objeto lidar:", lidar)
Frequency.tick()
```

Y vemos que devuelve lo siguiente:

<img width="421" height="235" alt="imagen" src="https://github.com/user-attachments/assets/32ed955e-6959-4bad-b67a-567d95259d05" />

Lo siguiente que tengo que hacer es entender de que posición a que posición del laser es el lado derecho, ya que solo me interesa parsear la información
del lado derecho, para ello solo imprimo los valores diferentes a inf mediante el siguiente codigo para asi ver por terminal donde están los limites de detección:

```python
import HAL
import Frequency
import math

# Velocidad de avance
VELOCIDAD = 0


# Avanzar recto
HAL.setV(VELOCIDAD)
HAL.setW(0.0)

# Obtener datos del LiDAR
lidar = HAL.getLidarData()

for punto in lidar.points: 
    x, y, z = punto
    if math.isfinite(x) and math.isfinite(y) and math.isfinite(z):
        print(f"X: {x:.2f}, Y: {y:.2f}, Z: {z:.2f}")
Frequency.tick()
```

Lo que nos devuelve todos estos datos:


https://github.com/user-attachments/assets/aaec1798-aff2-4836-a454-c0e7ecfcf331

### Sistema de coordenadas

Lo primero es identificar cómo está definido el sistema de coordenadas del LiDAR, ya que la documentación del ejercicio no lo especifica. Para ello se imprimen los datos que devuelve el sensor y se hacen pruebas colocando objetos en distintas posiciones respecto al robot, observando cómo cambian los valores de cada coordenada. A partir de estas pruebas se comprueba que la coordenada X positiva apunta hacia delante, la Y positiva hacia la derecha y la Z positiva hacia arriba. Con esta información se pueden interpretar correctamente las mediciones y establecer los filtros necesarios para detectar obstáculos y posibles huecos.

<img width="1024" height="559" alt="imagen" src="https://github.com/user-attachments/assets/852877d9-5b4c-4137-973d-815f2e640df4" />
Para filtrar los datos, hemos probado diferentes rangos de coordenadas hasta encontrar el que mejor se adapta a nuestro objetivo. Finalmente, seleccionamos los puntos con Z entre -1 y -0,9 metros y con Y entre -0,2 y 0 metros, descartando los del propio coche y sustituyendo los valores infinitos por 30. Así obtenemos una franja concreta de la nube de puntos, centrada en la zona que nos interesa para detectar huecos.

Después analizamos la coordenada X: los coches aparcados están a unos 4,5 metros, por lo que si obtenemos valores superiores a 5 metros, interpretamos que puede haber un hueco lateral.

Aquí podemos ver una prueba que ha sido bastante reveladora y la final:

https://github.com/user-attachments/assets/2c137591-0935-43e4-996c-af15cd73726f

Ahora solo hay que detectar en que huecos la X es mayor que 5 metiendo un margen y asi sabremos los huecos:

Uploading Grabación de pantalla desde 2026-10-19 20-35-07.mp4…


