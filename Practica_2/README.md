# Drone Rescue People

## Planificar la práctica
Lo primero que he hecho como el nombre indica es entender la práctica, sonará muy claro pero creía que iba a ser una práctica más a 
bajo nivel por eso pensaba al principio como implementar el movimiento, el despegue o el aterrizaje, pero al leer el guión me di cuenta
que está todo bastante abstraido en funciones como:

<img width="680" height="121" alt="imagen" src="https://github.com/user-attachments/assets/52df92b9-e9ed-4447-9598-f289c9affc2e" />

Para aterrizar y despegar o funciones como:

<img width="670" height="107" alt="imagen" src="https://github.com/user-attachments/assets/6232135b-5326-433b-9e55-951a442e5161" />

Para ir a un lugar determinado, por lo que lo simplifica y abstrae de gran manera.

## Máquina de estados

Después se me ha ocurrido usar una máquina de estados para poder diferenciar bien cada estado y su comportamiento, para ello tengo los
siguiente estados:

<img width="161" height="138" alt="imagen" src="https://github.com/user-attachments/assets/6d5559c0-32d8-4f6f-a23b-378f68aa5df4" />

Que funcionan del siguiente modo:
- TAKEOFF -> Estado que está activo durante el tiempo que está despegando.
- GO_TO_SURVIVORS -> Es el estado que lleva al robot a la posición inicial dada donde estan los supervivientes.
- RETURN_TO_BASE -> Es el estado que se activa cuando la batería esta por debajo del 35 para volver al barco a recargar la batería.
- SEARCH_PEOPLE -> Estado en el cual va recorriendo toda la zona del accidente en busca de superviventes.
- GO_TO_SAVED -> Este estado se activa después de recargar el dron, que va a la posición guardada cuando se ha quedado sin batería y sigue
  realizando el recorrido de busqueda.

## CONVERSIÓN DE POSICION

Primero, se convierten las coordenadas GPS de latitud y longitud a distancias en metros. Para ello, se calcula la diferencia de latitud y longitud entre la lancha y los supervivientes, utilizando el radio de la Tierra y corrigiendo la distancia longitudinal según la latitud. Así se obtienen las distancias hacia el este y hacia el norte.
Después, se toma la posición de la lancha como origen (0, 0) y se adaptan las coordenadas al sistema de referencia del dron, cuyo eje X apunta al norte y el eje Y al oeste.
Por tanto, la lancha se encuentra en (0, 0), mientras que la posición de los supervivientes se expresa como (X, Y) = (distancia norte, −distancia este), en metros.

## GESTIÓN DE BATERIA

La batería se descarga progresivamente en función del tiempo, pasando del 100 % al 0 % en 420 segundos (7 minutos). 
Cuando baja al 30 %, el dron guarda su posición y el estado de la misión, vuelve a la base y aterriza para recargarla al 100 %. 
Después, despega y retoma la misión desde el punto donde se había quedado.

La batería es simulada con un tiempo y se implementa mediante la variable `battery`, que comienza al 100 % y se va reduciendo en función del tiempo transcurrido, 
utilizando `time.time()` y una duración total de 420 segundos. Su nivel se muestra por consola cada 5 segundos. Cuando la batería baja al 30 %, 
el dron guarda su posición actual y el estado de la misión para poder retomarla posteriormente. A continuación, regresa a la base y aterriza para recargar 
la batería al 100 %. Una vez recargada, vuelve a despegar, recupera la posición guardada y continúa la misión desde el punto en el que se había quedado. Cada vez
que recargamos volvemos a coger el momento para calcular de nuevo la batería comparandola con el momento en el que hemos recargado.

## DETECCIÓN DE CARAS

La detección de caras se realiza mediante OpenCV y un clasificador Haar. 

El dron captura imágenes con la cámara ventral, las convierte a escala de grises y las analiza girándolas en intervalos de 45° hasta completar los 360°, este
giro se realiza debido a que el detector Haar solo detecta las cara cuando estan rectas, por lo que es necesarío girarlas varias veces para tratar de ponerla recta. 
Esto permite detectar caras con diferentes orientaciones, mejorando las posibilidades de detección. Para evitar perder información en las esquinas, 
se utiliza un lienzo más grande al girar las imágenes. Finalmente, las caras detectadas se marcan en verde y se eliminan las detecciones duplicadas para 
evitar contar varias veces a la misma persona. 

Las detecciones duplicadas se eliminan comparando el centro de cada cara detectada con los centros de las  caras que ya se han encontrado en la misma imagen. 
Si la distancia entre ambas es menor que el 60 % del tamaño de la cara, se considera que es la misma  persona y se descarta la nueva detección. 
Esto es necesario porque una misma cara puede detectarse varias veces al analizar la imagen con diferentes giros.

`face_cascade.detectMultiScale(rotated, scaleFactor=1.1, minNeighbors=4)`

- rotated: imagen girada que se analiza.
- scaleFactor=1.1: factor de reducción de escala entre niveles de búsqueda, que permite detectar caras de diferentes tamaños.
- minNeighbors=4: número mínimo de detecciones vecinas necesarias para aceptar una detección como cara.
  
Además, para evitar contar varias veces a una misma persona, se establece una distancia mínima de 5 metros entre las posiciones registradas. Si se detecta una cara a menos de esa distancia de otra persona ya registrada, se considera que corresponde al mismo individuo y se descarta.

Este umbral se relaciona con la geometría de la cámara: como el dron vuela a 5 metros de altura, la imagen ventral proyecta una superficie del terreno cuya extensión depende del campo de visión. Por ello, una misma persona puede aparecer en imágenes capturadas desde distintas posiciones del dron. El margen de 5 metros permite agrupar detecciones próximas y reducir los duplicados, aunque puede hacer que dos personas distintas que estén muy cerca se contabilicen como una sola.

## ALGORITMO PARA CUBRIR EL TERRENO: ESPIRAL

Cuando el dron llega a la zona donde se encuentran los supervivientes, comienza una búsqueda sistemática mediante un algoritmo de exploración en espiral. El objetivo es cubrir progresivamente el área de búsqueda, desplazándose por distintas posiciones para aumentar las posibilidades de detectar a todas las personas.

Este algoritmo se calculo antes de la fase de ejecución, creando una lista con todas las esquina. La espiral se crea partiendo de un punto central y realizando desplazamientos en cuatro direcciones: derecha, arriba, izquierda y abajo. En cada movimiento se calcula una nueva posición, que se añade a la lista de puntos de la trayectoria.

El algoritmo aumenta progresivamente la longitud de los desplazamientos, repitiendo dos movimientos con cada longitud antes de incrementarla. De esta forma, la trayectoria se expande alrededor del centro siguiendo una espiral cuadrada hacia el exterior sin explorar zonas exploradas anteriormente. El proceso continúa hasta que se alcanza el límite de distancia establecido respecto al centro.

<img width="887" height="558" alt="imagen" src="https://github.com/user-attachments/assets/e65924cb-759a-47f1-aee3-8373e43c5b67" />


## VIDEO DEL FUNCIONAMIENTO

https://youtu.be/laQWJkZ0fRg
