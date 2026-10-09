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

* **Cálculo de la transformación:** Se resuelve el sistema mediante `np.linalg.lstsq()`[cite: 5].

* **Escala y conversión:** Se calcula el tamaño real de la celda en metros (`size_cell_m`) a partir de la escala obtenida y la definición en píxeles (`cell_px = 30`)[cite: 5].

// añadir foto de la matriz

```

Gazebo (m)  <--->  Matriz de transformación  <--->  Mapa (px)

```

Esta relación sienta la base para construir el gridmap y posicionar al robot en su celda inicial.

---

### 2. Procesamiento del mapa y Gridmap

Cargamos el mapa de la casa mediante WebGUI y lo procesamos con OpenCV, al mapa le vamos a aplicar ina **erosión** para poder ajustar el límite de las regiones y asi asegurar que los obstaculo quedan adecuadamente delimitados antes de discretizar.

$$\text{Mapa original} \longrightarrow \text{Erosión morfológica} \longrightarrow \text{Mapa preparado}$$



































