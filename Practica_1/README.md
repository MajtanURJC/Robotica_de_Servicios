# PRACTICA 1 : LOCALIZED VACUUM CLEANER

Para empezar, dividimos el mapa en pequeñas celdas de un tamaño parecido al del robot. Para crear estas celdas, recorremos el mapa de izquierda a derecha y de arriba a abajo, separándolo en cuadrados del tamaño que hemos elegido.

Después, miramos cada celda para comprobar si tiene obstáculos. Usamos un **umbral del 30%**: si una celda tiene un **30% o más de píxeles negros**, la consideramos como un obstáculo. Si tiene menos, la dejamos como una zona libre.

Así conseguimos dividir el mapa en zonas por las que el robot puede pasar y zonas que debe evitar. Además, mostramos las celdas solo en las zonas libres para poder ver claramente cómo queda dividido el espacio.
