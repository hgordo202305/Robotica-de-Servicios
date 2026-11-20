# Práctica - Amazon Warehouse
Esta práctica consiste en programar un robot logístico para que mueva de forma autónoma las 6 estanterías del almacén
Warehouse 1 hasta unas posiciones de destino. Para ello, hemos utilizado la biblioteca de planificación OMPL junto con
un pilotaje reactivo que utiliza únicamente la posición del robot (odometría), organizado todo mediante una máquina de estados.

La práctica se ha resuelto en tres apartados:
- **Apartado a)** Robot holonómico tratado como un punto, con los obstáculos engordados.
- **Apartado b)** Robot holonómico con su geometría real, sin engordar los obstáculos, y con una geometría distinta en
la ida (robot solo) y en la vuelta (robot con la estantería encima).
- **Apartado c)** Robot rectangular con dinámica Ackermann, con las restricciones de movimiento de un coche.

## Funcionamiento General
### Mapa y conversión a píxeles
Obtenemos el mapa del almacén, lo pasamos a escala de grises y aplicamos un umbral para quedarnos con los obstáculos
(los píxeles oscuros). La resolución del mapa la calculamos dividiendo el ancho real del almacén (20,62 m) entre el
número de columnas de la imagen.

Como OMPL trabaja en metros y el mapa en píxeles, medimos seis puntos de calibración entre Gazebo y el mapa y
estimamos con OpenCV una transformación afín que nos permite pasar de una a otra:

$$u = a_{11}\,x + a_{12}\,y + b_1 \qquad v = a_{21}\,x + a_{22}\,y + b_2$$

Donde *(x, y)* es la posición en Gazebo y *(u, v)* el píxel del mapa.

### Estanterías y destinos
Las estanterías están todas en x = 3,728 m. Para los destinos hemos elegido dos filas: cuatro estanterías en
x = -3,70 m separadas 1,35 m entre ellas, y las otras dos en x = -1,15 m.

Con cada estantería repetimos siempre el mismo ciclo: ir a un punto de entrada situado delante de ella, entrar en línea
recta, levantarla, salir marcha atrás, planificar hasta su destino, entrar en línea recta, bajarla y salir marcha atrás.

### Actualización del mapa
Cada vez que planificamos reconstruimos el mapa a partir del original:
- Al levantar una estantería la borramos de su sitio, dejando ese espacio libre. De esta manera, en la vuelta tampoco
cuenta como obstáculo, ya que pasa a ser parte del robot.
- Las estanterías que ya están en su destino las marcamos como obstáculos en su nueva posición.
- Dejamos libre la zona donde está el robot para que el planificador pueda arrancar aunque esté cerca de un obstáculo.

### Planificación y control
Usamos el planificador RRT* con un objetivo de longitud de camino, para que la ruta sea lo más corta posible. Como es un
algoritmo aleatorio, si no encuentra solución volvemos a planificar en la siguiente iteración.

Para seguir la ruta, el robot se dirige de punto en punto. Calculamos el error angular entre la orientación deseada y
la actual, normalizado entre -π y π:

$$e_\theta = \operatorname{atan2}\big(\sin(\theta_{obj}-\theta),\ \cos(\theta_{obj}-\theta)\big)$$

Si el error es grande, el robot gira sobre sí mismo, y si es pequeño avanza corrigiendo el rumbo con una velocidad
angular proporcional al error y una velocidad lineal que depende de la distancia al punto.

### Máquina de estados
1. Planificar y seguir la ruta de ida hasta el punto de entrada.
2. Entrar bajo la estantería y levantar la plataforma.
3. Salir marcha atrás.
4. Planificar y seguir la ruta de vuelta hasta el punto de entrada del destino.
5. Entrar en el destino y bajar la plataforma.
6. Salir marcha atrás y pasar a la siguiente estantería, o terminar si ya están las 6.

## Apartado a) Robot puntual con obstáculos engordados
Tratamos al robot como un punto y engordamos los obstáculos con una dilatación circular. El espacio de estados es
euclídeo en 2D, ya que al ser holonómico no hace falta planificar su orientación.

El radio de engordamiento cambia según el trayecto:
- **Ida:** 0,22 m más 0,06 m de margen.
- **Vuelta:** 0,42 m más 0,06 m de margen, porque lleva la estantería. Es una aproximación, ya que la estantería es un
rectángulo de 2,06 m x 0,88 m y la estamos cubriendo con un círculo.

El punto de entrada está a 1,15 m de la estantería, y al salir del destino retrocedemos hasta 1,55 m.

## Apartado b) Geometría real sin engordar los obstáculos
Aquí dejamos de engordar los obstáculos y representamos al robot con una malla de puntos. Un estado es válido cuando
todos los puntos de la malla, girados y trasladados según la posición del robot, caen en píxeles libres. Usamos dos mallas:
- **Ida:** el robot solo, un cuadrado de 0,36 m de lado.
- **Vuelta:** el robot con la estantería, un rectángulo de 2,06 m x 0,88 m.

Como ahora importa la orientación, el espacio de estados incluye posición y giro, y comprobamos los estados con una
resolución de 5 mm. La orientación objetivo es de cara a la estantería en la ida y en sentido contrario en la vuelta.

Las estanterías depositadas las marcamos por sus cuatro esquinas, y las que ya hemos levantado las borramos con un
rectángulo un poco mayor que ellas. El punto de entrada está a 1,45 m y la salida del destino a 1,65 m.

## Apartado c) Robot rectangular con dinámica Ackermann
Un robot Ackermann no puede moverse lateralmente ni girar sobre sí mismo, así que usamos un espacio de estados de tipo
Reeds-Shepp, con un radio de giro mínimo de 1,3 m y que permite tramos hacia delante y hacia atrás.

El estado de OMPL representa el eje trasero, por lo que lo desplazamos 0,244 m hacia delante para obtener el centro del
robot y comprobar ahí la huella. La huella es una malla de puntos con un margen de seguridad de 15 cm, del chasis (1,15 m
x 0,53 m) en la ida y de la estantería (2,06 m x 0,88 m) en la vuelta. Además, una vez interpolada la ruta (un punto cada
5 cm), volvemos a comprobar todos sus puntos y, si alguno colisiona, la descartamos y replanificamos.

### Aproximación recta
El objetivo del planificador no es el centro de la estantería, sino un punto de aproximación situado delante de ella,
a 2,80 m en la ida y a 1,60 m en la vuelta. Desde ahí añadimos un último tramo recto hasta el centro, para llegar bien alineados.

### Seguimiento de la ruta
Como la ruta mezcla marcha adelante y marcha atrás, la dividimos en tramos según el sentido. Cada tramo se sigue con un
controlador *pure pursuit*: buscamos un punto de la ruta a 0,60 m y calculamos la curvatura del arco que nos lleva hasta él,

$$\kappa = \frac{2\,y_{lateral}}{d^2}$$

y la velocidad angular es la velocidad lineal multiplicada por esa curvatura. La velocidad baja al acercarse al final del tramo.

Para evitar tirones, la velocidad máxima crece poco a poco (hasta 0,20 m/s) durante los primeros 2,5 segundos de cada
tramo. Para esto usamos el tiempo de simulación, porque la velocidad de Gazebo puede variar. Entre un tramo y otro el
robot se detiene del todo, y tras subir o bajar la plataforma esperamos 2 segundos.

Al salir marcha atrás, retrocedemos 2,10 m desde la estantería y hasta 2,25 m del destino, a -0,20 m/s.

## Problemas encontrados
Al principio el robot chocaba con las estanterías al acercarse y no las dejaba bien colocadas. Lo hemos mejorado
añadiendo márgenes de seguridad y entrando siempre en línea recta a la estantería.

Otro problema ha sido mantener el mapa actualizado: había que borrar la estantería al levantarla y marcar las ya
entregadas, porque si no el planificador no encontraba camino o chocaba con ellas.

En el apartado a), aproximar la estantería con un círculo no es una solución perfecta, ya que se pierde espacio libre o
las esquinas pueden rozar. Por eso en los siguientes apartados pasamos a usar mallas de puntos.

En el apartado c) lo que más me ha costado ha sido adaptar el seguimiento al robot Ackermann. Con la marcha atrás hubo
que dividir la ruta en tramos, frenar entre ellos y suavizar la velocidad inicial para evitar tirones.

## Conclusión
El algoritmo desarrollado permite mover las 6 estanterías del almacén combinando planificación con OMPL, actualización
del mapa y una máquina de estados con control basado solo en la posición. Pasar del robot puntual a la geometría real y
después al robot Ackermann nos ha permitido ver cómo se van añadiendo restricciones hasta acercarnos a un robot logístico real.

## Videos de la solucion en cada apartado

[Video del apartado a)](ENLACE)

[Video del apartado b)](ENLACE)

[Video del apartado c)](ENLACE)
