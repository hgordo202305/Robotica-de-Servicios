# ASPIRADORA DE GAMA ALTA

## Objetivo de la práctica

El objetivo de esta práctica es desarrollar un sistema de planificación y navegación para una aspiradora robótica capaz de recorrer una vivienda utilizando un mapa 2D dividido en celdas y el simulador Gazebo.

La implementación se divide en tres bloques principales:

**Procesamiento del mapa → Planificación de la cobertura → Ejecución del recorrido**

Para la planificación se utiliza el algoritmo BSA (*Backtracking Spiral Algorithm*), complementado con una búsqueda en anchura (BFS) para encontrar caminos entre celdas libres y recuperar puntos de retorno pendientes.

---

### 1. Registro del sistema de referencia (Gazebo - Mapa)

Para relacionar las coordenadas del simulador Gazebo, expresadas en metros, con los píxeles de la imagen del mapa, se calcula una transformación afín mediante mínimos cuadrados.

<img width="616" height="201" alt="image" src="https://github.com/user-attachments/assets/5a745334-2e06-49d5-ad1c-ffc6b0ce9c25" />

- **Puntos de referencia:** se utilizan diez parejas de coordenadas conocidas, almacenadas en `puntos_gazebo` y `puntos_gimp`. Las posiciones del simulador se relacionan con sus correspondientes puntos de la imagen para calibrar la transformación.
  <img width="596" height="223" alt="image" src="https://github.com/user-attachments/assets/44153851-6ad5-4848-b649-29ac1b1f3973" />

- **Cálculo de las transformaciones:** mediante `np.linalg.lstsq()` se obtienen las matrices `directa_T` e `inversa_T`, que permiten convertir coordenadas entre ambos sistemas de referencia.
  <img width="646" height="112" alt="image" src="https://github.com/user-attachments/assets/883c0fde-34ad-4392-9afb-415d8a8048b4" />

- **Escala y tamaño de celda:** se establece un tamaño de celda de 30 píxeles (`cell_px = 30`) y se estima su tamaño equivalente en metros mediante `size_cell_m`.
- **Conversión de coordenadas:** las funciones `cell_to_gazebo()` y `gazebo_to_cell()` permiten pasar de una celda de la rejilla a las coordenadas del simulador y viceversa.

La relación entre ambos sistemas permite construir el mapa discretizado y localizar la posición inicial del robot.

```text
Coordenadas Gazebo (m)
          ↕
Transformación afín
          ↕
Coordenadas del mapa (px)
          ↕
Rejilla de celdas
```

---

### 2. Procesamiento del mapa y construcción del Gridmap

El mapa de la vivienda se carga mediante `WebGUI.getMap()` y se procesa utilizando OpenCV para preparar la información necesaria para la planificación.

El procesamiento sigue esta secuencia:

```text
Mapa original → Erosión morfológica → Discretización → Matriz de ocupación
```

1. **Erosión morfológica:** se aplica `cv2.erode()` con un kernel de tamaño 4 × 4 para modificar los límites de las regiones representadas en el mapa y facilitar la identificación de obstáculos.
2. **Discretización:** la imagen se divide en una rejilla de celdas de 30 × 30 píxeles, definida mediante `num_rows` y `num_columns`.
3. **Cálculo de ocupación:** para cada celda se calcula la proporción de píxeles negros.
4. **Clasificación de celdas:** se utiliza el umbral `taken = 0.07`. Si la proporción de píxeles negros supera este valor, la celda se marca como ocupada (`1`); en caso contrario, se considera libre (`0`).
5. **Ajustes del mapa:** se incorporan determinadas celdas adicionales a la matriz de ocupación mediante `celdas_forzadas_obstaculo`, con el objetivo de representar obstáculos concretos del entorno.

El resultado es `matriz_celdas_ocupadas`, que utiliza el planificador para determinar qué celdas puede atravesar el robot. La función `free_cell()` comprueba que una celda se encuentre dentro de los límites del mapa y no esté ocupada.

---

### 3. Planificación de cobertura mediante BSA (Backtracking Spiral Algorithm)

El algoritmo BSA genera un recorrido sobre la rejilla, avanzando por celdas libres no visitadas y almacenando alternativas para poder regresar a ellas cuando la exploración queda bloqueada.

#### Posición inicial y direcciones

La posición inicial se obtiene a partir del primer punto de referencia del mapa. La orientación inicial del robot, proporcionada por `HAL.getPose3d().yaw`, se transforma al sistema de coordenadas de la imagen para determinar la dirección de exploración.

Se utilizan cuatro direcciones:

- Norte: `(-1, 0)`
- Este: `(0, 1)`
- Sur: `(1, 0)`
- Oeste: `(0, -1)`

La lista `direcciones` permite evaluar las celdas vecinas siguiendo el orden de exploración establecido.

#### Avance y puntos de retorno

En cada iteración se buscan las celdas vecinas libres que todavía no han sido visitadas. Si existen varias alternativas, se selecciona la primera según el orden de exploración y las restantes se almacenan como puntos de retorno.

Las principales estructuras utilizadas son:

- `celdas_visitadas`: registra las celdas marcadas como visitadas durante la planificación.
- `bsa_rute`: almacena la secuencia de celdas que forman la ruta planificada.
- `pts_retorno`: contiene los puntos de retorno pendientes.
- `pendientes_pts_retorno`: permite identificar las alternativas de exploración que todavía deben recuperarse.
- `pts_criticos`: registra las celdas donde la exploración se queda sin vecinos nuevos disponibles.

Cuando no existen movimientos nuevos, el algoritmo busca el punto de retorno pendiente más cercano al que se pueda llegar.

#### Búsqueda de caminos mediante BFS

La función `searching_path()` implementa una búsqueda en anchura (*Breadth-First Search*, BFS) sobre las celdas libres.

El algoritmo explora las celdas vecinas, registra sus predecesoras y reconstruye el camino cuando encuentra el objetivo. Si no existe un camino, devuelve una lista vacía.

Esta búsqueda permite conectar distintas zonas transitables del mapa y recuperar puntos de retorno sin atravesar las celdas marcadas como obstáculos.

#### Visualización del recorrido

La función `show_bsa()` genera una representación gráfica del estado de la planificación mediante `WebGUI.showNumpy()`.

Los colores utilizados son:

- **Negro:** obstáculos.
- **Verde:** celdas visitadas durante la planificación.
- **Rojo:** puntos críticos.
- **Cian:** puntos de retorno pendientes.
- **Azul oscuro:** celdas recorridas físicamente por el robot.
- **Amarillo:** posición actual del robot en la visualización.

La visualización permite observar la evolución de la planificación y comparar la ruta calculada con las posiciones reales del robot durante su ejecución.

---

### 4. Navegación local y control reactivo

Una vez calculada la ruta, la función `move_to_cell()` se encarga de desplazar físicamente el robot hacia el centro de cada celda objetivo.

#### Control de movimiento

El controlador utiliza la posición y orientación actuales del robot para calcular:

- **Distancia al objetivo:** se obtiene a partir de la diferencia entre las coordenadas actuales y las coordenadas objetivo.
- **Error angular:** se calcula mediante la diferencia entre la orientación deseada y la orientación actual, normalizada al intervalo \([-\pi,\pi]\).
- **Velocidad lineal:** se ajusta en función de la distancia y del error angular.
- **Velocidad angular:** se utiliza para orientar el robot hacia el objetivo y corregir su trayectoria.

Las velocidades se aplican mediante `HAL.setV()` y `HAL.setW()`. Cuando el error angular supera un umbral, el robot gira antes de avanzar. Además, se limita la velocidad para mantener un movimiento controlado.

#### Detección de obstáculos mediante láser

La función `leer_laser()` obtiene los datos del sensor mediante `HAL.getLaserData()` y comprueba que exista una cantidad suficiente de medidas válidas.

Los valores no finitos o no positivos se sustituyen por una distancia elevada para evitar que interfieran en los cálculos.

A partir de las medidas se estiman las distancias mínimas a obstáculos en tres sectores:

- **Frontal:** permite detectar obstáculos en la dirección de avance.
- **Derecho:** permite identificar paredes cercanas a la derecha.
- **Izquierdo:** permite identificar paredes cercanas a la izquierda.

Cuando se detecta una pared frontal a corta distancia y el robot está orientado aproximadamente hacia ella, se detiene, comprueba si el objetivo está suficientemente cerca y, si es necesario, intenta recuperarse mediante una maniobra de retroceso y giro hacia el lado con mayor espacio disponible.

Durante el desplazamiento también se reduce la velocidad lineal cuando hay obstáculos próximos y se aplican correcciones angulares para alejarse de las paredes laterales.

#### Registro de la ejecución

Durante el movimiento se registra la celda real del robot mediante `gazebo_to_cell()`. Las posiciones recorridas se almacenan en `celdas_recorridas`.

Además, se muestran trazas de depuración con la celda actual, el objetivo, la distancia, el error angular, las velocidades lineal y angular y las distancias detectadas por el láser. Esta información facilita la identificación de problemas de navegación y de maniobras de recuperación.

---

## 5. Implementación y navegación

Entre los aspectos relevantes de la implementación se encuentran:

- La calibración entre el mapa y el sistema de coordenadas del simulador mediante puntos de referencia.
- La generación de una matriz de ocupación para identificar obstáculos y evitar las celdas no transitables.
- La exploración sistemática de celdas libres mediante el algoritmo BSA.
- La recuperación de rutas hacia puntos pendientes mediante búsqueda en anchura (BFS).
- La corrección de la trayectoria utilizando la posición, la orientación y las distancias proporcionadas por el láser.
- La visualización del mapa, los puntos de planificación y las celdas recorridas físicamente.

Para ello, se han implementado funciones de transformación entre coordenadas, planificación de rutas, control del movimiento y detección de obstáculos. Además, se incorporan maniobras de retroceso y giro para intentar superar situaciones en las que el robot encuentra un bloqueo durante el desplazamiento.

**Resultados y limitaciones:** no se ha conseguido completar la cobertura del mapa en la simulación. El principal motivo es que no se han podido calibrar adecuadamente las maniobras de evasión para resolver todos los bloqueos encontrados durante la navegación. Como consecuencia, algunas celdas quedan sin recorrer físicamente, aunque formen parte de la planificación inicial.

---
## 6. Muestra de funcionamiento

En el siguiente vídeo se muestra el funcionamiento del código implementado. Se puede observar la fase de planificación de la cobertura del mapa mediante el algoritmo BSA y una parte del movimiento real del robot en Gazebo.

**Vídeo demostrativo:** [Localized Vacuum Cleaner – Muestra de funcionamiento](https://youtu.be/rlQDo79swc8)
