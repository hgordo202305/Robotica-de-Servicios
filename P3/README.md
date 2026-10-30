# Práctica 3 - Autoparking
Esta práctica consiste en desarrollar un coche capaz de aparcar de manera autónoma en una plaza situada entre otros 
vehículos. Para ello, hemos implementado un algoritmo basado en una máquina de estados que combina la información de 
un LiDAR 3D con la odometría del vehículo, permitiendo detectar huecos libres, calcular distancias y controlar las 
diferentes fases de la maniobra de aparcamiento.

## Funcionamiento General
### Procesamiento del LiDAR y detección de obstáculos
El vehículo utiliza un LiDAR 3D para obtener información del entorno y detectar los vehículos que se encuentran a su 
alrededor. Para poder trabajar con las medidas del sensor, transformamos sus coordenadas al sistema de referencia 
del coche:
- Longitudinal: *x=-z_lidar*
- Lateral: *y=-x_lidar*
- Altura: *z=y_lidar*

Además, filtramos las lecturas para descartar los puntos que no se encuentran dentro de los rangos de altura y distancia
establecidos. En nuestro caso, utilizamos una altura comprendida entre 0,2 y 1,2 metros y un rango longitudinal entre
-1 y 3 metros para la detección de huecos.

Una vez procesados los puntos, dividimos la zona lateral del vehículo en sectores de 0,5 metros y contamos cuántos puntos del LiDAR aparecen en cada uno de ellos.

Consideramos que existe un posible hueco cuando encontramos cuatro sectores consecutivos con menos de 76 puntos. De esta manera, podemos identificar los espacios libres entre los vehículos estacionados sin necesidad de construir un mapa completo del entorno.

### Aproximación a la fila de vehículos
Antes de comenzar la búsqueda de una plaza, el coche realiza una maniobra de aproximación dividida en dos fases.

En la primera fase, el vehículo avanza girando hacia la derecha hasta alcanzar un desplazamiento lateral de un metro respecto a su posición inicial. Para calcular este desplazamiento utilizamos la odometría y la orientación inicial del coche:

<img width="302" height="44" alt="image" src="https://github.com/user-attachments/assets/3ae8e827-c6a9-4eaa-8f59-49f36978de1f" />

Donde *theta_0* representa el ángulo de orientación inicial y *Delta x* e *Delta y* son las diferencias de posición respecto al punto de partida.

En la segunda fase, el coche continúa avanzando mientras gira hacia la izquierda para recuperar la orientación original de la calle.

Para controlar este proceso calculamos la diferencia angular mínima entre la orientación actual y la inicial:

  <img width="318" height="43" alt="image" src="https://github.com/user-attachments/assets/2a9b51cb-8ed1-4869-a1b6-97e965e87eaf" />


Cuando la desviación es inferior a la tolerancia establecida de 0,07 radianes, consideramos que el vehículo ha recuperado suficientemente su orientación y comienza la búsqueda del hueco.

### Búsqueda y medición del hueco
Una vez completada la aproximación, el vehículo avanza en línea recta siguiendo la fila de coches hasta detectar un posible espacio libre.

Cuando se detecta un hueco, guardamos la posición inicial mediante la odometría y comenzamos a medir la distancia recorrida mientras el LiDAR continúa detectando el espacio vacío.

Para calcular esta distancia utilizamos la distancia euclídea entre la posición inicial y la actual:

<img width="261" height="35" alt="image" src="https://github.com/user-attachments/assets/b8173fb0-37e0-45fa-9690-45c31297bf3b" />

Si durante la búsqueda el LiDAR deja de detectar el hueco, comprobamos la distancia acumulada. Si esta alcanza los metros establecidos, consideramos que el 
espacio es suficientemente grande y lo aceptamos como una plaza válida.

En caso contrario, descartamos el hueco y continuamos avanzando para buscar otro.

Una vez confirmado el espacio disponible, el coche avanza un metro adicional para rebasar el vehículo delantero y 
colocarse en una posición adecuada antes de comenzar la maniobra de aparcamiento.

### Maniobra de aparcamiento
La maniobra de aparcamiento se divide en dos fases principales, controladas mediante la odometría y la orientación del vehículo.

### Primera fase: introducir la parte trasera
El coche comienza a retroceder girando hacia la derecha para introducir la parte trasera dentro de la plaza.

Durante esta fase calculamos el desplazamiento lateral respecto a la posición inicial de aparcamiento. 
Cuando alcanza un metro, damos por finalizada esta primera parte y comenzamos a enderezar el vehículo.

### Segunda fase: recuperar la orientación
En esta fase el vehículo continúa retrocediendo mientras gira hacia la izquierda para recuperar la orientación inicial de la calle.

Para controlar el giro utilizamos un controlador proporcional basado en el error angular. De esta manera, la velocidad angular depende 
de la diferencia entre la orientación actual y la deseada, reduciéndose progresivamente cuando el coche se acerca a la orientación inicial.

Consideramos que el vehículo ha terminado de aparcar cuando la desviación angular es inferior a 0,07 radianes

### Control de seguridad durante la marcha atrás
Para evitar colisiones durante el aparcamiento, incorporamos un sistema de detección de obstáculos traseros mediante el LiDAR 3D. 
Cuando detecta un vehículo a menos de 0,65 metros, modifica la maniobra: interrumpe la marcha atrás y, si es necesario, 
avanza lentamente para corregir la orientación y finalizar el aparcamiento de forma segura.

### Máquina de estados y control del movimiento
Para organizar todo el proceso hemos utilizado una máquina de estados que permite ejecutar las diferentes etapas de forma secuencial:

1. Aproximación a la fila girando hacia la derecha.
2. Recuperación de la orientación inicial.
3. Búsqueda de un posible hueco.
4. Medición y validación de la longitud del hueco.
5. Avance adicional para rebasar el vehículo delantero.
6. Primera fase de aparcamiento, introduciendo la parte trasera.
7. Segunda fase de aparcamiento, recuperando la orientación.
8. Finalización y detención del vehículo.

Cada estado establece las velocidades lineales y angulares correspondientes y comprueba las condiciones necesarias para pasar al siguiente.

Además, utilizamos la odometría para controlar los desplazamientos y evitar depender exclusivamente de las lecturas instantáneas del LiDAR.

## Problemas encontrados
Uno de los principales problemas encontrados durante el desarrollo ha sido conseguir detectar correctamente los huecos entre los vehículos. 
Ha sido necesario ajustar los filtros del LiDAR, el tamaño de los sectores y el número mínimo de puntos para evitar detectar como huecos zonas ocupadas o perder espacios realmente disponibles.

Otro problema importante ha sido adaptar las maniobras de aparcamiento al sistema de coordenadas del sensor y conseguir que el vehículo recuperase correctamente su orientación después de introducir 
la parte trasera en la plaza.

También hemos tenido dificultades para evitar las colisiones con los coches situados detrás, no lo he solucionado de manera correcta como se puede ver en los videos mas adelante.
Otro problema que también me ha costado el proceso de maniobrar para entrar al hueco del aparcamiento.

## Conclusión
El algoritmo desarrollado permite realizar un aparcamiento autónomo combinando LiDAR 3D, odometría y una máquina de estados. La detección de huecos y el control de las diferentes fases permiten localizar una plaza y ejecutar la maniobra de aparcamiento. 

## Videos de la solucion en distintos casos

[Video del aparcamiento con coches delante y detras](https://youtube.com/shorts/KSed4HCm7eg)

[Video del aparcamiento sin coche detrás](https://youtube.com/shorts/ls2XxjZyEmc)

[Video del aparcamiento sin coche delante](https://youtube.com/shorts/gpMMCTr3jfQ)
