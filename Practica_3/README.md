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

https://github.com/user-attachments/assets/cd53d73b-0377-472b-80c1-146bdef73437

Al principio, me encontré con el problema de que los rayos del LiDAR que devuelven infinito no nos permiten conocer directamente la posición de un posible hueco, ya que no tenemos información sobre la distancia a la que se encuentra un obstáculo. Por tanto, siguiendo esta metodología, no podíamos determinar si había un aparcamiento en una posición concreta.

Tras darle varias vueltas, se me ocurrió comparar dos puntos consecutivos del LiDAR y calcular la distancia que había entre ellos en el eje Y. De esta forma, podía identificar separaciones suficientemente grandes entre puntos y utilizarlas para detectar posibles huecos de aparcamiento.

Sin embargo, este sistema tampoco terminaba de funcionar como necesitábamos, ya que para detectar el aparcamiento teníamos que tener dos coches, uno delante y otro detrás, que delimitaran el hueco. Por ello, me puse a pensar en otro sistema que no dependiera de tener los dos coches y que permitiera detectar el aparcamiento midiendo el espacio libre que hay a un lado del vehículo.

La idea consiste en aprovechar los puntos que detecta el LiDAR en una zona lateral concreta, filtrándolos según su altura, su posición y la distancia a la que se encuentran. De esta forma, podemos identificar cuándo hay un obstáculo junto al coche y guardar su posición como referencia. Mientras seguimos avanzando, comprobamos si el LiDAR continúa detectando suficientes puntos en esa zona. Si los detecta, actualizamos la posición de referencia, ya que seguimos teniendo un obstáculo al lado. En cambio, si dejamos de detectar puntos suficientes, mantenemos la última posición registrada y empezamos a medir la distancia que recorre el coche desde ese punto.

Si al inicio no tenemos ninguna posición de referencia, utilizamos la posición actual del coche como punto de partida para medir esa distancia. Cuando el vehículo ha avanzado al menos cuatro metros sin detectar suficientes puntos en esa zona y tenemos una posición lateral válida guardada previamente, consideramos que hemos encontrado un posible aparcamiento. A partir de ahí, calculamos la posición objetivo en la que queremos aparcar y pasamos a la siguiente fase de la maniobra. De esta forma, no necesitamos detectar dos coches que delimiten el hueco, sino que podemos identificarlo a partir de la distancia que recorremos sin encontrar obstáculos laterales.

Una vez implementado y funcionando este sistema, apareció otro problema: el LiDAR proporciona las posiciones de los puntos respecto al propio sensor, por lo que sus coordenadas cambian a medida que se mueve el coche. Para poder comparar las posiciones detectadas durante el recorrido y utilizarlas en la detección de aparcamientos, necesitamos expresar todos los puntos en un mismo sistema de referencia.

Para solucionarlo, calculamos la posición del coche respecto a su posición inicial y obtenemos su orientación actual. Después, utilizamos las coordenadas de cada punto detectado por el LiDAR y aplicamos una transformación que tiene en cuenta el giro del vehículo mediante las funciones seno y coseno. Además, como los ejes del LiDAR están colocados de forma diferente a los del sistema de referencia que utilizamos, tenemos que intercambiar las coordenadas y cambiar los signos correspondientes. Por último, sumamos la posición actual del coche para obtener las coordenadas de cada punto respecto a la posición inicial. De esta forma, todos los puntos quedan expresados en el mismo sistema de referencia y podemos compararlos para detectar los aparcamientos.

Además, tenemos el problema de que los sistemas de coordenadas del LiDAR y del mapa están girados entre sí, ya que los ejes del LiDAR no coinciden con los que utilizamos en nuestro sistema de referencia. En nuestro caso, el LiDAR tiene el eje X positivo hacia la derecha y el eje Y positivo hacia delante, mientras que en nuestro sistema de referencia utilizamos el eje X hacia delante y el eje Y hacia la izquierda. Por tanto, antes de aplicar la rotación correspondiente a la orientación del coche y sumar su posición, tenemos que intercambiar las coordenadas y cambiar el signo de una de ellas para adaptarlas al sistema de referencia que estamos utilizando. De esta forma, conseguimos que los puntos detectados por el LiDAR estén correctamente orientados antes de calcular su posición absoluta en el mapa.

<img width="1424" height="1044" alt="imagen" src="https://github.com/user-attachments/assets/57da8351-1c2a-45a9-8728-c7d2e162bb70" />

Uno de los principales problemas que me he encontrado es que el coche es no holonómico, es decir, no puede desplazarse lateralmente, sino que tiene que realizar arcos para cambiar su posición en este eje. Esto supone que necesita recorrer varios metros hacia delante o hacia atrás para conseguir un desplazamiento lateral relativamente pequeño. Además, el giro está limitado por la velocidad angular máxima y el radio de la curva depende de la velocidad, por lo que no se puede cerrar el giro todo lo que se quiera. Esto es especialmente problemático en una plaza de unos 4 m, donde el coche ocupa casi todo el espacio disponible y puede llegar a golpear al vehículo de atrás antes de completar la maniobra.

Para solucionar este problema, he acercado el carril de búsqueda hacia la derecha, de forma que el coche circula más cerca de los vehículos aparcados y necesita realizar un desplazamiento lateral menor para entrar en la plaza. Durante esta primera fase, controla su orientación en función del error lateral y comienza a detectar el hueco únicamente cuando ha alcanzado el carril y circula prácticamente recto. Para detectar el aparcamiento, solo se tienen en cuenta los puntos del LiDAR situados en el lado derecho, donde se encuentran los vehículos estacionados.

Además, he comprobado experimentalmente que resulta conveniente aprovechar el avance previo al aparcamiento para orientar el coche hacia la plaza. Por ello, durante la fase de alineación, el vehículo va aumentando progresivamente su ángulo hasta alcanzar aproximadamente 360/5,7° (unos 63,2°) en el punto de inicio de la maniobra. Este valor se ha obtenido probando diferentes ángulos y comparando cuáles permiten acercarse más al vehículo de al lado y completar el aparcamiento con el menor número de maniobras. De esta forma, cuando comienza a retroceder, el coche ya está inclinado y no necesita emplear parte del espacio disponible en realizar el giro inicial.

Para organizar todo el proceso, he implementado una máquina de estados que divide la maniobra en siete fases: BUSCAR, ALINEAR, ATRAS_GIRANDO, ATRAS_ENDEREZANDO, ADELANTE_DERECHA, AJUSTAR y FIN. En el estado BUSCAR, el coche se desplaza hasta el carril establecido y se aproxima a los vehículos aparcados, circulando lo más cerca posible de ellos para reducir el desplazamiento lateral necesario durante la maniobra. Una vez alcanzado este carril, comienza a analizar los puntos del LiDAR situados en el lado derecho para detectar posibles huecos. Cuando identifica un espacio libre suficientemente largo, calcula la posición desde la que debe comenzar a aparcar. A continuación, pasa al estado ALINEAR, en el que avanza hasta alcanzar dicha posición mientras ajusta progresivamente su orientación para facilitar la entrada en la plaza. Una vez alcanzado el punto de inicio, entra en ATRAS_GIRANDO, donde comienza a retroceder con la orientación establecida para introducirse en el aparcamiento. Cuando alcanza la posición en la que debe enderezarse, pasa a ATRAS_ENDEREZANDO, continúa retrocediendo y corrige su orientación hasta quedar paralelo a los vehículos aparcados. Si necesita corregir su trayectoria o recolocarse, pasa al estado ADELANTE_DERECHA, en el que avanza girando hacia la derecha para aproximarse a la posición adecuada. Después, entra en AJUSTAR, donde utiliza un controlador proporcional (P) para corregir su posición longitudinal hasta alcanzar la posición deseada. Finalmente, cuando el error de posición se encuentra dentro de la tolerancia establecida, pasa al estado FIN y termina la maniobra. De esta manera, cada estado tiene un objetivo concreto y las transiciones dependen de la posición y la orientación del coche, lo que permite controlar la maniobra de forma ordenada y reducir los movimientos innecesarios.

<img width="977" height="509" alt="imagen" src="https://github.com/user-attachments/assets/65c92919-9877-4bf9-af1c-2588d72b6275" />

La ejecución final nos quedaría asi:

[[https://youtube.com/shorts/Xp2qRfM9xp4](https://youtube.com/shorts/Ee_OFzZrVBc)](https://youtube.com/shorts/3BxwspNXKOw)
