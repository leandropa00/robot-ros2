# robot-ros2

Repositorio con el desarrollo de una actividad de **robótica con ROS2** (Robot Operating System 2): levantamiento de un entorno de simulación, construcción de un mapa del entorno con SLAM y navegación del robot. Contiene las evidencias de funcionamiento generadas durante la actividad.

## Tabla de Contenidos

1. [Objetivos y Alcance](#1-objetivos-y-alcance)
2. [Arquitectura y Componentes](#2-arquitectura-y-componentes)
3. [Diagrama de Bloques / Flujo](#3-diagrama-de-bloques--flujo)
4. [Desarrollo e Implementación (Código/Configuración)](#4-desarrollo-e-implementación-códigoconfiguración)
5. [Pruebas y Evidencias de Funcionamiento](#5-pruebas-y-evidencias-de-funcionamiento)
6. [Registro de Incidencias, Análisis y Conclusiones](#6-registro-de-incidencias-análisis-y-conclusiones)

---

## 1. Objetivos y Alcance

### Objetivo general
Poner en marcha una pila de navegación y mapeo en **ROS2**, controlando un robot móvil en un entorno simulado, generando un mapa del entorno (**SLAM**) y verificando que el sistema funciona de extremo a extremo.

### Objetivos específicos
- Levantar una **simulación** de un robot móvil (TurtleBot/Gazebo) con ROS2.
- Construir un **mapa del entorno** mediante SLAM (láser + odometría) y exportarlo.
- Publicar y visualizar el **grafo de comunicación** (`rqt_graph`) para validar los nodos y tópicos activos.
- Registrar **evidencias de funcionamiento** (captura animada del robot navegando/mapeando).

### Alcance
- Trabajo realizado en ROS2 con el paquete de mapas y recursos de simulación/navegación.
- Dentro de alcance: mapa de ocupación (`.pgm` + `.yaml`) y evidencia visual (`.gif`) del funcionamiento.
- Fuera de alcance: control físico de hardware real, despliegue en el robot físico y desarrollo de paquetes ROS2 de producción.

---

## 2. Arquitectura y Componentes

### Plataforma / marco de trabajo
- **ROS2** (Robot Operating System 2) como middleware de comunicación entre nodos.
- **Simulador / visor** usado para mover el robot en el entorno virtual.
- Herramienta **`rqt_graph`** para obtener el grafo de ejecución del sistema.

### Archivos que componen el repositorio

| Archivo | Tipo | Descripción |
|---|---|---|
| `ros2.pgm` | Imagen (PGM) | **Mapa de ocupación** (occupancy grid) generado por SLAM; celdas que codifican espacio libre/ocupado/desconocido. |
| `ros2.yaml` | Configuración (map_server) | Metadatos del mapa: imagen, resolución, origen, umbrales de ocupación (`map_server` de ROS2). |
| `rosgraph.svg` | Diagrama vectorial | **Grafo de cómputo** ROS2 (nodos y tópicos) exportado desde `rqt_graph`. |
| `evidencia.gif` | Imagen animada | **Evidencia de funcionamiento**: captura en movimiento del robot simulando/mapeando el entorno. |
| `nav2_commander.py` | Script Python (ROS2) | **Cliente de navegación (Nav2)**: envía objetivos `NavigateToPose` al robot para que navegue a coordenadas dadas del mapa. |
| `README.md` | Documentación | Documento actual (esta estructura). |

### Detalle del mapa (`ros2.yaml`)
```yaml
image: ros2.pgm
mode: trinary
resolution: 0.050          # 5 cm por píxel
origin: [-9.722, -5.537, 0] # desplazamiento [x, y, yaw] del mapa
negate: 0
occupied_thresh: 0.65       # >65% probabilidad de ocupado
free_thresh: 0.196          # <19.6% probabilidad de libre
```

---

## 3. Diagrama de Bloques / Flujo

```
[Simulación / Robot móvil]
        │
        ├── topic /scan (láser) ────────────────┐
        ├── topic /odom (odometría) ────────────┤
        └── topic /tf (transformaciones) ───────┘
                                                ▼
                                     [Nodo de SLAM (mapeo)]
                                                │
                                        mapa de ocupación
                                                ▼
                                       ros2.pgm + ros2.yaml
                                                │
                                        visualización (GViz)

[topic /navigate_to_pose (acción Nav2)]
        ▲
        │  objetivo de navegación
   nav2_commander.py  ────►  [Servidor Nav2] ────► robot en movimiento
```

Flujo de la actividad:
1. Se lanza la simulación con el robot móvil y sus sensores.
2. El láser y la odometría alimentan al nodo de **SLAM/mapeo**, que va construyendo el mapa del entorno en tiempo real.
3. El mapa construido se **exporta** a `ros2.pgm` (imagen de ocupación) junto con su `ros2.yaml` (metadatos resolutivos).
4. El **grafo de nodos** (`rosgraph.svg`) documenta cómo se comunican los nodos y tópicos del sistema.
5. Se captura **`evidencia.gif`** mostrando el funcionamiento del robot en el entorno.
6. Con **`nav2_commander.py`** se envían objetivos de navegación al servidor Nav2 (`navigate_to_pose`) para que el robot se desplace a las coordenadas indicadas.

---

## 4. Desarrollo e Implementación (Código/Configuración)

### Configuración del mapa (map_server)
El `ros2.yaml` define cómo el `map_server` interpreta el `.pgm`:

```yaml
# ros2.yaml
image: ros2.pgm        # archivo de imagen del mapa
mode: trinary          # celdas libre/ocupada/desconocida
resolution: 0.050      # metros por píxel
origin: [-9.722, -5.537, 0]  # posición [x, y, yaw] del origen del mapa
negate: 0              # no invertir niveles de gris
occupied_thresh: 0.65  # umbral de ocupado
free_thresh: 0.196     # umbral de libre
```

### Notas de implementación típicas de la pila (para reproducir)
- **Simulación**: lanzar el mundo de Gazebo/ignition con el robot (p. ej. TurtleBot).
- **Mapeo**: ejecutar el nodo de SLAM (p. ej. `slam_toolbox` o `gmapping`) suscrito a `/scan`, `/odom` y `/tf`.
- **Mapa → server**: `ros2 run nav2_map_server map_server --ros-args -p yaml_filename:=ros2.yaml` para servir el mapa construido.
- **Grafo**: abrir `rqt_graph` para visualizar y exportar las conexiones entre nodos/ópicos (`rosgraph.svg`).

### Navegación con `nav2_commander.py`
Script Python que actúa como **cliente de la acción `NavigateToPose`** de Nav2:

```python
# nav2_commander.py (resumen)
from nav2_msgs.action import NavigateToPose

class Nav2Commander(Node):
    def __init__(self):
        super().__init__('nav2_commander_node')
        self._action_client = ActionClient(self, NavigateToPose, 'navigate_to_pose')

    def send_goal(self, x, y, theta_z, theta_w):
        self._action_client.wait_for_server()
        # Construye el Goal: frame 'map', posición (x, y) y orientación (cuaternión)
        self._action_client.send_goal_async(goal_msg)
```

- Dependencias: `rclpy`, `nav2_msgs` (acción `NavigateToPose`) y `geometry_msgs` (`PoseStamped`).
- El **goal** se construye en el frame `map` con posición `(x, y)` y orientación expresada como **cuaternión** (`z`, `w`).
- El envío es **asíncrono** (`send_goal_async`); se verifica si Nav2 **acepta** el objetivo y luego se espera el resultado.
- Cambia el destino en `main()`:
  ```python
  nav_commander.send_goal(1.5, 0.5, 0.0, 1.0)   # X, Y, orientación Z, W
  ```
- Al finalizar la navegación se imprime el código de estado y se cierra el nodo (`rclpy.shutdown()`).

> Nota: este repositorio conserva **resultados/evidencias** de la actividad. Los scripts de lanzamiento y los paquetes ROS2 no se encuentran versionados aquí; se recomienda incluirlos en futuras iteraciones.

---

## 5. Pruebas y Evidencias de Funcionamiento

| Evidencia | Archivo | Qué valida |
|---|---|---|
| Mapa de ocupación | `ros2.pgm` + `ros2.yaml` | Que el SLAM construyó un mapa coherente del entorno (resolución 5 cm, dimensión acorde al origen `[-9.7, -5.5]`). |
| Grafo de comunicación | `rosgraph.svg` | Que los nodos y tópicos de la pila están activos y bien conectados. |
| Funcionamiento en movimiento | `evidencia.gif` | Captura animada del robot navegando/mapeando el entorno en simulación. |
| Envío de objetivos Nav2 | `nav2_commander.py` | Envía coordenadas al servidor `navigate_to_pose`; se confirma que el robot acepta el objetivo y navega (estado final reportado por la acción). |

- Se verificó que las celdas del `.pgm` presentan zonas **libres**, **ocupadas** y **desconocidas** (modo `trinary`), coherentes con un entorno mapeado.
- Los umbrales (`occupied_thresh` 0.65 / `free_thresh` 0.196) son compatibles con parámetros típicos de `map_server` en ROS2.
- El `rosgraph.svg` documenta las conexiones publicadas entre nodos relevantes de la pila (láser, odometría, mapeo).

---

## 6. Registro de Incidencias, Análisis y Conclusiones

### Incidencias y consideraciones
- **Coherencia del mapa**: la calidad del `.pgm` depende de la correcta alineación entre el láser (`/scan`), la odometría (`/odom`) y las transformaciones (`/tf`); si no se publican bien, el mapa sale distorsionado o duplicado.
- **Resolución/origen**: `ros2.yaml` fija la resolución (0.05 m/px) y el origen; si estos valores no coinciden con los del mapa generado, el `map_server` falla o localiza mal el mapa en el mundo.
- **Celdas `trinary`**: con este modo el mapa solo distingue libre/ocupado/desconocido; si se espera una gradación continua de probabilidad, usar `scale`/`raw` según la herramienta de mapeo.
- **Reproducibilidad**: al no versionar los paquetes y lanzamientos, la actividad no es 100% reproducible desde este repo; conviene añadir los scripts de lanzamiento y el workspace ROS2.

### Análisis
- La combinación `ros2.pgm` + `ros2.yaml` constituye un mapa de ocupación estándar y consumible por la pila de navegación (Nav2) para planificación y localización posteriores.
- El grafo exportado (`rosgraph.svg`) evidencia un sistema distribuido funcionando de forma correcta, con tópicos y nodos comunicándose entre sí.
- La evidencia animada (`evidencia.gif`) respalda el funcionamiento real del robot dentro del entorno mapeado.

### Conclusiones
- La actividad logra **mapear un entorno con ROS2** y entregar evidencias verificables (mapa, grafo y animación).
- La arquitectura de mapas de ocupación es portable: el mapa generado puede reutilizarse para **localización (AMCL)** y **navegación (Nav2)** en trabajos posteriores.
- Se recomienda **versionar también los paquetes y scripts de lanzamiento** para que el proyecto sea reproducible y ampliable.
