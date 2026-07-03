# Cardboard VR - Implementación de VR en Entorno de Escritorio

- **Alumno:** Ortega Plaza Diego
- **Carrera:** Ing. en Desarrollo de Software.
- **Clase:** Realidad Virtual
- **Grupo:** 06IDPRMA
- **Actividad:** Implementación de VR en Entorno de Escritorio
- **Profesor:** Gutierrez Nuñez Jose Armando
- **Fecha:** 02/07/2026

---

## Descripción del proyecto

Este proyecto implementa un sistema de realidad virtual tipo Google Cardboard desarrollado en Godot Engine. El sistema simula la visión estereoscópica dividiendo la pantalla en dos vistas (una por ojo), aplicando distorsión de lente mediante shaders y sincronizando el movimiento de cámara con el giroscopio del dispositivo.

---

## 1. Arquitectura general

El sistema se organiza en tres partes:

### 1. CardboardVRCamera (vr/scripts/cardboard_vr_camera.gd)

Es la cámara principal del sistema. Al iniciar (_ready()), en lugar de tener las cámaras ya colocadas en la escena, estas se crean por código: se instancian dos Camera3D (una por ojo), cada una dentro de un Node3D utilizado como pivote para permitir su rotación, y cada pivote dentro de su propio SubViewport. En total intervienen tres cámaras: la principal, que actúa únicamente como controladora, y las dos que renderizan la imagen de cada ojo.

### 2. CardboardView (vr/scenes/CardboardView.tscn + cardboard_view.gd)

Es un CanvasLayer con un HBoxContainer dividido en dos TextureRect (uno por ojo). A cada uno se le asigna la textura generada por su SubViewport correspondiente mediante la función SetViewPorts(). Ambos TextureRect tienen aplicado, además, el shader de distorsión de lente como material.

### 3. LensBarrelShader.gdshader

Es el shader encargado de curvar la imagen. Calcula la distancia de cada píxel respecto al centro de la imagen y, si esta se encuentra dentro de un radio determinado (distortion_radius), desplaza dicho píxel según el valor de distortion_strength. Los píxeles que, tras la distorsión, quedan fuera del rango visible se pintan de negro.

El movimiento de la cámara se controla mediante el giroscopio del dispositivo (Input.get_gyroscope()), utilizado durante el uso real del visor Cardboard.

---

## 2. Problemas técnicos encontrados y solución

### 2.1 Errores al exportar a Android

Este fue, con diferencia, el punto que más dificultades presentó durante el desarrollo. Godot depende de herramientas externas para generar un .apk, SDK de Android, un JDK, y utilidades como adb o aapt, que no se incluyen junto con el motor. La mayoría de los errores obtenidos al exportar no estaban relacionados con el código del proyecto, sino con la falta de instalación o configuración de dichas herramientas. La solución que finalmente resolvió estos inconvenientes fue instalar Android Studio completo, ya que incluye el SDK, las Command Line Tools y un JDK compatible, evitando así instalar y enlazar cada componente por separado.

A continuación se detallan los errores concretos encontrados y la solución aplicada en cada caso:

- **Error / síntoma:** Godot's Android SDK path is not set or invalid
- **Causa:** Godot no detecta automáticamente la ubicación del SDK de Android; debe indicarse manualmente.
- **Solución:** Se instaló Android Studio, se obtuvo la ruta del SDK desde el SDK Manager ("Android SDK Location") y se configuró en Godot en Editor Settings - Export - Android - Android SDK Path.
---
- **Error / síntoma:** Unable to find Android SDK Build-tools / plantilla de exportación faltante
- **Causa:** No estaban instaladas las Build-Tools ni las Platform-Tools de Android, o la versión instalada no coincidía con la requerida.
- **Solución:** Desde el SDK Manager de Android Studio (pestaña SDK Tools), se instalaron las Build-Tools, Platform-Tools y Command-line Tools de Android.
---
- **Error / síntoma:** Error al firmar el APK / keytool o jarsigner no encontrado
- **Causa:** Estas herramientas forman parte del JDK, sin un JDK instalado y correctamente referenciado, la firma del APK falla.
- **Solución:** Se utilizó el JDK incluido con Android Studio, configurando su ruta en Editor Settings - Export - Android - Java SDK Path.
---
- **Error / síntoma:** Fallos de compilación por incompatibilidad de versión de Java
- **Causa:** El JDK instalado por separado no era compatible con lo requerido por el proceso de exportación.
- **Solución:** Se reemplazó por el JDK incluido en Android Studio, apuntando esa ruta desde la configuración de exportación de Godot.
---
**Conclusión:** en este proyecto, exportar a Android sin tener Android Studio instalado prácticamente no funcionó en ningún caso. La recomendación y lo que resolvió la mayoría de los errores fue instalar Android Studio, permitir que su SDK Manager instale los componentes necesarios, y luego enlazar dichas rutas manualmente en Godot, en Editor Settings - Export - Android.

---

### 2.2 Sincronización de las dos cámaras con la posición del jugador

- **Problema:** al utilizar dos cámaras independientes para simular cada ojo, era necesario que ambas replicaran exactamente la posición del jugador (parent) en cada fotograma, además de aplicar su propio desplazamiento de separación (EyesSeparation) y de convergencia (EyeConvergencyAngle), sin que ambos ojos se desincronizan al rotar.

- **Solución:** en _process(), la posición global de ambos pivotes (LeftEyePivot / RightEyePivot) se actualiza en cada fotograma en función de parent.global_position, sumando la altura de los ojos (EyeHeight). La rotación, en cambio, se aplica de forma independiente sobre cada pivote mediante rotate_y() y rotate_object_local(). La separación entre ojos y el ángulo de convergencia se configuran una única vez en _ready(), como posición y rotación local de cada cámara respecto a su pivote, evitando así que interfieran con la rotación correspondiente al movimiento de la cabeza.

---

### 2.3 Distorsión de lente aplicada de forma independiente a cada ojo

- **Problema:** al aplicar una distorsión de barril a cada mitad de la pantalla por separado, existía el riesgo de que aparecieran bordes negros o recortes incorrectos si el cálculo no se realizaba en coordenadas UV normalizadas y relativas a cada TextureRect.

- **Solución:** el shader LensBarrelShader.gdshader opera en coordenadas UV (0.0 a 1.0) relativas al nodo en el que se aplica, centrando el cálculo mediante UV - vec2(0.5). Dado que cada TextureRect ocupa su propia mitad de la pantalla gracias al HBoxContainer, la distorsión resulta correcta para cada ojo. Los píxeles fuera de rango se pintan de negro, en lugar de recortarse o repetirse, simulando así el borde del campo de visión de la lente.

---

### 2.4 Control por mouse no funcional

- **Problema:** el script incluye lógica pensada para controlar la rotación de la cámara mediante el mouse, con el objetivo de poder probar el proyecto en computadora sin necesidad de exportarlo al celular cada vez. Sin embargo, en la práctica ese control no responde: el movimiento de cámara únicamente funciona de forma correcta a través del giroscopio de un dispositivo Android.

- **Solución / Estado actual:** se identificó que el bloque de entrada por mouse depende de varias condiciones cumplidas al mismo tiempo (que UseGysroscope esté en false, que Handle_Mouse_Capture esté activo, y que Input.mouse_mode se encuentre en MOUSE_MODE_CAPTURED), por lo que basta con que alguna de ellas no se cumpla correctamente para que ese bloque nunca llegue a ejecutarse. Hasta el momento de esta entrega no se logró determinar con certeza la causa exacta ni corregirla, por lo que queda documentada como una limitación conocida del proyecto: actualmente el control de cámara solo es completamente funcional mediante el giroscopio de un dispositivo Android.

---

## 3. Uso de Inteligencia Artificial como apoyo

Durante el desarrollo se utilizó un asistente de IA como apoyo puntual para resolver dudas de implementación, depurar errores y revisar la calidad del código propio en GDScript. A continuación se documentan, a modo de registro, algunos de los prompts utilizados durante el desarrollo.

---

### 3.1 Prompt utilizado para solicitar ayuda con - cardboard_vr_camera.gd

Este es un script en GDScript (Godot 4.6) que implementa la cámara de un visor tipo Google Cardboard. Extiende Camera3D y en _ready() crea por código dos cámaras hijas (una por ojo), cada una dentro de su propio SubViewport, con un Node3D utilizado como pivote para poder rotarlas.

[cardboard_vr_camera.gd]

El comportamiento requerido es el siguiente:

* Que ambos ojos sigan la posición del jugador (CharacterBody3D padre) en cada fotograma.
* Que la rotación de la cámara se controle mediante giroscopio del celular.

---

### 3.2 Prompt utilizado para solicitar ayuda con - LensBarrelShader.gdshader

Necesito un shader en Godot 4.6 (shader_type canvas_item) para simular un visor de realidad virtual tipo Google Cardboard. El shader se va a aplicar sobre un TextureRect que muestra la imagen de un ojo (izquierdo o derecho), y tiene que corregir la distorsión que producen las lentes del visor.

Necesito que:

* Calcule la distancia de cada pixel al centro de la imagen (en coordenadas UV, o sea de 0.0 a 1.0).
* Si esa distancia está dentro de un radio configurable, aplique una distorsión tipo "barril" (empujar el UV hacia afuera o adentro según qué tan lejos esté del centro), para compensar el efecto de la lente.
* Si al aplicar la distorsión el UV resultante queda fuera del rango 0.0-1.0, que pinte ese pixel de negro en vez de mostrar algo recortado o repetido.
* Los pixeles fuera del radio de distorsión se muestran normales, sin modificar.
* Que la fuerza y el radio de la distorsión sean uniforms ajustables desde el editor de Godot, con hint_range para poder moverlos con un slider.
