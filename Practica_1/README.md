# PRACTICA 1 : LOCALIZED VACUUM CLEANER

Para empezar, dividimos el mapa en pequeñas celdas de un tamaño parecido al del robot. Para crear estas celdas, recorremos el mapa de izquierda a derecha y de arriba a abajo, separándolo en cuadrados del tamaño que hemos elegido.

Después, miramos cada celda para comprobar si tiene obstáculos. Usamos un **umbral del 30%**: si una celda tiene un **30% o más de píxeles negros**, la consideramos como un obstáculo. Si tiene menos, la dejamos como una zona libre.

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

Y comparando con la imagen de unibotics donde esta el robot deducimos que esta en el [650,550].

Una vez con el punto implemento el algoritmo para recorrer el path, para ello primero defino las direcciones privilegiadas de modo N, E, S, O, y guardo la dirección actual y lo pruebo para ver:

VIDEO EXPANSION

Ya una vez funcionando el algoritmo de espiral ponemos los puntos de retorno, para vamos añadiendo en una lista todos los puntos de retorno, los puntos de retorno que defino son todos los 4 vecinos de cada punto al que avanzo, de modo que cada punto nuevo va añadiendo 3 puntos de retorno, ya que solo permito coger puntos de retorno en celdillas libres y cada vez que avanzo a una nueva celdilla compruebo si esta es un punto de retorno y si es un punto de retorno la elimino de la lista, es importante aclarar que permito al robot avanzar tanto a puntos de retorno como a celdillas libres. Probandolo sin capacidad de volver a una celda de retorno funciona asi:

VIDEO MOVIMIENTO

Ahora tengo que permitir que vaya a la celda de retorno más cercana sin chocar con nada, para ello usare el algoritmo que usabamos para el laser que es el BFS, que va explorando en todas las direcciónes hasta que encuentra o una celdilla de retorno que entonces se mueve a esa o una celdilla ocupada que entonces desiste de esa dirección y se sigue expandiendo en otras direcciónes, probandolo queda de este modo:

VIDEO BFS Y MOVIMIENTO

Ahora toca definir el movimiento para ello tenemos que sacar en cada iteración la posición y el angulo del robot y la posición de la celda a la que deseamos ir, con eso, consguimos el angulo hacia la celda y la distancia, y mediante un PID sencillo implementamos el movimiento.

