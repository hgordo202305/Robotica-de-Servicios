# ASPIRADORA DE GAMA ALTA

## Objetivo de la práctica
El objetivo de esta práctica es desarrollar un sistema de planificación y navegación para una aspiradora robótica que recorra una vivienda de manera eficiente sobre un mapa dividido por celdas, y usando el simulador Gazebo ademas de un mapa 2D de la vivienda a limpiar.

El desarrollo de la practica lo he llevado a cabo en tres bloques principales:

**Registro del mapa** --> **Planificación de la cobertura** --> **Ejecución del recorrido**

La implementación actual ha usado el algoritmo BSA (Backtracking Spiral Algorithm) para desarrollar el registro y procesamiento del mapa además de la planificación.

---

### 1. Registro del sistema de referencia (Gazebo - Mapa)
Para poder traducir las coordenadas continuas de Gazbo a los pixeles de la iamgen se ha usado una transformación afín estimada por mínimos cuadrados.

* **Puntos de referencia:** se usan parejas de coordenadas que conocemos y que hemos registrado poco a poco tanto en el gazebo moviendo en este caso la aspiradora poco a poco a 10 posiciones distintas, como en el mapa usando gimp para relacionar las 10 posiciones de gazebo con 10 coordenadas de la imagen (`puntos_gazebo`, `puntos_gimp`) para calcular las matrices de transformación directa e inversa.

* **Cálculo de la transformación:** Se resuelve el sistema mediante `np.linalg.lstsq()`.

* **Escala y conversión:** Se calcula el tamaño real de la celda en metros (`size_cell_m`) a partir de la escala obtenida y la definición en píxeles (`cell_px = 30`).

// añadir foto de la matriz

```

Gazebo (m)  <--->  Matriz de transformación  <--->  Mapa (px)

```

Esta relación sienta la base para construir el gridmap y posicionar al robot en su celda inicial.

---

### 2. Procesamiento del mapa y Gridmap

Cargamos el mapa de la casa mediante WebGUI y lo procesamos con OpenCV, al mapa le vamos a aplicar ina **erosión** para poder ajustar el límite de las regiones y asi asegurar que los obstaculo quedan adecuadamente delimitados antes de discretizar.

$$\text{Mapa original} \longrightarrow \text{Erosión morfológica} \longrightarrow \text{Mapa preparado}$$

El mapa procesado permite diferenciar zona stransitables de obstaculos para estructurar la planificación:

1. **Erosión morfológica:** Se aplica `cv2.erode()` sobre la imagen original para dilatar los obstáculos y prevenir colisiones cerca de las paredes.
2. **Discretización en celdas:** La imagen se divide en una rejilla (`num_rows` x `num_columns`).
3. **Matriz de ocupación:** A partir de `size_cell_m` y `cell_px` se subdivide el espacio en una cuadrícula:

  * Para cada celda se analiza la proporción de píxeles ocupados respecto a un **umbral**.
  * **Celda ocupada**: La proporción supera el umbral (zona a evitar).
  * **Celda libre**: Candidata a ser transitada.

  Toda esta información se almacena en una **matriz de ocupación**, generando el *gridmap* sobre el cual trabajará el algoritmo de cobertura.Si el porcentaje de píxeles negros en una celda supera el umbral `taken = 0.07`, se marca como obstáculo (`1`), de lo contrario se clasifica como libre (`0`).

---

### 3. Planificación de cobertura mediante BSA (Backtracking Spiral Algorithm)
El algoritmo **BSA** planifica el recorrido sobre la rejilla combinando avance por celdas contiguas y retroceso a celdas guardadas cuando la ruta queda bloqueada:

* **Posición e inicio:** El robot arranca en la celda correspondiente a su posición inicial y se le asigna la dirección inicial según su orientación (`yaw_actual`).
* **Direcciones de exploración (NESO):** Se evalúan las celdas contiguas en orden de prioridad **Norte → Este → Sur → Oeste** (`direcciones = [(-1,0), (0,1), (1,0), (0,-1)]`).
* **Puntos de retorno y críticos:** Si hay más de una celda vecina disponible, la primera se explora y las demás se guardan como *puntos de retorno* (`pendientes_pts_retorno`). Si el robot no puede continuar, la celda actual se declara *punto crítico*.
* **Reconexión mediante BFS:** Al encallarse en un punto crítico, el algoritmo usa **Búsqueda en Anchura (BFS)** (`searching_path()`) para encontrar el camino libre más corto hacia el punto de retorno pendiente más cercano.
* **Visualización:** Se dibuja el avance en tiempo real a través de `WebGUI.showNumpy()` pintando obstáculos (negro), celdas visitadas (verde), puntos de retorno (azul) y puntos críticos (rojo).

---

### 4. Navegación local y control reactivo
La función `move_to_cell()` ejecuta físicamente los movimientos planificados en el simulador:

* **Control proporcional:** Ajusta las velocidades lineal (`HAL.setV`) y angular (`HAL.setW`) según la distancia y el error de orientación (`error_yaw`) al centro de la celda objetivo.
* **Respuesta rápida al Bumper:** Mediante `HAL.getBumperData()`, si se detecta colisión física contra un obstáculo no mapeado, el robot ejecuta una maniobra de emergencia retrocediendo y girando para desbloquearse inmediatamente.

---

### 5. Conclusiones, problemas y resultados






























