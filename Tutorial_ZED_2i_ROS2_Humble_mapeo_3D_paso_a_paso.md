# Tutorial: instalar y usar la ZED 2i en ROS 2 Humble para construir un mapa 3D

Guía para estudiantes de preparatoria e ingeniería. Incluye comandos, explicaciones y comprobaciones después de cada etapa.

## 1. Objetivo y alcance

Al terminar podrás:

1. Conectar una ZED 2i y comprobar imagen y profundidad.
2. Instalar el controlador oficial de ZED en un workspace de ROS 2.
3. Consultar tópicos y suscribirte a ellos desde la terminal.
4. Visualizar una nube de puntos en RViz2.
5. Construir un mapa 3D acumulado al mover la cámara.
6. Mantener el mapa actualizándose con la solución temporal que funcionó en el equipo original.
7. Grabar los mensajes del mapa para conservarlos y reproducirlos.

La instalación, compilación, visualización y actualización del mapa se comprobaron durante esta sesión. La grabación y reproducción se incluyen como pasos posteriores: todavía no se probaron en el equipo original. Esta guía no incluye exportación a PLY/PCD ni un nodo Python propio; las suscripciones se realizan con herramientas de ROS 2.

## 2. Conceptos antes de empezar

| Concepto | Explicación |
|---|---|
| SDK de ZED | Software de Stereolabs que obtiene imágenes, profundidad y seguimiento de la cámara. |
| Wrapper de ROS 2 | Adaptador que conecta el SDK con ROS 2. |
| Nodo | Programa que realiza una tarea dentro de ROS 2. |
| Tópico | Canal con nombre por el que circulan mensajes. |
| Publicador | Parte de un nodo que envía mensajes a un tópico. |
| Suscriptor | Parte de un programa que recibe mensajes de un tópico. |
| Nube de puntos | Conjunto de puntos con coordenadas 3D; puede incluir color. |
| Seguimiento de posición | Estimación del movimiento de la cámara. |
| Mapeo espacial | Integración de observaciones sucesivas en un mapa. |
| RViz2 | Programa para visualizar datos de ROS 2. |
| TF y frame | TF relaciona sistemas de coordenadas, llamados frames. |

La nube instantánea muestra lo que observa la cámara ahora. La nube fusionada integra superficies observadas desde distintas posiciones. Aumentar `Decay Time` en RViz2 deja mensajes antiguos en pantalla, pero no sustituye al mapeo espacial.

## 3. Equipo y versiones de referencia

La instalación original utilizó:

| Elemento | Equipo original |
|---|---|
| Sistema | Ubuntu 22.04.5 LTS |
| ROS 2 | Humble |
| Cámara | ZED 2i |
| GPU | NVIDIA GeForce RTX 4060 Laptop, 8 GB |
| Controlador NVIDIA | 580.196.07 |
| ZED SDK | 5.5.0 |
| CUDA Toolkit | 13.4 |
| Workspace | `/home/felix/ros2_ws` |

La compilación y el mapeo funcionaron en ese equipo. Esto no demuestra que la misma combinación funcione en cualquier PC. Antes de instalar, consulta las matrices de compatibilidad del SDK, CUDA y controlador de NVIDIA. El SDK mencionado es la versión que reportó la instalación original; no es una recomendación universal.

El valor `CUDA Version` de `nvidia-smi` describe la compatibilidad del controlador. `nvcc --version` informa la versión del Toolkit instalado. Son datos distintos.

Necesitas una GPU NVIDIA compatible con el SDK elegido, conexión USB 3 adecuada, espacio libre y buena iluminación. Utiliza preferentemente el cable de la cámara y un puerto directo, sin concentrador USB.

## 4. Convenciones de esta guía

Ejecuta todos los comandos en la PC conectada a la cámara. Copia solo el contenido de los bloques, sin el prompt `felix@...$`.

Usaremos `~/ros2_ws`: `~` significa la carpeta personal del usuario. Para Felix equivale a `/home/felix/ros2_ws`; en otra PC se adapta al usuario sin cambiar los comandos.

Si un paso termina con errores, resuélvelos antes de continuar. No borres `src`, `build` o `install` para intentar corregir un error sin identificarlo.

## 5. Preparación de una PC nueva

### 5.1 Instalar los componentes base

Si la PC ya tiene Ubuntu 22.04, ROS 2 Humble y el visor ZED funcionando, pasa a la sección 5.2.

En una PC nueva, instala en este orden:

1. Ubuntu 22.04 de 64 bits.
2. Un controlador NVIDIA compatible con la GPU y el SDK elegido. Puedes usar “Software y actualizaciones → Controladores adicionales”; reinicia después de instalarlo.
3. CUDA y ZED SDK siguiendo los requisitos del instalador oficial para Ubuntu 22.04.
4. ROS 2 Humble Desktop siguiendo la documentación oficial. La edición Desktop incluye RViz2.

No se incluye una orden universal para instalar CUDA o el SDK: sus paquetes dependen de la GPU y de las versiones disponibles. Elegirlos sin consultar compatibilidad puede romper una instalación que ya funciona.

Documentación oficial:

- ROS 2 Humble: <https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html>
- ZED SDK: <https://www.stereolabs.com/developers/release/>
- Wrapper: <https://github.com/stereolabs/zed-ros2-wrapper>

### 5.2 Verificar la instalación base

```bash
lsb_release -a
ls /opt/ros
source /opt/ros/humble/setup.bash
printenv ROS_DISTRO
nvidia-smi
nvcc --version
cat /usr/local/zed/zed-config-version.cmake
readlink -f /usr/local/cuda
```

Comprueba que Ubuntu sea 22.04, que ROS muestre `humble`, que NVIDIA detecte la GPU y que el archivo de ZED informe la versión instalada. Si `nvcc` no se encuentra, verifica la instalación y el `PATH` del Toolkit antes de compilar.

### 5.3 Probar la cámara sin ROS

Conecta la cámara y ejecuta:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia ZED_Depth_Viewer
```

Las variables se utilizaron en la laptop original para seleccionar NVIDIA en aplicaciones gráficas. En una PC con una sola GPU pueden no ser necesarias.

Debes ver imagen y profundidad. Si el SDK no puede usar la cámara, corrige primero ese problema. El visor no publica tópicos de ROS 2.

**Cierra el visor antes de arrancar la cámara desde ROS 2.**

## 6. Crear o reutilizar el workspace

Instala las herramientas necesarias:

```bash
sudo apt update
sudo apt install git python3-colcon-common-extensions python3-rosdep
```

Crea la estructura si no existe:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
pwd
ls -la
```

`mkdir -p` conserva las carpetas existentes. No elimines paquetes que ya tengas. En el equipo original había un paquete llamado `practicas_python`; no es un requisito para usar la ZED.

Descarga el wrapper:

```bash
cd ~/ros2_ws/src
git clone --recurse-submodules https://github.com/stereolabs/zed-ros2-wrapper.git
```

Si `zed-ros2-wrapper` ya existe, no repitas el clon ni lo borres. Comprueba su estado:

```bash
git -C ~/ros2_ws/src/zed-ros2-wrapper status
```

La estructura principal será:

```text
~/ros2_ws/
└── src/
    ├── zed-ros2-wrapper/
    └── otros_paquetes_si_existen/
```

### 6.1 Revisar compatibilidad antes de compilar

```bash
cd ~/ros2_ws
grep -nE 'ZED SDK|Humble|CUDA|find_package\(ZED' \
  src/zed-ros2-wrapper/README.md \
  src/zed-ros2-wrapper/zed_components/CMakeLists.txt
```

El repositorio cambia con el tiempo. Comprueba que la revisión descargada admita Humble y tu SDK. Si no coincide, consulta las versiones publicadas del repositorio y sus requisitos antes de continuar.

En la sesión original, el README mencionaba SDK 5.2, el SDK instalado era 5.5.0 y la compilación tuvo éxito. No interpretes que cualquier SDK posterior será compatible.

Para poder repetir tu instalación más adelante, guarda la revisión descargada:

```bash
git -C ~/ros2_ws/src/zed-ros2-wrapper rev-parse HEAD
```

Anota ese identificador junto con las versiones del SDK y CUDA. El identificador exacto de la instalación original no se registró en la conversación.

## 7. Inicializar rosdep e instalar dependencias

`rosdep` identifica e instala dependencias del sistema que necesitan los paquetes ROS.

```bash
source /opt/ros/humble/setup.bash
rosdep update
```

Si aparece `no sources directory exists` o indica que no está inicializado:

```bash
sudo rosdep init
rosdep update
```

`sudo rosdep init` se realiza una sola vez por sistema. Si ya está inicializado, no lo repitas. Ejecuta `rosdep update` sin `sudo`.

Instala dependencias:

```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y --rosdistro humble
```

La versión utilizada indica que `zed_msgs` está disponible como paquete binario para Humble. No asumas que viene como submódulo: deja que las dependencias declaradas se resuelvan y revisa cualquier error.

## 8. Compilar el workspace

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash

colcon build \
  --symlink-install \
  --parallel-workers 2 \
  --cmake-args -DCMAKE_BUILD_TYPE=Release
```

Explicación:

- `colcon build`: compila los paquetes del workspace.
- `--symlink-install`: utiliza enlaces cuando es posible durante la instalación.
- `--parallel-workers 2`: limita a dos los paquetes construidos simultáneamente; no limita necesariamente todos los procesos internos de compilación.
- `Release`: solicita una compilación optimizada.

En el equipo original se obtuvo:

```text
Summary: 5 packages finished [41.2s]
```

Tu tiempo y número de paquetes pueden variar. Lo necesario es que no haya paquetes fallidos.

Después:

```bash
source ~/ros2_ws/install/setup.bash
ros2 pkg prefix zed_wrapper
```

Debe aparecer una ruta del workspace. Ahora existirán `build`, `install` y `log` además de `src`.

Si falla, conserva el primer error, el bloque `stderr` y el resumen. Si el problema se relaciona con CUDA, comprueba compatibilidad y rutas antes de reinstalar componentes.

## 9. Primera ejecución de la ZED 2i

En la **terminal A**, con el visor del SDK cerrado:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
ros2 launch zed_wrapper zed_camera.launch.py camera_model:=zed2i
```

Mantén esta terminal abierta. El parámetro `camera_model:=zed2i` identifica el modelo. En la primera ejecución puede haber optimización del modelo de profundidad; espera y revisa los mensajes.

## 10. Publicación y suscripción por tópicos

En la **terminal B**:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 topic list -t
```

Con los nombres predeterminados de esta instalación aparecieron:

| Tópico | Información |
|---|---|
| `/zed/zed_node/rgb/color/rect/image` | Imagen de color rectificada |
| `/zed/zed_node/depth/depth_registered` | Imagen de profundidad |
| `/zed/zed_node/point_cloud/cloud_registered` | Nube de puntos instantánea |
| `/zed/zed_node/pose` | Posición y orientación |
| `/zed/zed_node/odom` | Odometría |
| `/zed/zed_node/imu/data` | Datos de IMU |
| `/tf` y `/tf_static` | Transformaciones |

Si cambias el namespace o nombre de cámara, cambiarán los nombres. Utiliza los que muestre tu instalación.

Recibe un mensaje de posición:

```bash
ros2 topic echo /zed/zed_node/pose --once
```

Consulta la frecuencia de la nube:

```bash
ros2 topic hz /zed/zed_node/point_cloud/cloud_registered
```

Espera unos segundos y pulsa `Ctrl+C` para terminar esta consulta. Esto no cierra la cámara de la terminal A.

Consulta publicadores, suscriptores y QoS:

```bash
ros2 topic info /zed/zed_node/point_cloud/cloud_registered --verbose
```

Aquí el nodo ZED es el publicador; `echo` y `hz` son suscriptores. No necesitas escribir un programa para realizar estas primeras pruebas.

## 11. Visualizar la nube instantánea en RViz2

En la **terminal C**, si RViz2 no está abierto:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2
```

En la ventana:

1. En `Global Options → Fixed Frame`, selecciona `map` si está disponible.
2. Pulsa `Add → By topic`.
3. Selecciona `/zed/zed_node/point_cloud/cloud_registered` como `PointCloud2`.
4. Añade un display `TF` para ver los sistemas de coordenadas.
5. Deja `Decay Time` en `0` para mostrar la nube más reciente.

Si no aparecen puntos, revisa `Status` del display. Ante una incompatibilidad de fiabilidad, prueba `Reliability Policy → Best Effort`.

Si hay un error de transformaciones, consulta:

```bash
ros2 topic echo /zed/zed_node/point_cloud/cloud_registered \
  --field header --once
```

Usa temporalmente el `frame_id` obtenido como `Fixed Frame` para comprobar la nube. Para el mapa acumulado volveremos a `map`.

## 12. Revisar las opciones de mapeo de tu versión

Antes de activar parámetros, comprueba qué admite la revisión instalada:

```bash
grep -RniE \
  'mapping_enabled|pos_tracking_enabled|publish_fused|fused_pointcloud|mapping_resolution|mapping_range' \
  ~/ros2_ws/src/zed-ros2-wrapper/zed_wrapper/config

ros2 launch zed_wrapper zed_camera.launch.py --show-args
```

En la versión comprobada:

- `pos_tracking_enabled` ya estaba en `true`.
- `mapping_enabled` estaba en `false`.
- `max_mapping_range` era `5.0` metros.
- `fused_pointcloud_freq` era `1.0` Hz.
- El lanzamiento admitía `param_overrides` para sobrescribir parámetros.

Un alcance de 5 metros limita la profundidad de observaciones utilizadas; no significa que el mapa completo esté limitado a una habitación de 5 metros.

Si una revisión futura no admite estos argumentos, consulta su configuración antes de copiar el siguiente comando.

## 13. Activar el mapa acumulado

En la terminal A, pulsa `Ctrl+C` para detener el lanzamiento anterior. Espera a que termine. Puedes dejar RViz2 abierto.

Reinicia con mapeo:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
ros2 launch zed_wrapper zed_camera.launch.py \
  camera_model:=zed2i \
  param_overrides:="mapping.mapping_enabled:=true"
```

Esto activa el parámetro durante ese lanzamiento; no modifica los YAML del repositorio. Si arrancas después sin este argumento, volverás a los valores predeterminados.

En la terminal B comprueba:

```bash
ros2 param get /zed/zed_node mapping.mapping_enabled
ros2 topic list -t | grep -Ei 'fused|mapping|cloud'
```

En la instalación original apareció:

```text
Boolean value is: True
/zed/zed_node/mapping/fused_cloud [sensor_msgs/msg/PointCloud2]
/zed/zed_node/point_cloud/cloud_registered [sensor_msgs/msg/PointCloud2]
```

## 14. Mostrar el mapa en RViz2

1. Pon `Global Options → Fixed Frame` en `map`.
2. Desmarca la nube instantánea `cloud_registered`.
3. Pulsa `Add → By topic`.
4. Añade `/zed/zed_node/mapping/fused_cloud` como `PointCloud2`.
5. Deja `Decay Time` en `0`.
6. Si hay un aviso de QoS, prueba `Reliability Policy → Best Effort`.

La frecuencia configurada de 1 Hz es una frecuencia objetivo, no una garantía de actualización exacta cada segundo. Influyen el procesamiento y la disponibilidad de datos.

## 15. Solución temporal comprobada: el mapa se congela al ocultar la nube original

### Síntoma observado

El mapa fusionado se actualizaba con ambas nubes activadas en RViz2. Al desactivar la nube instantánea, el mapa dejaba de avanzar.

La prueba confirmó que mantener otro suscriptor a la nube instantánea restablecía las actualizaciones. No se investigó el código para determinar la causa exacta; no lo consideres un requisito universal del wrapper.

### Qué hacer, paso a paso

1. Deja activado el display de `/zed/zed_node/mapping/fused_cloud` en RViz2.
2. Desmarca el display de `/zed/zed_node/point_cloud/cloud_registered`.
3. Abre una terminal adicional y ejecuta:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

ros2 topic hz /zed/zed_node/point_cloud/cloud_registered
```

4. Deja el comando funcionando.
5. Mueve la cámara lentamente y observa el mapa.

Este comando se suscribe a la nube instantánea sin dibujarla. En el equipo original permitió mantener el mapa actualizándose. Es una solución temporal y añade tráfico/procesamiento; una revisión distinta podría no necesitarla.

Para repetir la sesión tal como funcionó, mantén abiertos:

| Proceso | Función |
|---|---|
| Lanzamiento de ZED con mapeo | Publicar datos y construir el mapa |
| `ros2 topic hz .../cloud_registered` | Mantener la suscripción que resolvió la congelación |
| RViz2 mostrando `fused_cloud` | Visualizar el mapa acumulado |

## 16. Cómo recorrer el espacio

1. Empieza con una habitación pequeña y bien iluminada.
2. Apunta a muebles o paredes con textura.
3. Avanza despacio y evita giros bruscos.
4. Mantén zonas comunes visibles entre una vista y la siguiente.
5. Observa desde varios ángulos las superficies que quieras reconstruir.
6. Regresa a zonas ya observadas y comprueba la coherencia del mapa.

Prueba sencilla: observa una mesa, gira lentamente hacia otra zona y comprueba que la mesa permanece en la nube fusionada.

Cristales, espejos, superficies brillantes, objetos que se mueven y paredes sin textura pueden producir huecos o errores. El resultado es una reconstrucción estimada; puede acumular deriva y no representa automáticamente un levantamiento de precisión.

## 17. Guardar datos: qué significa cada opción

El mapa todavía está en memoria mientras funciona la cámara. Reiniciar el nodo puede perderlo.

Consulta los servicios de tu instalación:

```bash
ros2 service list -t | grep -Ei 'mapping|save|export'
```

En el equipo original aparecieron únicamente:

```text
/zed/zed_node/enable_mapping [std_srvs/srv/SetBool]
/zed/zed_node/save_area_memory [zed_msgs/srv/SaveAreaMemory]
```

`save_area_memory` guarda información del seguimiento para reconocer un lugar y relocalizarse. No equivale a guardar la nube visible como archivo PLY/PCD. No se identificó un servicio de exportación de nube en esa lista.

### 17.1 Grabar el tópico con rosbag2 — paso propuesto, pendiente de probar

Mantén la cámara, RViz2 y el suscriptor de la sección 15 funcionando. En otra terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

mkdir -p ~/ros2_ws/mapas

ros2 bag record \
  -o "$HOME/ros2_ws/mapas/mapa_$(date +%Y%m%d_%H%M%S)" \
  /zed/zed_node/mapping/fused_cloud \
  /tf \
  /tf_static
```

Comprueba que se inicia la grabación y que el mapa continúa actualizándose. Déjalo al menos unos 10 segundos. La nube fusionada puede contener muchos puntos: grabaciones largas pueden ocupar bastante espacio.

Pulsa `Ctrl+C` solo en la terminal de grabación y espera a que cierre los archivos. Esto conserva mensajes recibidos durante la grabación; no crea un archivo PLY ni una malla.

Consulta las carpetas creadas:

```bash
ls -lt ~/ros2_ws/mapas
```

Con el nombre real de una carpeta, comprueba el contenido de la grabación:

```bash
ros2 bag info ~/ros2_ws/mapas/NOMBRE_REAL_DE_LA_GRABACION
```

`NOMBRE_REAL_DE_LA_GRABACION` es un marcador: sustitúyelo por el nombre generado. Comprueba que `fused_cloud` tenga mensajes. Si los frames estáticos no quedaron registrados, habrá que revisar QoS de `/tf_static` antes de depender de la grabación para reproducir toda la escena.

### 17.2 Reproducir — pendiente de probar

Una vez guardado y comprobado el bag, detén el lanzamiento de la cámara y el comando `hz` para evitar mezclar datos en vivo y grabados.

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

ros2 bag play ~/ros2_ws/mapas/NOMBRE_REAL_DE_LA_GRABACION --clock
```

Para visualizar con el reloj de reproducción, inicia una nueva ventana de RViz2:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2 --ros-args -p use_sim_time:=true
```

Selecciona `map` y añade `fused_cloud`. Si faltan transformaciones, revisa el contenido del bag. La exportación a PLY/PCD requiere un paso adicional que esta guía no implementa.

## 18. Guardar la configuración de RViz2

Cuando la vista funcione:

1. En RViz2 selecciona `File → Save Config As`.
2. Guarda, por ejemplo, `~/ros2_ws/zed_mapeo.rviz`.

Para abrir esa vista después:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2 -d ~/ros2_ws/zed_mapeo.rviz
```

Esto guarda los displays y opciones de visualización, no los puntos del mapa.

## 19. Arranque rápido después de instalar todo

### Terminal A: cámara con mapeo

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
ros2 launch zed_wrapper zed_camera.launch.py \
  camera_model:=zed2i \
  param_overrides:="mapping.mapping_enabled:=true"
```

### Terminal B: suscriptor para la solución temporal

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 topic hz /zed/zed_node/point_cloud/cloud_registered
```

### Terminal C: visualizador

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2
```

En RViz2: `Fixed Frame = map`, display de `/zed/zed_node/mapping/fused_cloud` y `Decay Time = 0`.

## 20. Problemas frecuentes

| Síntoma | Qué revisar |
|---|---|
| `ros2: command not found` | Ejecuta `source /opt/ros/humble/setup.bash`. |
| `Package not found` | Comprueba compilación y ejecuta `source ~/ros2_ws/install/setup.bash`. |
| rosdep no inicializado | Ejecuta una vez `sudo rosdep init` y luego `rosdep update`. |
| `rg` no existe | Usa `grep`; esta guía no necesita ripgrep. |
| El clon dice que la carpeta existe | Reutiliza la copia y revisa `git status`; no la borres automáticamente. |
| La cámara no abre | Cierra otros visores, revisa USB y los mensajes del SDK. |
| Nube instantánea visible, mapa no aparece | Comprueba `mapping.mapping_enabled`, seguimiento y tópico fusionado. |
| Mapa congelado al desmarcar la nube original | Aplica la suscripción de la sección 15. |
| RViz2 muestra error de TF | Revisa `Fixed Frame`, `/tf`, `/tf_static` y `frame_id`. |
| Aviso de fiabilidad QoS | Prueba `Best Effort` en el display. |
| No se actualiza exactamente cada segundo | 1 Hz es el objetivo; comprueba procesamiento y recepción con `ros2 topic hz`. |
| Compilación falla con CUDA | Guarda el error y revisa compatibilidad del controlador, Toolkit y SDK. |
| La PC se queda sin memoria al compilar | Prueba `--parallel-workers 1`; si persiste, revisa también el paralelismo interno de CMake. |

## 21. Qué conservar para otra PC

Conserva esta guía, la configuración de RViz2, los bags que hayas verificado y un registro de versiones:

```bash
lsb_release -a
printenv ROS_DISTRO
nvidia-smi
nvcc --version
cat /usr/local/zed/zed-config-version.cmake
git -C ~/ros2_ws/src/zed-ros2-wrapper rev-parse HEAD
```

En otra PC reconstruye el workspace a partir de `src` y dependencias compatibles. Evita copiar `build` e `install` como sustituto de una instalación: contienen resultados y rutas asociados al equipo original.

El resultado comprobado de esta guía es un mapa 3D fusionado que crece al mover la ZED 2i, publicado mediante ROS 2 y visible en RViz2, utilizando la suscripción adicional que mantuvo las actualizaciones en el equipo original.
