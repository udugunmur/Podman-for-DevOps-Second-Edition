# Tabla de Contenidos: Podman for DevOps (Segunda Edición)

## Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

### Capítulo 1: Introducción a la Tecnología de Contenedores
- Requisitos técnicos
- Convenciones del libro
- ¿Qué son los contenedores?
- ¿Por qué necesito un contenedor?
- ¿De dónde provienen los contenedores?
- ¿Dónde se utilizan los contenedores hoy en día?
- Resumen
- Lecturas adicionales

---

### Capítulo 2: Comparación entre Podman y Docker
- Requisitos técnicos
- Arquitectura del demonio de contenedores de Docker
- Arquitectura sin demonio (*daemonless*) de Podman
- Las principales diferencias entre Docker y Podman
- Resumen
- Lecturas adicionales

---

### Capítulo 3: Ejecución del Primer Contenedor
- Requisitos técnicos
- Elección de un sistema operativo y método de instalación
- Preparación del entorno
- Ejecutar tu primer contenedor
- Resumen
- Lecturas adicionales

---

### Capítulo 4: Gestión de Contenedores en Ejecución
- Requisitos técnicos
- Gestión de imágenes de contenedores
- Operaciones con contenedores en ejecución
- Inspección de información de contenedores
- Captura de logs de contenedores
- Ejecución de procesos dentro de un contenedor en ejecución
- Ejecución de contenedores en pods
- Resumen

---

### Capítulo 5: Implementación de Almacenamiento para los Datos del Contenedor
- Requisitos técnicos
- ¿Por qué es importante el almacenamiento para los contenedores?
- Características de almacenamiento de los contenedores
- Copiar archivos hacia y desde un contenedor
- Adjuntar almacenamiento del host a un contenedor
- Adjuntar otros tipos de almacenamiento a un contenedor
- Resumen
- Lecturas adicionales

---

## Parte 2: Construcción de tu Contenedor desde Cero con Buildah

### Capítulo 6: Conoce Buildah – Construcción de Contenedores desde Cero
- Requisitos técnicos
- Construcción de imágenes básicas con Podman
- Presentación de Buildah, el compañero de Podman
- Preparación de nuestro entorno
- Elección de nuestra estrategia de construcción
- Construcción de imágenes desde cero (*from scratch*)
- Construcción de imágenes desde un Dockerfile
- Resumen
- Lecturas adicionales

---

### Capítulo 7: Integración con Procesos de Construcción de Aplicaciones Existentes
- Requisitos técnicos
- Construcciones de contenedores multietapa (*multistage builds*)
- Ejecución de Buildah dentro de un contenedor
- Integración de Buildah con constructores personalizados
- Resumen
- Lecturas adicionales

---

### Capítulo 8: Elección de la Imagen Base del Contenedor
- Requisitos técnicos
- El formato de imagen de la Open Container Initiative (OCI)
- ¿De dónde provienen las imágenes de contenedores?
- Fuentes confiables de imágenes de contenedores
- Presentación de la Imagen Base Universal (*Universal Base Image* - UBI)
- Resumen
- Lecturas adicionales

---

### Capítulo 9: Publicación de Imágenes en un Registro de Contenedores
- Requisitos técnicos
- ¿Qué es un registro de contenedores?
- Registros de contenedores en la nube y locales (*on-premises*)
- Gestión de imágenes de contenedores con Skopeo
- Ejecución de un registro de contenedores local
- Resumen
- Lecturas adicionales

---

### Capítulo 10: Aseguramiento de Contenedores
- Requisitos técnicos
- Ejecución de contenedores sin privilegios de root (*rootless*) con Podman
- Evitar la ejecución de contenedores con UID 0
- Firma de nuestras imágenes de contenedores
- Comprensión de la interacción de SELinux con los contenedores
- Resumen
- Lecturas adicionales

---

## Parte 3: Gestión e Integración Segura de Contenedores

### Capítulo 11: Solución de Problemas y Monitorización de Contenedores
- Requisitos técnicos
- Solución de problemas en contenedores en ejecución
- Monitorización de contenedores mediante comprobaciones de salud (*health checks*)
- Inspección de los resultados de construcción de tus contenedores
- Solución avanzada de problemas con nsenter
- Resumen
- Lecturas adicionales

---

### Capítulo 12: Implementación de Conceptos de Redes de Contenedores
- Requisitos técnicos
- Configuración de redes de contenedores y puesta a punto de Podman
- Interconexión de dos o más contenedores
- Exposición de contenedores fuera del host subyacente
- Comportamiento de red en contenedores sin root (*rootless*)
- Resumen
- Lecturas adicionales

---

### Capítulo 13: Consejos y Trucos para la Migración desde Docker
- Requisitos técnicos
- Migración de imágenes existentes y experimentación con alias de comandos
- Comandos de Podman frente a comandos de Docker
- Uso de Docker Compose con Podman
- Resumen
- Lecturas adicionales

---

### Capítulo 14: Interacción con systemd y Kubernetes
- Requisitos técnicos
- Configuración de los prerrequisitos para el sistema operativo host
- Integración de contenedores en systemd mediante Quadlets
- Generación de recursos YAML de Kubernetes
- Ejecución de archivos de recursos de Kubernetes en Podman
- Prueba de los resultados en Kubernetes
- Resumen
- Lecturas adicionales

---

### Capítulo 15: Gestión de Cargas de Trabajo de Contenedores, Kubernetes e IA desde una Interfaz Gráfica
- Requisitos técnicos
- Configuración de los prerrequisitos para el sistema operativo host
- Construcción, ejecución, detención y gestión sencilla de contenedores mediante una interfaz intuitiva
- Conexión e interacción directa con clústeres de Kubernetes
- Despliegue de inferencia de IA local para la integración de aplicaciones
- Integración de un modelo de IA en ejecución en Podman para construir tu propio caso de uso de chatbot
- Resumen
- Lecturas adicionales

---

### Capítulo 16: Desbloquea tus Beneficios Exclusivos
- Desbloquea los beneficios gratuitos de este libro en tres sencillos pasos
- Índice
