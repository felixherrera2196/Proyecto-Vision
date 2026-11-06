# ZED 2i: construir y guardar un mapa 3D con ROS 2

Esta guía empieza **con todas las terminales y RViz2 cerrados**. Utiliza la instalación que ya funciona en Ubuntu 22.04 con ROS 2 Humble y el workspace `~/ros2_ws`.

En el equipo de Felix, `~/ros2_ws` es `/home/felix/ros2_ws`. No necesitas volver a instalar ni compilar para cada sesión.

Para instalar en otra PC, consulta el [tutorial completo](Tutorial_ZED_2i_ROS2_Humble_mapeo_3D_paso_a_paso.md).

> El mapa en vivo y la solución para mantenerlo actualizándose se comprobaron en el equipo original. Los pasos de grabación y reproducción siguientes todavía deben verificarse allí. Guardaremos mensajes de ROS 2 en un **bag**; esto no es una exportación a PLY, PCD ni una malla.

## 1. Preparar la cámara

1. Conecta la ZED 2i a un puerto USB 3.
2. Asegúrate de que ningún visor de ZED esté usando la cámara.
3. Trabaja en una habitación iluminada, preferiblemente con muebles y superficies con textura.
4. Abre una terminal con **Ctrl+Alt+T**.

Llamaremos a esta ventana **terminal 1**. Cada terminal de esta guía debe permanecer abierta mientras se utiliza.

## 2. Terminal 1: iniciar la cámara con el mapeo activado

Copia el bloque completo:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
ros2 launch zed_wrapper zed_camera.launch.py \
  camera_model:=zed2i \
  param_overrides:="mapping.mapping_enabled:=true"
```

Este comando inicia la cámara y activa el mapeo espacial. El seguimiento de posición ya estaba activado en la configuración comprobada.

Espera a que la cámara arranque. La primera ejecución puede necesitar más tiempo para optimizar el modelo de profundidad.

**No cierres esta ventana ni pulses Ctrl+C todavía.** El comando debe seguir funcionando.

## 3. Terminal 2: mantener las actualizaciones del mapa

Abre otra terminal con **Ctrl+Alt+T** y ejecuta:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

ros2 topic hz /zed/zed_node/point_cloud/cloud_registered
```

Deja el comando ejecutándose. Deberían aparecer mediciones de frecuencia tras unos segundos.

**¿Para qué sirve?** En el equipo original, el mapa se congelaba cuando la nube instantánea no tenía un suscriptor. Este comando mantiene esa suscripción sin mostrar la nube instantánea. Es la solución temporal que permitió seguir actualizando el mapa; otras versiones podrían no necesitarla.

## 4. Terminal 3: abrir RViz2

Abre otra terminal y ejecuta:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2
```

Dentro de RViz2:

1. Localiza el panel **Displays**.
2. En **Global Options → Fixed Frame**, escribe o selecciona `map`.
3. Pulsa **Add**.
4. Selecciona la pestaña **By topic**.
5. Busca `/zed/zed_node/mapping/fused_cloud`.
6. Selecciona su opción **PointCloud2** y pulsa **OK**.
7. En las propiedades del display, deja **Decay Time = 0**.
8. Si aparece un aviso de incompatibilidad de QoS, cambia **Reliability Policy** a **Best Effort**.

Si tenías una configuración de RViz2 guardada, desmarca la nube `/zed/zed_node/point_cloud/cloud_registered`. Para ver el mapa acumulado basta con mostrar `fused_cloud`; la terminal 2 mantiene la otra suscripción.

**No aumentes Decay Time para simular el mapa.** La acumulación debe hacerla el mapeo espacial, no el historial de visualización de RViz2.

## 5. Construir el mapa

1. Apunta la cámara hacia una mesa o una pared con textura.
2. Mueve la cámara lentamente.
3. Gira de forma suave hacia otras superficies.
4. Mantén parte de la escena anterior visible durante el movimiento.
5. Observa si las superficies anteriores permanecen mientras aparecen otras.
6. Empieza con una zona pequeña antes de recorrer toda la habitación.

La configuración comprobada utiliza una frecuencia objetivo de nube fusionada de **1 Hz** y un alcance de mapeo de **5 metros**. La actualización no tiene por qué ocurrir exactamente cada segundo. El alcance limita la profundidad de las observaciones, no el tamaño total del recorrido.

Evita movimientos bruscos, espejos, cristales y objetos que se estén moviendo.

**Hasta este momento el mapa está en memoria. Todavía no está guardado.**

## 6. Terminal 4: comprobar que el mapa está activo

Sin cerrar las otras ventanas, abre otra terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

ros2 param get /zed/zed_node mapping.mapping_enabled
ros2 topic list -t | grep -Ei 'fused|mapping|cloud'
```

Debes obtener `Boolean value is: True` y ver:

```text
/zed/zed_node/mapping/fused_cloud [sensor_msgs/msg/PointCloud2]
```

Si necesitas comprobar recepción:

```bash
ros2 topic hz /zed/zed_node/mapping/fused_cloud
```

Espera unos segundos y pulsa **Ctrl+C en esta terminal 4** para terminar esa consulta. No detengas las terminales 1 y 2.

## 7. Guardar el mapa como grabación de ROS 2

Usaremos la terminal 4. La cámara, el suscriptor y RViz2 deben seguir abiertos.

### 7.1 Preparar la carpeta y el QoS de las transformaciones estáticas

Copia:

```bash
mkdir -p ~/ros2_ws/mapas

cat > ~/ros2_ws/mapas/qos_grabacion.yaml <<'YAML'
/tf_static:
  reliability: reliable
  durability: transient_local
  history: keep_last
  depth: 100
YAML
```

El archivo permite solicitar las transformaciones estáticas aunque se hayan publicado antes de empezar a grabar. Se reutiliza en futuras sesiones.

### 7.2 Iniciar la grabación

En esa misma terminal:

```bash
ros2 bag record \
  --qos-profile-overrides-path "$HOME/ros2_ws/mapas/qos_grabacion.yaml" \
  -o "$HOME/ros2_ws/mapas/mapa_$(date +%Y%m%d_%H%M%S)" \
  /zed/zed_node/mapping/fused_cloud \
  /tf \
  /tf_static
```

El nombre incluye fecha y hora para crear una carpeta distinta en cada grabación.

Graba durante al menos **10 a 20 segundos**, comprobando que el mapa continúa actualizándose. Puedes seguir moviendo la cámara durante la grabación.

Se conservarán las nubes fusionadas que se publiquen durante ese intervalo y las transformaciones recibidas. El bag contiene una secuencia de mensajes; no es una única nube exportada. Las grabaciones largas pueden ocupar bastante espacio.

### 7.3 Finalizar y cerrar correctamente el archivo

Pulsa **Ctrl+C solamente en la terminal 4** y espera a que termine la grabación y vuelva el prompt.

No cierres la ventana a la fuerza durante el guardado.

### 7.4 Comprobar lo guardado

Lista las carpetas:

```bash
ls -lt ~/ros2_ws/mapas
```

Busca la carpeta que empiece con `mapa_`. Para inspeccionarla, copia el siguiente comando **sustituyendo el nombre de ejemplo por el que realmente aparece**:

```bash
ros2 bag info ~/ros2_ws/mapas/mapa_AAAAMMDD_HHMMSS
```

Comprueba que:

- `/zed/zed_node/mapping/fused_cloud` aparece con **más de cero mensajes**.
- Aparecen los tópicos de transformaciones grabados.
- La duración corresponde aproximadamente al tiempo de grabación.

Si no hay mensajes de nube, no des el mapa por guardado: revisa que se estuviera actualizando y repite la grabación. Si faltan transformaciones estáticas, revisa los avisos de QoS y comprueba después la reproducción.

Conserva **toda la carpeta del bag**, incluido `metadata.yaml` y sus archivos de datos.

## 8. Cerrar la sesión después de guardar

Cuando la grabación ya haya terminado y hayas comprobado su contenido:

1. En la **terminal 2**, pulsa **Ctrl+C** para detener la consulta de frecuencia.
2. En la **terminal 1**, pulsa **Ctrl+C** para detener la cámara y espera a que termine.
3. Cierra RViz2.
4. Ya puedes cerrar las terminales.

**Reiniciar la cámara no recupera automáticamente el mapa anterior.** El bag conserva los datos grabados para reproducirlos; no reanuda por sí solo la reconstrucción.

## 9. Abrir después el mapa grabado

Esta parte reproduce lo guardado sin necesitar la cámara.

### 9.1 Evitar mezclar datos en vivo y grabados

Detén cualquier lanzamiento de ZED que siga activo y cierra el RViz2 anterior.

### 9.2 Abrir RViz2 con reloj de reproducción

En una terminal nueva:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2 --ros-args -p use_sim_time:=true
```

Configura **Fixed Frame = map**. Si el tópico aún no aparece para añadirlo, espera a iniciar la reproducción.

### 9.3 Reproducir el bag

En otra terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

ros2 bag play ~/ros2_ws/mapas/mapa_AAAAMMDD_HHMMSS --clock --loop
```

**Sustituye `mapa_AAAAMMDD_HHMMSS` por el nombre real.** `--loop` repite la grabación para que tengas tiempo de configurar la vista; detén la reproducción con **Ctrl+C** cuando termines.

En RViz2 añade `/zed/zed_node/mapping/fused_cloud` como **PointCloud2** y deja **Decay Time = 0**. Si aparece un aviso de fiabilidad, prueba **Best Effort**.

Si no se ve, revisa el estado del display, el contenido del bag, las transformaciones y el reloj de simulación. La reproducción es la comprobación final de que la grabación sirve para volver a visualizar el mapa.

## 10. Guardar también la vista de RViz2

Para evitar configurar los displays cada vez:

1. En RViz2 selecciona **File → Save Config As**.
2. Guarda la vista en `~/ros2_ws/zed_mapeo.rviz`.

Para abrir esa configuración en una sesión en vivo:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash

__NV_PRIME_RENDER_OFFLOAD=1 \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
rviz2 -d ~/ros2_ws/zed_mapeo.rviz --ros-args -p use_sim_time:=false
```

**Guardar la vista de RViz2 no guarda el mapa.** La vista conserva displays y ajustes; el bag conserva mensajes.

## 11. Qué se guarda y qué no

| Opción | Qué conserva |
|---|---|
| Configuración `.rviz` | Displays, cámara de visualización y ajustes de RViz2 |
| Grabación rosbag2 | Mensajes de nube y transformaciones recibidos |
| Servicio `save_area_memory` | Información del seguimiento para relocalización; no equivale a exportar la nube |
| PLY o PCD | Archivo de nube para otras herramientas; requiere una exportación adicional no incluida aquí |

En la instalación comprobada, la consulta de servicios solo mostró `enable_mapping` y `save_area_memory` entre los relacionados con mapeo y guardado. Por eso esta guía utiliza una grabación del tópico y no inventa un servicio para exportar PLY.

## 12. Borrar un mapa guardado

Cada grabación está dentro de una carpeta independiente en `~/ros2_ws/mapas`. Para borrar un mapa debes eliminar **su carpeta completa**, no únicamente `metadata.yaml` o uno de sus archivos de datos.

### 12.1 Detener el uso de esa grabación

Si estás grabando o reproduciendo el mapa que quieres borrar, pulsa **Ctrl+C en la terminal correspondiente** y espera a que termine. No borres una grabación mientras se está escribiendo o reproduciendo.

### 12.2 Localizar el mapa

Abre una terminal y ejecuta:

```bash
ls -lt ~/ros2_ws/mapas
```

Identifica la carpeta por su fecha y hora. Para conocer su tamaño:

```bash
du -sh ~/ros2_ws/mapas/mapa_AAAAMMDD_HHMMSS
```

**Sustituye `mapa_AAAAMMDD_HHMMSS` por el nombre exacto de la grabación.** No copies ese marcador literalmente.

Si quieres confirmar su contenido antes de borrarla:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 bag info ~/ros2_ws/mapas/mapa_AAAAMMDD_HHMMSS
```

### 12.3 Eliminar únicamente esa carpeta

Cuando hayas comprobado que es la grabación correcta, ejecuta con su nombre real:

```bash
rm -rI -- ~/ros2_ws/mapas/mapa_AAAAMMDD_HHMMSS
```

La opción es una **I mayúscula**. Solicita confirmación antes del borrado recursivo. Lee la ruta que muestra la pregunta y confirma solamente si corresponde al mapa elegido; si tienes dudas, pulsa **Ctrl+C** para cancelar.

**Este comando no envía los archivos a la papelera. El borrado es permanente.** Si necesitas conservar el mapa, copia antes toda su carpeta a otro disco.

No uses comodines como `mapa_*`: seleccionarían varias grabaciones. No borres la carpeta general `mapas` ni `qos_grabacion.yaml`, que se reutiliza para grabar.

### 12.4 Comprobar el resultado

```bash
ls -lt ~/ros2_ws/mapas
```

La carpeta elegida ya no debe aparecer. Las demás grabaciones deben permanecer. Borrar un bag del disco no reinicia ni borra el mapa que esté construyendo la cámara en memoria.

## 13. Si algo no funciona

| Problema | Acción |
|---|---|
| `ros2: command not found` | Ejecuta el `source /opt/ros/humble/setup.bash` de esa terminal. |
| No encuentra `zed_wrapper` | Ejecuta `source ~/ros2_ws/install/setup.bash` y verifica la instalación. |
| No abre la cámara | Cierra el visor del SDK y revisa cable, puerto USB y mensajes de la terminal 1. |
| No aparece `fused_cloud` | Comprueba que lanzaste con `mapping.mapping_enabled:=true`. |
| El mapa se congela | Comprueba que el comando de la terminal 2 sigue ejecutándose. |
| Error de transformaciones en RViz2 | Comprueba `Fixed Frame = map` y el estado de TF. |
| El bag tiene cero mensajes de nube | Mantén el mapa actualizándose y repite la grabación. |
| Solo tienes un archivo `.rviz` | Aún no has guardado los puntos; realiza la sección 7. |

Cada vez que cierres todas las ventanas y quieras empezar **un mapa nuevo**, vuelve a la sección 1.
