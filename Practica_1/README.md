# PRACTICA 1 : LOCALIZED VACUUM CLEANER

Para empezar, dividimos el mapa en pequeñas celdas de un tamaño parecido al del robot. Para crear estas celdas, recorremos el mapa de izquierda a derecha y de arriba a abajo, separándolo en cuadrados del tamaño que hemos elegido.

Después, miramos cada celda para comprobar si tiene obstáculos. Usamos un umbral del 2%: si una celda tiene un 2% o más de píxeles negros, la consideramos como un obstáculo. Si tiene menos, la dejamos como una zona libre.

Así conseguimos dividir el mapa en zonas por las que el robot puede pasar y zonas que debe evitar. Además, mostramos las celdas solo en las zonas libres para poder ver claramente cómo queda dividido el espacio.

Lo siguiente que hemos hecho ha sido decidir la ubicación de comienzo del path, para ello tenía dos opciones, la primera que se me ocurrio fue utilizar una aplicación de edición de imagenes para sacar los pixeles de comienzo, pero me he dado cuenta que la propia plataforma nos da la posición del robot, así que he decidio que mediante las herramientas que nos da la propia plataforma que son:

<img width="576" height="88" alt="imagen" src="https://github.com/user-attachments/assets/bd5b77b0-811a-495f-aa80-ebc181683600" />

Conocer la posición en la que está el robot y elegirla como posición de inicio, que en este caso nos ha dado [-1,1.5]

<img width="379" height="61" alt="imagen" src="https://github.com/user-attachments/assets/d8dceb85-07d0-48d5-9ab7-f466983bebeb" />

Pero al probarlo no funciona, ya que HAL te lo devuelve en metros por lo que al final hay que descargar la imagen y ponerle los pixeles, de todos modos se ve a simple vista ya que no pueden salir pixeles negativos.

Lo más dificil fue descargar la imagen, que lo hice de este modo:

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

Y comparando con la imagen de Unibotics donde está el robot, deducimos que su posición en la imagen es aproximadamente [635, 530]. Para conseguir hacer el registro, cogemos más puntos y vemos la relación entre ellos. Para obtener estos puntos, simplemente movemos el robot a diferentes posiciones y obtenemos sus coordenadas mediante HAL, y después, en GIMP, miramos el punto correspondiente en el que se encuentra el robot en la imagen.

Durante este proceso nos encontramos con un problema, ya que los sistemas de coordenadas que utilizaban HAL y la imagen eran muy diferentes, tanto en el origen como en el sentido de los ejes. Por ello, no bastaba con hacer una simple conversión directa entre coordenadas, sino que era necesario tener en cuenta el cambio de orientación de los ejes y la traslación. La transformación sigue la sigueinte forma:

<img width="181" height="270" alt="imagen" src="https://github.com/user-attachments/assets/81225866-0d7a-4214-b4c4-9d48afa2ba2b" />


Además, es verdad que el robot en la imagen aparece algo desplazado hacia arriba y hacia la izquierda respecto a la posición que obteníamos inicialmente. Por ello, mediante muchas, MUCHAS pruebas con diferentes puntos y ajustes, vemos que la mejor matriz de transformación es:


``` 
T = np.array([
    [-101,    0,   0.0, 580],
    [   0,  101,   0.0, 425],
    [ 0.0,  0.0,   1.0,   0.0],
    [ 0.0,  0.0,   0.0,   1.0]
])
```

Otra de las cosas más dificiles es ver cuantos pixeles hay por metro, que gracias a las diferentes mediciones de pixeles y comparaciones entre HAL y pixeles con la imágen descargada he visto que la mejor relación que he conseguido hayar es de 101 pixeles por metro.

Una vez con el con todo el registro implementado desarrollo el algoritmo para recorrer el path, para ello primero defino las direcciones privilegiadas de modo N, O, S, E, y guardo la dirección actual y lo pruebo para ver viendo que me hace una pequeña espiral pero se queda encierrado en el punto crítico.

Ya una vez funcionando el algoritmo de espiral ponemos los puntos de retorno, para vamos añadiendo en una lista todos los puntos de retorno, los puntos de retorno que defino son todos los 4 vecinos de cada punto al que avanzo, de modo que cada punto nuevo va añadiendo 3 puntos de retorno, ya que solo permito coger puntos de retorno en celdillas libres y cada vez que avanzo a una nueva celdilla compruebo si esta es un punto de retorno y si es un punto de retorno la elimino de la lista y la pinto como visitada, es importante aclarar que permito al robot avanzar tanto a puntos de retorno como a celdillas libres.

Ahora he pensado como coger el camino para ir a la celda de retorno seleccionada, para eso cojo todos los puntos expandidos para llegar a la celda de retorno más cercana, el modo de expandir es como un laser, llego al obstaculo, y cuando llega al obstaculo y si en ninguna de las direcciones de los hay celdilla de retorno y chocan contra obstaculos, lo que hago es que se expandan hacia los lados hasta que encuentre una celdilla de retorno, una vez encontrado el punto de retorno he ido guardando todos los puntos expandidos, lo que hago es darle la vuelta y la añado al camino para así poder tener el camino hasta el punto de retorno, que, aunque no sea recto, me asegura que llego a limpiar toda la casa:


https://github.com/user-attachments/assets/d81692e6-a112-4f5a-806e-dc7bddc11487


Ahora tengo que permitir que vaya a la celda de retorno más cercana sin chocar con nada, para ello usare el algoritmo que usabamos para el laser que es el BFS, que va explorando en todas las direcciónes hasta que encuentra o una celdilla de retorno que entonces se mueve a esa o una celdilla ocupada que entonces desiste de esa dirección y se sigue expandiendo en otras direcciónes, probandolo queda de este modo:

https://youtu.be/WdyD_zK2MoU



