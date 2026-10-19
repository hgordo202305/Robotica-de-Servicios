# Práctica2 - Drone Rescue People
Esta practica consiste en desarrollar una mision de rescate maritima con la ayuda de un dron que es capaza de explorar una zona y detectar supervivientes
## Funcionamiento General

### Máquina de estados

El programa utiliza una máquina de estados para controlar las fases de la misión de rescate del dron:

- Takeoff: El dron despega hasta alcanzar la altura de vuelo establecida.
- Explore: Busca personas en la zona mediante un patrón de exploración.
- Return: Regresa a la lancha cuando necesita recargar, guardando su posición.
- Charging: Aterriza en la lancha y recarga la batería.
- Resume: Reanuda la búsqueda desde el punto donde se quedó.

### Cálculo de coordenadas y área de búsqueda

Para determinar la zona donde se encuentran los supervivientes, se convierten las coordenadas GPS de la lancha y de la posición estimada del superviviente a coordenadas UTM. A partir de la diferencia entre ambas posiciones se obtiene el desplazamiento en metros respecto a la base.

La conversión de grados, minutos y segundos a coordenadas decimales se realiza mediante la función `conv_gps()`. Posteriormente, `calculo_area()` calcula el desplazamiento local que se utiliza como centro de la zona de búsqueda.

### Generación de la ruta

Una vez calculado el centro del área, se genera una ruta de búsqueda de 30 × 30 metros, con una separación de 2 metros entre las pasadas. Para ello se utiliza un patrón en zigzag, que permite recorrer la zona de forma ordenada y reducir las áreas que quedan sin explorar.

La ruta se compone de una serie de puntos de navegación. El dron se dirige hacia ellos utilizando `HAL.set_cmd_pos()` y pasa al siguiente cuando se encuentra a menos de 0,8 metros del objetivo.

De esta forma, se consigue una exploración sistemática de la zona en lugar de un movimiento aleatorio.

### Detección de supervivientes mediante Haar

Para detectar posibles supervivientes se utiliza el clasificador Haar de OpenCV, que permite identificar caras en las imágenes capturadas por la cámara ventral del dron.

La detección se realiza cada 10 ciclos durante la exploración, siempre que el dron se encuentre a más de 10 metros de la base y no esté en una pausa de detección. Se convierte la imagen a escala de grises y se analiza mediante distintas rotaciones, desde 0 hasta 345 grados, en intervalos de 15 grados.

Esto permite detectar caras con diferentes orientaciones. Como una misma cara puede aparecer varias veces al analizar las imágenes rotadas, se comprueba la distancia entre los centros de las detecciones y se descartan las que se consideran duplicadas.

### Cálculo de la posición de los supervivientes

Una vez detectada una cara, se calcula su centro en la imagen y se estima su posición relativa al dron. Para transformar los píxeles en metros se utiliza una escala que depende de la altura de vuelo y del ancho de la imagen:

A continuación, se tiene en cuenta la orientación del dron (`yaw`) para transformar ese desplazamiento en coordenadas del entorno. Así se obtiene una posición aproximada X e Y para cada superviviente detectado.

Las posiciones se guardan en una lista y se comparan con las registradas anteriormente. Si un nuevo punto está a menos de 4 metros de uno ya registrado, se considera duplicado y no se añade.

Cuando se registra un nuevo superviviente, se muestra su posición por consola. El objetivo de la misión es localizar seis supervivientes.

### Visualización de las detecciones

Para comprobar el funcionamiento de la detección, se dibuja un rectángulo verde alrededor de cada cara identificada y se añade la etiqueta `CARA DETECTADA`.

La imagen ventral procesada se muestra mediante `WebGUI.showImage()`, mientras que la imagen de la cámara frontal se muestra en el lateral con `WebGUI.showLeftImage()`.

Esto permite observar simultáneamente las detecciones desde arriba y la imagen frontal del dron durante la misión.

## Gestión de batería y retorno a la base

La batería comienza al 100 % y disminuye un punto cada 100 ciclos del programa. Se establece un límite mínimo del 50 %, que activa el regreso a la lancha cuando se alcanza.

Si el dron está explorando o reanudando la búsqueda, guarda sus coordenadas actuales en `last_x` y `last_y`, y activa la variable `resume`. Después, cambia al estado de retorno y se dirige a las coordenadas de la base, situadas en el origen (0,0).

Cuando llega a menos de 0,5 metros de la base, inicia el aterrizaje. Una vez en tierra, la batería se restablece al 100 % y se reinicia el contador de ciclos.

Si todavía quedan supervivientes por encontrar, el dron vuelve a despegar y regresa al punto de interrupción antes de continuar la búsqueda. Si ya se han encontrado los seis, la misión termina y se muestran por consola las posiciones registradas.

## Problemas Encontrados

Uno de los principales problemas que he encontrado durante la practica ha sido la detección de las caras y ajustar bien el Haar para poder detectar bien a todos los supeprvivientes y que no detecte otros elementos como el mar o partes del barco.


## Conclusión

Con este algoritmo, el dron puede realizar una búsqueda sistemática mediante una ruta en zigzag, detectar posibles supervivientes utilizando visión artificial y registrar sus posiciones aproximadas. Además, incorpora un sistema de gestión de batería que permite regresar a la base, recargarse y reanudar la misión desde el punto donde se había interrumpido.

La combinación de la máquina de estados, la navegación por waypoints y la detección mediante Haar permite organizar las diferentes tareas de la misión y automatizar el proceso de búsqueda y rescate.

## Video del Funcionamiento

En el video se puede observar cómo el dron recorre la zona de búsqueda y pasa varias veces por las mismas posiciones donde se encuentran los supervivientes. Sin embargo, no los registra de nuevo cada vez que los detecta, ya que se ha implementado un sistema para evitar duplicados.

Este sistema compara las coordenadas de cada nueva detección con las de los supervivientes registrados anteriormente. Si la distancia entre ambas posiciones es inferior a 4 metros (`DIST_DUPLICADOS = 4`), la detección se considera duplicada y no se añade a la lista. De esta forma, se evita contabilizar varias veces al mismo superviviente y se mantiene un registro más fiable de los encontrados.
Ademas, aunque no se vea por el aterriaje forzoso del final, el dorn nos muestra todas las coordenadas de los supervivientes juntas al final de la ejecucion.

https://youtu.be/BVQ5LUxotX4
