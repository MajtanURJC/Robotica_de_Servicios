# Práctica 4: Amazon Warehouse.

En esta práctica hay que programar un robot de almacén que recoge estanterías y las deja en otro sitio. Para ello se usa el planificador **OMPL**, que trabaja con posiciones `(x, y)` y orientaciones (`yaw`). Voy a repasar cómo lo he resuelto en tres escenarios, cada uno más realista que el anterior:

1. El robot como un **punto** con los obstáculos expandidos.
2. El robot con su **forma real** (rectángulo), sin expandir los obstáculos.
3. Un robot **Ackermann**, con restricciones de giro y marcha atrás.

## 1. Calibración

Antes de planificar nada necesitaba saber cómo se mueve realmente el robot. Escribí un programa de calibración que recorre varios puntos, avanza una distancia fija, gira 90° y muestra la posición medida en cada parada. Así comparo lo que ordeno con lo que ocurre.

Este código se puede ver en el fichero de codigo_calibracion y su funcionalidad quedaría asi:

https://github.com/user-attachments/assets/d74d25c3-4522-4733-95f0-229e64b8ab16

Una vez tengo los puntos de referencia, los comparo con el mapa utilizando GIMP para obtener los píxeles correspondientes a cada coordenada. De esta forma puedo relacionar las coordenadas del mundo, expresadas en metros, con las coordenadas de la imagen, expresadas en píxeles, y obtener los datos necesarios para realizar la calibración.

A partir de estas correspondencias, utilizo OpenCV para calcular una matriz de transformación afín mediante la función `cv2.estimateAffine2D`. Esta matriz permite convertir las coordenadas del mundo en píxeles, teniendo en cuenta la escala, el desplazamiento y la orientación del mapa. Además, calculo su matriz inversa para poder realizar también la conversión de píxeles a coordenadas del mundo cuando sea necesario.

![Calibración](https://github.com/user-attachments/assets/7b4985ba-1328-4650-bd19-85561b7e09b5)

Así queda el recorrido de calibración, que utilizo para comprobar que la correspondencia entre ambos sistemas es correcta:

https://github.com/user-attachments/assets/dd037a1e-63e7-46b8-b337-020fba69b047

### Caso 1: Robot como un punto y expansión de obstáculos

En este primer caso, simplificamos el robot considerándolo como un punto. Para tener en cuenta sus dimensiones, en lugar de comprobar toda su geometría, expandimos los obstáculos del mapa. De esta forma, si el punto que representa al robot puede pasar por una zona libre del mapa expandido, entendemos que el robot completo también puede hacerlo.

La expansión depende de si el robot lleva una estantería o no. Cuando está libre, utilizamos un tamaño de 0,55 m y, cuando está cargado, 0,70 m. A estas dimensiones les añadimos un margen de seguridad de 0,20 m. Por tanto, los obstáculos se expanden 0,50 m cuando el robot está libre y 0,90 m cuando lleva una estantería. Esta expansión se realiza hacia todos los lados, por lo que los obstáculos adquieren una forma similar a un círculo más grande.

Para ampliar las zonas ocupadas del mapa, utilizo una operación de OpenCV llamada dilatación, que hace que los obstáculos ocupen más espacio en la imagen. Para ello, utilizo una forma elíptica y calculo cuántos píxeles tengo que ampliar según la distancia que quiero añadir en metros.

También tenemos que actualizar el mapa conforme el robot va recogiendo y depositando estanterías. Para ello, creamos un segundo mapa de ocupación a partir del mapa original, sobre el que aplicamos los cambios acumulados durante el recorrido. De esta forma, no modificamos el mapa original, sino que generamos una nueva versión cada vez que necesitamos planificar una trayectoria. Para saber qué cambios aplicar, guardamos dos listas: una con las zonas donde hemos recogido estanterías y otra con las posiciones donde las hemos depositado.

Cuando recogemos una estantería, liberamos su zona para que deje de aparecer como un obstáculo. Cuando depositamos una, la volvemos a representar en el mapa. En este último caso distinguimos entre las dos situaciones del robot: si está libre, representamos únicamente las esquinas de la estantería, que corresponden a las patas y permiten que el robot pueda pasar por debajo. En cambio, si lleva otra estantería, representamos la estantería depositada como un obstáculo completo.

Hay un detalle importante en la actualización del mapa: después de expandir los obstáculos, volvemos a liberar las zonas de recogida. Esto es necesario porque la expansión de las estanterías cercanas podría llegar a cubrir parte de la zona a la que queremos acceder.

Para la planificación utilizamos OMPL con un espacio de estados SE2, aunque en este caso la comprobación de colisiones solo tiene en cuenta las coordenadas X e Y, ya que el robot está reducido a un punto. También establecemos una orientación objetivo diferente según esté cargado o libre.

El planificador utilizado es RRTConnect. Realizamos cinco intentos de planificación, dedicando un máximo de 2 segundos a cada uno. Cabe destacar que este es el tiempo máximo permitido, pero RRTConnect no necesariamente lo utiliza por completo. De las trayectorias encontradas seleccionamos la de menor longitud mediante una ordenación por longitud. Posteriormente, interpolamos sus puntos para obtener un recorrido más detallado que pueda seguir el robot.

La locomoción del robot es relativamente sencilla, ya que utilizamos un sistema de tracción diferencial. Por tanto, basta con seguir los puntos de la trayectoria generada, sin necesidad de realizar ajustes complejos de ganancias angulares ni controlar una geometría de giro específica. El robot va corrigiendo su orientación mientras avanza y, si detectamos que está demasiado desorientado respecto al siguiente punto, reducimos la velocidad o detenemos el movimiento hasta que vuelva a estar correctamente orientado. Una vez alineado, continúa siguiendo la trayectoria.

Una de las principales ventajas de este método es que resulta relativamente sencillo de implementar y permite tener en cuenta el tamaño del robot mediante la expansión de obstáculos. Sin embargo, tiene una limitación importante: la expansión es una aproximación de la geometría real. Al utilizar una expansión circular, no representa exactamente la forma rectangular de la estantería ni tiene en cuenta su orientación. Esto puede hacer que se bloqueen pasos por los que realmente podría pasar el robot o que la representación no se ajuste completamente a su geometría.

### VIDEO

https://youtu.be/-Xi1o5X3_xc

### Caso 2: Robot con huella rectangular

En este segundo caso mejoramos la representación del robot. En lugar de considerarlo un punto y expandir los obstáculos, utilizamos directamente su huella rectangular para comprobar si puede ocupar una determinada posición y orientación.

Definimos unas dimensiones distintas dependiendo de si el robot está libre o transporta una estantería. Cuando está libre, utilizamos un rectángulo de 0,55 × 0,55 m. Cuando está cargado, las dimensiones pasan a ser 0.88 × 2.08 m. Además, añadimos un margen de seguridad de 0,30 m a cada lado de la huella, aumentando así el espacio que debe quedar libre alrededor del robot.

La principal diferencia respecto al caso anterior es que ahora la comprobación de colisiones depende también de la orientación del robot. Para ello, utilizamos el yaw de cada estado del planificador y calculamos las cuatro esquinas del rectángulo en coordenadas del mundo. Después transformamos esas esquinas a píxeles mediante la calibración del mapa.

Con las cuatro esquinas construimos un polígono que representa la huella del robot. Mediante `cv2.fillConvexPoly` generamos una máscara de los píxeles que ocupa dicho polígono y comprobamos si alguno de ellos coincide con un obstáculo del mapa. También verificamos que la huella completa permanezca dentro de los límites de la imagen. Si alguna parte del rectángulo sale del mapa o se solapa con un obstáculo, esa posición se considera no válida.

Para implementar esta comprobación utilizamos un `State Validity Checker` de OMPL. De esta forma, cada vez que el planificador evalúa un estado, se comprueba si la huella rectangular del robot puede ocupar esa posición con la orientación correspondiente. Como ya tenemos en cuenta las dimensiones reales del robot en esta comprobación, no necesitamos expandir previamente los obstáculos.

También tenemos que considerar la orientación de la estantería que transporta el robot. Para ello, guardamos el ángulo con el que se recogió y calculamos la diferencia respecto a la orientación deseada de la estantería. Esta diferencia se utiliza para orientar correctamente la huella rectangular durante la planificación. Así, aunque el robot cambie de orientación, podemos seguir representando correctamente la posición de la estantería que lleva.

Al igual que en el caso anterior, el mapa se actualiza cada vez que se recoge o deposita una estantería. Las zonas de recogida se liberan y las estanterías depositadas se incorporan como nuevos obstáculos. Además, cuando el robot está libre, las estanterías depositadas se representan mediante sus patas, mientras que cuando está cargado se consideran obstáculos completos.

Para planificar seguimos utilizando RRTConnect, con cinco intentos de hasta 2 segundos cada uno, seleccionando la trayectoria de menor longitud entre las encontradas. La diferencia está en que ahora cada estado debe superar una comprobación de colisión mucho más completa, ya que se tiene en cuenta tanto la posición como la orientación y las dimensiones del robot.

La principal ventaja de este método es que representa de una forma más fiel la geometría del robot y de la estantería que transporta. Esto permite aprovechar mejor los espacios disponibles y comprobar si el robot puede atravesar zonas estrechas teniendo en cuenta su orientación. Sin embargo, también aumenta el coste computacional, porque en cada comprobación hay que calcular las esquinas del rectángulo, generar su máscara y comprobar los píxeles ocupados.

En resumen, el primer caso utiliza una aproximación más sencilla mediante la expansión de obstáculos, mientras que el segundo comprueba directamente la geometría rectangular del robot. El segundo método ofrece una representación más precisa, aunque requiere más cálculos durante la planificación.

Por último, cabe destacar que en estos dos casos he notado un cambio del coste computacional ocurriendo a veces que no era capaz de calcular el angulo y la ruta en RT y por lo tanto perdiendose a veces.

### VIDEO

https://youtu.be/blhjDPClez0

## 4. Robot Ackermann

Para el robot Ackermann cambia bastante el planteamiento, ya que tiene unas limitaciones de movimiento que no tenía el robot anterior. No puede girar sobre sí mismo y necesita respetar un radio de giro mínimo. Al principio utilizaba SE2 State Space, que tiene en cuenta la posición X e Y y la orientación del robot, pero no tiene en cuenta estas restricciones de movimiento. En la práctica, el planificador podía devolver una trayectoria en la que los ángulos indicaban cómo debía llegar el robot a cada posición, pero no garantizaba que la orientación fuese coherente con el recorrido que debía realizar entre esos puntos. Es decir, podía conseguir que el robot llegase con la orientación deseada al punto final, pero sin respetar cómo debía ir orientándose durante el trayecto. Esto generaba trayectorias que después resultaban difíciles de seguir. Este comportamiento se puede ver en la prueba que hice para ver si con SE2 se podía llegar a implementar:

https://github.com/user-attachments/assets/dd3f9b8d-2536-426b-8f5c-01c44512ec61

Para solucionar esto pasé a utilizar Reeds-Shepp State Space, que sí tiene en cuenta el radio mínimo de giro y permite generar trayectorias combinando movimientos hacia delante y hacia atrás. Además, la orientación de cada punto está relacionada con la dirección en la que se desplaza el robot entre ese punto y el siguiente. Es decir, no se trata simplemente de llegar a una posición con una orientación determinada, sino que el planificador tiene en cuenta cómo debe estar orientado el robot para realizar ese desplazamiento. Esto es especialmente importante en un robot Ackermann, ya que no puede cambiar su orientación libremente. Con el planteamiento anterior, utilizando SE2 y RRTConnect, no se tenían en cuenta estas restricciones cinemáticas, por lo que podían aparecer cambios de orientación que no se correspondían con la dirección real del movimiento.

El radio de giro mínimo que utilizo es de 1,4 metros. Esto permite al planificador generar trayectorias que se adaptan mejor al movimiento real del robot y resolver situaciones en las que sería complicado llegar utilizando únicamente la marcha hacia delante.

También tengo que tener en cuenta que el robot cambia de dimensiones dependiendo de si lleva una estantería o no. Por eso utilizo una huella rectangular para comprobar las colisiones, añadiendo además un margen de seguridad de 0,07 metros. De esta manera no planifico únicamente teniendo en cuenta las dimensiones exactas del robot, sino que dejo una pequeña separación respecto a los obstáculos para evitar que pase demasiado cerca de ellos.

El mapa de ocupación se sigue actualizando como en los casos anteriores, liberando las zonas de las estanterías recogidas y marcando las que se han dejado. Además, compruebo las colisiones teniendo en cuenta la orientación del robot y de la estantería que transporta.

Para mejorar la llegada a las estanterías también implementé unos tramos de aproximación. En lugar de hacer que el planificador llegue directamente al punto de recogida o depósito, busco una posición desde la que pueda realizar una aproximación recta hasta el objetivo. Compruebo diferentes distancias y sentidos para encontrar un tramo que esté libre de obstáculos. Si no encuentro ninguno válido, permito que el planificador intente llegar directamente al objetivo. De hecho, en el vídeo he forzado esta situación en la segunda estantería para que también se pueda ver que el robot es robusto ante distintas situaciones y que, aun así, encuentra una ruta y la sigue correctamente sin chocarse.

El planificador genera una trayectoria con posición y orientación en cada punto, pero no indica directamente si el robot tiene que avanzar o retroceder. Para solucionarlo comparo el desplazamiento entre dos puntos consecutivos con la orientación del robot. Si el desplazamiento coincide con la dirección hacia la que está orientado, debe avanzar; si se produce en la dirección contraria, debe retroceder.

Una vez calculado el sentido, divido la trayectoria en diferentes tramos. Cada vez que cambia el sentido de circulación comienza un tramo nuevo. De esta forma puedo seguir cada parte de la trayectoria manteniendo el sentido correspondiente y detener el robot cuando tenga que cambiar de marcha. Esto es muy importante porque, por seguridad, hago que el robot salga por el lateral de la estantería y para ello debe dar marcha atrás. Con esta división consigo que lo haga correctamente y, además, mantengo un número de cambios de sentido bastante reducido.

Una de las mayores dificultades que he tenido ha sido conseguir que el robot siga correctamente la trayectoria, especialmente por el control del ángulo. Al principio el seguimiento no era suficientemente preciso y el robot no corregía bien su orientación respecto a los puntos que tenía que alcanzar.

Para solucionarlo introduje una ganancia angular, de manera que la velocidad de giro depende del error entre la orientación actual del robot y la orientación deseada que devuelve el planificador. Cuanto mayor es el error, mayor es la corrección angular, hasta alcanzar el límite de velocidad angular establecido.

Sin embargo, al aumentar la ganancia angular aparecía otro problema: las sobreoscilaciones. El robot corregía demasiado su orientación, se pasaba del ángulo deseado y después tenía que volver a corregir en sentido contrario. Esto provocaba movimientos poco suaves y hacía que le costase estabilizarse sobre la trayectoria. Tuve que ajustar la ganancia y limitar la velocidad angular para conseguir un seguimiento más estable. Al final conseguí que siguiese las rutas con mucha precisión, prácticamente al píxel, pero a cambio tuve que subir la frecuencia y aumentar la carga computacional. Esto hacía que a veces se me sobrecalentase el ordenador y tuviese que parar las pruebas, ya que perdía constantemente los deadlines de RT y no llegaba a realizar las correcciones a tiempo.

También tuve que modificar la velocidad lineal en función del error angular. Cuando el robot está bastante desalineado, reduzco su velocidad para evitar que siga avanzando mientras intenta corregir su orientación. Además, aplico una reducción de velocidad al acercarse al objetivo, de forma que no llegue demasiado rápido y pueda detenerse con mayor precisión.

Otro aspecto importante ha sido añadir un espacio de seguridad alrededor del robot. No basta con comprobar que sus dimensiones exactas no coincidan con un obstáculo, ya que en la práctica interesa mantener cierta separación. Para ello aumento las dimensiones de la huella rectangular mediante un margen de seguridad. Esto hace que el planificador busque recorridos algo más conservadores y evita que el robot pase demasiado cerca de las estanterías y las paredes.

Por último, uno de los problemas que más dificultades me ha dado ha sido el tiempo de ejecución de la simulación. No he podido trabajar con tiempo de simulación (ST), ya que mientras el programa realizaba los cálculos el robot seguía avanzando y podía salirse de la ruta. Por este motivo me ha resultado más adecuado utilizar tiempo real (RT), aunque también he tenido problemas para cumplir los deadlines. En ocasiones el planificador no disponía realmente del tiempo que yo esperaba, o las comprobaciones entre intentos no se realizaban en las condiciones previstas. Como consecuencia, podía comparar trayectorias que no se habían calculado en condiciones equivalentes o descartar intentos antes de tiempo. Esto hacía que los resultados fuesen bastante variables y que, aunque el código fuese el mismo, unas ejecuciones funcionasen correctamente y otras no.

Para intentar mejorar esto dividí el tiempo total de planificación entre varios intentos. En cada uno utilizo RRTstar para buscar una trayectoria y, si encuentra una, compruebo que sea válida y que llegue correctamente a la posición y orientación de aproximación. Guardo las trayectorias válidas y finalmente selecciono la de menor longitud. De esta manera intento no quedarme con la primera solución que encuentre el planificador, sino comparar varias alternativas.

Aun así, la falta de un tiempo de simulación controlado ha seguido siendo una limitación importante durante las pruebas, ya que los deadlines y los tiempos de ejecución no siempre se comportaban de forma consistente. De todos modos, al final el planificador conseguía seleccionar una trayectoria bastante coherente y el robot podía seguirla correctamente.

Cuando llega al final de un tramo intermedio en el que tiene que cambiar de sentido, el robot se detiene durante un instante y comienza el siguiente tramo en el sentido contrario, de forma similar a lo que haría un coche al cambiar de marcha.

Por último, el robot se encarga de seguir los puntos que le proporciona el planificador, avanzando de uno a otro y ajustando su orientación para intentar mantenerse sobre la trayectoria calculada. En cada iteración comprueba cuál es el siguiente punto al que debe dirigirse y calcula el movimiento necesario para alcanzarlo. Este proceso se repite hasta recorrer todos los puntos de la trayectoria. Una vez que llega al último punto, se considera que el recorrido ha finalizado y el robot puede pasar a la siguiente tarea del programa.
### VIDEO 

https://youtu.be/Sw9CspkspCI


Mientras que SE2 planifica posiciones y orientaciones sin imponer restricciones sobre cómo debe desplazarse el robot entre ellas, Reeds-Shepp genera trayectorias que respetan la geometría de movimiento de un vehículo Ackermann. Esto permite obtener recorridos más adecuados para su ejecución real, especialmente cuando es necesario realizar maniobras marcha atrás.


