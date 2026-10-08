# PRÁCTICA 1: LOCALIZED VACUUM CLEANER

En la practica he desarrollado tres fases, el registro, la planificación y el movimiento. La primera fase es el registro.

## REGISTRO

Para empezar, dividimos el mapa en pequeñas celdas de un tamaño parecido al del robot pero un poco menor, yo he elegido 30. Para crear estas celdas, recorremos el mapa de izquierda a derecha y de arriba a abajo, separándolo en cuadrados del tamaño que hemos elegido.

Después, miramos cada celda para comprobar si tiene obstáculos. Usamos un umbral del 3%: si una celda tiene un 3% o más de píxeles negros, la consideramos como un obstáculo. Si tiene menos, la dejamos como una zona libre que va a ser recorrida por el robot.

Así conseguimos dividir el mapa en zonas por las que el robot puede pasar y zonas que debe evitar. Además, mostramos las celdas solo en las zonas libres para poder ver claramente cómo queda dividido el espacio.

<img width="393" height="270" alt="Captura desde 2026-10-06 09-50-22" src="https://github.com/user-attachments/assets/a709eaad-c9cd-4ae3-a992-26a12af36277" />

Lo siguiente que hemos hecho ha sido decidir la ubicación de comienzo del path, para ello tenía dos opciones, la primera que se me ocurrio fue utilizar la posición que devuelve el robot con las herraminetas que nos da la plataforma que son:

<img width="576" height="88" alt="imagen" src="https://github.com/user-attachments/assets/bd5b77b0-811a-495f-aa80-ebc181683600" />

Y esas funciones nos devuelven lo siguiente en la posición inicial:

<img width="379" height="61" alt="imagen" src="https://github.com/user-attachments/assets/d8dceb85-07d0-48d5-9ab7-f466983bebeb" />

Pero nos damos cuenta que el HAL nos devuelve la posición en metros y la necesitamos en pixeles, por lo que tuvimos que pasar a la siguiente opcion, que es descargar la imagen que creía que era mucho más sencillo pero todo lo contrario, tuve que buscar la imagen dentro del docker y descargarla del siguiente modo:

```bash
docker ps
docker exec -it vibrant_satoshi bash
find / -type d -name "resources" 2>/dev/null
find /resources/exercises/vacuum_cleaner_loc -type f
find /resources/exercises/vacuum_cleaner_loc/images -type f
ls -lh /resources/exercises/vacuum_cleaner_loc/images/
exit
docker cp vibrant_satoshi:/resources/exercises/vacuum_cleaner_loc/images/mapgrannyannie.png .
xdg-open mapgrannyannie.png
```

Y de este modo consguimos descargar la imagen.

Ahora abrimos la imagen en gimp para sacar el pixel en el que empieza el robot:

<img width="814" height="480" alt="imagen" src="https://github.com/user-attachments/assets/c937489c-ee73-471b-a8ce-46a2c882752a" />

(ACLARACIÓN: Después me di cuenta que se podía hacer simplemente "GUARDAR COMO" sobre la imagen y sacarla pero no se me ocurrio)

Y comparando con la imagen de Unibotics donde está el robot, deducimos que su posición en la imagen es aproximadamente [635, 530]. Para conseguir hacer el registro, cogemos más puntos y vemos la relación entre ellos. Para obtener estos puntos, simplemente movemos el robot a diferentes posiciones y obtenemos sus coordenadas mediante HAL, y después, en GIMP, miramos el punto correspondiente en el que se encuentra el robot en la imagen, algunos puntos que he cogido son:

```text
(-1.000,  1.500) -> [635, 535]
( 2.422,  2.647) -> [335, 696]
( 3.449,  5.174) -> [229, 947]
(-0.956,  5.374) -> [676, 960]
( 3.871,  2.728) -> [189, 699]
( 3.220, -0.334) -> [256, 379]
(-3.643,  0.286) -> [939, 453]
(-2.250,  2.929) -> [809, 729]
( 0.190, -0.249) -> [561, 408]
( 4.581, -2.298) -> [119, 187]
```

Durante este proceso nos encontramos con un problema, ya que los sistemas de coordenadas que utilizaban HAL y la imagen eran muy diferentes, tanto en el origen como en el sentido de los ejes. Por ello, no bastaba con hacer una simple conversión directa entre coordenadas, sino que era necesario tener en cuenta el cambio de orientación de los ejes y la traslación. La transformación sigue la sigueinte forma:

<img width="181" height="270" alt="imagen" src="https://github.com/user-attachments/assets/81225866-0d7a-4214-b4c4-9d48afa2ba2b" />

Además, es verdad que el robot en la imagen aparece algo desplazado hacia arriba y hacia la izquierda respecto a la posición que obteníamos inicialmente. Por ello, mediante muchas, MUCHAS pruebas con diferentes puntos y ajustes, ya que he tenido que cambiar los pixeles de cada metro ya que al estar un poco arriba a la izquierda he tenido que forzar algunos puntos para que no salgan valores muy diferentes y que concuerde con lo hayado anteriormente, vemos que la mejor matriz de transformación es:

```text
T = np.array([
    [-101,    0,   0.0, 580],
    [   0,  101,   0.0, 425],
    [ 0.0,  0.0,   1.0,   0.0],
    [ 0.0,  0.0,   0.0,   1.0]
])
```

Este offset ha salido de las ecuaciones con los puntos hallados:

```text
tx = xp - a * xm
ty = yp - b * ym
```

Y tras muchos valores diferentes vemos una media más o menos en 580 y 425 que sale de estos valores:

```text
tx = [582.000, 579.622, 577.049, 579.444, 579.971, 581.220, 570.857, 581.750, 580.190, 581.681]
ty = [385, 432.219, 429.992, 423.592, 427.038, 413.300, 424.680, 436.737, 433.715, 417.664]
```

Otra de las cosas más dificiles es ver cuantos pixeles hay por metro, que gracias a las diferentes mediciones de pixeles y comparaciones entre HAL y pixeles con la imágen descargada he visto que la mejor relación que he conseguido hallar es de 101 pixeles por metro, es por ello que la matriz aparece 101 y -101.

## PLANIFICACIÓN

Una vez terminado el registrlo paso a desarrollar el algoritmo para recorrer el path, para ello primero defino las direcciones privilegiadas de modo N, O, S, E, y guardo la dirección actual y lo pruebo para ver viendo que me hace una pequeña espiral pero se queda encierrado en el punto crítico.

Ya una vez funcionando el algoritmo de espiral ponemos los puntos de retorno, para vamos añadiendo en una lista todos los puntos de retorno, los puntos de retorno que defino son todos los 4 vecinos de cada punto al que avanzo, de modo que cada punto nuevo va añadiendo hasta 3 puntos de retorno, ya que solo permito coger puntos de retorno en celdillas libres.

Cada vez que avanzo a una nueva celdilla compruebo si esta es un punto de retorno y si es un punto de retorno la elimino de la lista que contiene los puntos de retorno y la pinto como visitada, es importante aclarar que permito al robot avanzar tanto a puntos de retorno como a celdillas libres.

Ahora he pensado como coger el camino para ir a la celda de retorno seleccionada, para ello, como el camino en linea recta no me solucionaba nada y dejaba mucha zona sin limpiar ya que era incapaz de llegar a ellas ya que había obstaculos por medio, he implementado un algoritmo BFS que desarrollamos en IA para encontrar el camino a un destino, para ello primero expando todos las celdillas que no están ocupadas o que no he expandido ya en todas las direcciones y si está libre la guardo y creando un diccionario guardo la celdilla a la que he avanzado y de donde vengo, así expando todos los pixeles hasta que llego al punto de retorno destino, una vez en el retorno lo que hago es recrear el camino desde el destino hasta el origen, es decir, de manera inversa, gracias al diccionario creado anteriormente, busco la celda en la que estoy en el diccionario y añado a la lista la posicion anterior, que es la que está relacionada en el diccionario con la posicion en la que estamos, y así vamos recorriendo el camino de manera inversa, vemos el funcionamiento asi:

https://github.com/user-attachments/assets/d81692e6-a112-4f5a-806e-dc7bddc11487

Como podemos ver en el video expande todas las celdas y por eso vemos que va pegado por la pared.

## MOVIMIENTO

Entonces ya teniendo solucionado tanto el registro como la planificación nos toca el movimiento, para utilizamos el angulo del robot y tenemos que hallar el ángulo a la celdilla que queremos ir y mediante comparación de angulos y con un poco de rango de fallo, hacemos que coincidan, una vez que coinciden los angulos le damos velocidad v comparando en todo momento la posición y el angulo, una vez que llega a la posición sacamos la siguiente celdilla de la lista del camino y volvemos a comparar angulo y distancia hasta llegar a la siguiente celdilla, así con todas las celdillas hasta llegar al final.

Después para poder girar el robot de manera correcta y sin pasarme y estar recalculando todo el rato he añadido un controlador P que cuanto más cerca está del ángulo objetivo más lento gira hasta llegar dentro del margen de 0.1 para empezar a avanzar en linea recta, meto un margen tan pequeño para evitar que choque porque tenga un angulo demasiado diferente al objetivo.

A su vez, para que no haga micromovimientos para llegar al centro de la celda le meto un cierto margen a la posición para que asi cuando este cerca de la posición podamos marcarla como visitada y pasar a la siguiente.

Por último he añadido un algoritmo VFF sencillo porque el mapa está un poco desfasado en algunos lugares y para evitar que se choque he añadido una fuerza repulsiva y atractiva que modifican el angulo, siendo la atractiva hacia la siguiente celdilla en el camino y repulsiva si hay una pared a una distancia menor que D_REP que yo he ajustado a 0.3 por lo que solamente cuando tengamos una pared muy cerca entrara el VFF ya que solo me interesa para situaciones críticas donde el choque vaya a ser inminente.

Una vez terminadas las fases de registro, planificación y movimiento tenemos el ejercicio completo. Demuestro el video del funcionamiento del robot haciendo la ruta completa aquí:

https://youtu.be/WdyD_zK2MoU


