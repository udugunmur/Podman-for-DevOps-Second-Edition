# Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

## Capítulo 2: Comparación entre Podman y Docker

Como aprendimos en el capítulo anterior, la tecnología de contenedores no es tan nueva como podríamos pensar y, por lo tanto, la implementación y la arquitectura han sido influenciadas a lo largo de los años, alcanzando su estado actual.

En este capítulo, repasaremos un poco de la historia y la arquitectura principal de los motores de contenedores Docker y Podman, completando el panorama con una comparación lado a lado para permitir que los lectores con cierta experiencia en Docker se incorporen fácilmente y conozcan las principales diferencias antes de profundizar en una exploración detallada de Podman.

Si no tienes mucha experiencia con Docker, puedes saltar fácilmente al siguiente capítulo y volver a este cuando sientas que es el momento de aprender sobre las diferencias entre los motores de contenedores Podman y Docker.

En este capítulo, cubriremos los siguientes temas principales:

- Arquitectura del demonio de contenedores de Docker
- Arquitectura sin demonio (*daemonless*) de Podman
- Las principales diferencias entre Docker y Podman

---

### Requisitos técnicos

Este capítulo no tiene ningún prerrequisito técnico; ¡siéntete libre de leerlo sin preocuparte demasiado por instalar o configurar ningún tipo de software en tu estación de trabajo!

Si deseas replicar algunos de los ejemplos que se describirán en este capítulo, necesitarás instalar y configurar Podman y Docker en tu estación de trabajo. Como describimos antes, puedes pasar fácilmente al siguiente capítulo y volver a este una vez que sientas que es hora de aprender sobre las diferencias entre los motores de contenedores Podman y Docker.

Ten en cuenta que, en el próximo capítulo, se te presentará la instalación y configuración de Podman, por lo que pronto podrás replicar cualquier ejemplo que veas en este capítulo y en los siguientes.

---

### Arquitectura del demonio de contenedores de Docker

Los contenedores son una respuesta simple e inteligente a la necesidad de ejecutar instancias de procesos aisladas. Podemos afirmar con seguridad que los contenedores son una forma de aislamiento de aplicaciones que funciona en muchos niveles, como el sistema de archivos, la red, el uso de recursos, los procesos, etc.

Como vimos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), en la sección *Contenedores frente a máquinas virtuales*, los contenedores también difieren de las máquinas virtuales porque los contenedores comparten el mismo kernel con el host, mientras que las máquinas virtuales tienen su propio kernel de sistema operativo invitado. Desde el punto de vista de la seguridad, las máquinas virtuales proporcionan un mejor aislamiento frente a posibles ataques, pero una máquina virtual consumirá habitualmente más recursos que un contenedor. Para poner en marcha un sistema operativo invitado, normalmente necesitamos asignar más RAM, CPU y almacenamiento que los recursos necesarios para iniciar un contenedor.

Allá por 2013, el motor de contenedores Docker apareció en el panorama de los contenedores y rápidamente se volvió muy popular.

Como explicamos antes, un motor de contenedores es una herramienta de software que acepta y procesa solicitudes de los usuarios para crear un contenedor; puede verse como una especie de orquestador. Por otro lado, un runtime de contenedores es una pieza de software de nivel inferior utilizada por los motores de contenedores para ejecutar contenedores en el host, gestionando el aislamiento, el almacenamiento, las redes, etc.

En las primeras etapas, el motor de contenedores Docker utilizó LXC como runtime de contenedores, pero luego lo reemplazó después de un tiempo con su propia implementación, `libcontainer`.

El motor de contenedores Docker consta de tres pilares fundamentales:

- Demonio de Docker (*Docker daemon*)
- API REST de Docker
- CLI de Docker

Estos tres pilares están representados en la siguiente arquitectura:

*Figura 2.1 – Arquitectura de Docker*

Una vez que el demonio de Docker está en ejecución, como se muestra en el diagrama anterior, puedes interactuar con él a través de un cliente Docker o una API remota. El demonio de Docker es responsable de muchas actividades locales de contenedores, así como de interactuar con registros externos para descargar (*pull*) o subir (*push*) imágenes de contenedores.

El demonio de Docker es la pieza más crítica de la arquitectura y siempre debe estar en funcionamiento; de lo contrario, ¡tus queridos contenedores no sobrevivirán por mucho tiempo! Veamos sus detalles en la siguiente sección.

#### El demonio de Docker (*The Docker daemon*)

Operando en segundo plano, los demonios son procesos que realizan tareas esenciales del sistema o brindan servicios a otros programas.

El demonio de Docker es el proceso en segundo plano que se encarga de lo siguiente:

- Escuchar solicitudes de la API de Docker
- Manejar, gestionar y comprobar los contenedores en ejecución
- Gestionar imágenes, redes y volúmenes de almacenamiento de Docker
- Interactuar con registros de imágenes de contenedores externos/remotos

Todas estas acciones deben ser indicadas al demonio a través de un cliente o llamando a su API, pero veamos cómo comunicarse con él.

#### Interacción con el demonio de Docker

Se puede contactar con el demonio de Docker a través del socket de un proceso, habitualmente disponible en el sistema de archivos de la máquina host: `/var/run/docker.sock`.

Dependiendo de la distribución de Linux de tu elección, es posible que debas configurar los permisos adecuados para que tus usuarios no root puedan interactuar con el demonio de Docker o simplemente agregar tus usuarios no privilegiados al grupo `docker`.

Como puedes ver en el siguiente comando, estos son los permisos establecidos para el demonio de Docker en un sistema operativo Fedora 40:

```bash
# ls -la /var/run/docker.sock
srw-rw----. 1 root docker 0 Aug 25 12:48 /var/run/docker.sock
```

No hay ningún otro tipo de seguridad o autenticación para un demonio de Docker habilitado de forma predeterminada, así que ten cuidado de no exponer públicamente el demonio a redes que no sean de confianza, ya que el acceso al socket del demonio equivaldría a un acceso root sin contraseña en el sistema.

#### La API REST de Docker

Una vez que el demonio de Docker está en funcionamiento, puedes comunicarte a través de un cliente o directamente mediante la API REST. A través de la API de Docker, puedes realizar todo tipo de actividades que puedes llevar a cabo a través de la herramienta de línea de comandos, como las siguientes:

- Listar contenedores
- Crear un contenedor
- Inspeccionar un contenedor
- Obtener registros (*logs*) de contenedores
- Exportar un contenedor
- Iniciar o detener un contenedor
- Matar (*kill*) un contenedor
- Renombrar un contenedor
- Pausar un contenedor

La lista continúa. Al observar una de estas API, podemos descubrir fácilmente cómo funcionan y cuál es la salida de ejemplo devuelta por el demonio.

En el siguiente comando, utilizaremos la herramienta de línea de comandos de Linux `curl` para realizar una solicitud de Protocolo de Transferencia de Hipertexto (HTTP) para obtener detalles sobre cualquier imagen de contenedor ya almacenada en la caché local del demonio:

```bash
# curl --unix-socket /var/run/docker.sock \
http://localhost/v1.41/images/json | jq
```

```json
[
  {
    "Containers": -1,
    "Created": 1724036893,
    "Id": "sha256:a95dd4643de6c46ee41ea5d8b7bb4049ce82453e8ec9c8238b14e729219541fe",
    "Labels": {
      "architecture": "x86_64",
      "build-date": "2024-08-19T03:00:40",
      "com.redhat.component": "ubi9-container",
      "com.redhat.license_terms": "https://www.redhat.com/en/about/red-hat-end-user-license-agreements#UBI",
      "description": "The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.",
      "distribution-scope": "public",
      "io.buildah.version": "1.29.0",
      "io.k8s.description": "The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.",
      "io.k8s.display-name": "Red Hat Universal Base Image 9",
      "io.openshift.expose-services": "",
      "io.openshift.tags": "base rhel9",
      "maintainer": "Red Hat, Inc.",
      "name": "ubi9",
      "release": "1181.1724035907",
      "summary": "Provides the latest release of Red Hat Universal Base Image 9.",
      "url": "https://access.redhat.com/containers/#/registry.access.redhat.com/ubi9/images/9.4-1181.1724035907",
      "vcs-ref": "e309397d02fc53f7fa99db1371b8700eb49f268f",
      "vcs-type": "git",
      "vendor": "Red Hat, Inc.",
      "version": "9.4"
    },
    "ParentId": "",
    "RepoDigests": [
      "registry.access.redhat.com/ubi9/ubi@sha256:9e6a89ab2a9224712391c77fab2ab01009e387aff42854826427aaf18b98b1ff"
    ],
    "RepoTags": [
      "registry.access.redhat.com/ubi9/ubi:9.4-1181.1724035907"
    ],
    "SharedSize": -1,
    "Size": 212442936,
    "VirtualSize": 212442936
  }
]
```

Como puedes ver en el comando anterior, la salida está en formato JSON, muy detallada con múltiples datos de metadatos, desde el nombre de la imagen del contenedor hasta su tamaño. En este ejemplo, obtuvimos previamente una versión mínima de RHEL Universal Base Image versión 9 que ocupa solo ¡200 MB!

Por supuesto, las API no están diseñadas para el consumo o la interacción humana; encajan bien con la interacción máquina a máquina y, por lo tanto, se utilizan habitualmente para la integración de software. Por esta razón, exploremos ahora cómo funciona el cliente de línea de comandos y qué opciones están disponibles.

#### Comandos del cliente de Docker (*Docker client commands*)

El demonio de Docker tiene su propio complemento que lo instruye y configura: un cliente de línea de comandos.

El cliente de línea de comandos de Docker tiene más de 30 comandos con sus respectivas opciones que permitirán a cualquier administrador de sistemas o usuario de Docker instruir y controlar el demonio y sus contenedores. La siguiente es una descripción general de los comandos más comunes:

- `build`: Construir una imagen a partir de un Dockerfile
- `cp`: Copiar archivos/carpetas entre un contenedor y el sistema de archivos local
- `exec`: Ejecutar un comando en un contenedor en ejecución
- `images`: Listar imágenes
- `inspect`: Devolver información de bajo nivel sobre objetos de Docker
- `kill`: Matar uno o más contenedores en ejecución
- `load`: Cargar una imagen desde un archivo tar o STDIN
- `login`: Iniciar sesión en un registro de Docker
- `logs`: Obtener los registros de un contenedor
- `ps`: Listar contenedores en ejecución
- `pull`: Descargar una imagen o un repositorio desde un registro
- `push`: Subir una imagen o un repositorio a un registro
- `restart`: Reiniciar uno o más contenedores
- `rm`: Eliminar uno o más contenedores
- `rmi`: Eliminar una o más imágenes
- `run`: Ejecutar un comando en un contenedor nuevo
- `save`: Guardar una o más imágenes en un archivo TAR (transmitido a stdout por defecto)
- `create`: Crear un contenedor nuevo a partir de una imagen especificada sin iniciarlo
- `start`: Iniciar uno o más contenedores detenidos
- `stop`: Detener uno o más contenedores en ejecución
- `tag`: Crear una etiqueta `TARGET_IMAGE` que hace referencia a `SOURCE_IMAGE`

La lista continúa. Como puedes ver en este subconjunto, hay muchos comandos disponibles para administrar las imágenes de contenedores y los contenedores en ejecución, incluso para exportar una imagen de contenedor o construir una nueva.

Una vez que lanzas el cliente Docker con uno de estos comandos y sus respectivas opciones, el cliente se comunicará con el demonio de Docker, donde le indicará lo que se necesita y qué acción se debe realizar. Nuevamente, el demonio aquí es el elemento clave de la arquitectura y debe estar en funcionamiento, así que asegúrate de ello antes de intentar usar el cliente Docker, así como cualquiera de sus API REST.

#### Imágenes de Docker (*Docker images*)

Una imagen de Docker es un formato introducido por Docker para gestionar datos binarios y metadatos como plantilla para la creación de contenedores. Las imágenes de Docker son paquetes para distribuir y transferir runtimes, bibliotecas y todo lo necesario para que un proceso determinado se ponga en funcionamiento.

A partir de la versión 1.12, Docker comenzó a adoptar una especificación de imagen que, a lo largo de los años, ha evolucionado hacia la versión actual, que cumple con la Especificación del Formato de Imagen de OCI (*OCI Image Format Specification*).

La primera especificación de imagen de Docker incluía muchos conceptos y campos que ahora forman parte de la especificación del formato de imagen de OCI, como los siguientes:

- Una lista de capas (*layers*)
- Fecha de creación
- Sistema operativo
- Arquitectura de CPU
- Parámetros de configuración para su uso dentro de un runtime de contenedores

El contenido de una imagen de Docker (binarios, bibliotecas, datos del sistema de archivos) se organiza en capas. Una capa es solo un conjunto de cambios en el sistema de archivos que no contiene variables de entorno ni argumentos predeterminados para un comando determinado. Estos datos se almacenan en el manifiesto de la imagen que posee los parámetros de configuración.

Pero, ¿cómo se crean estas capas y luego se agregan en una imagen de Docker? La respuesta no es tan simple. Las capas de una imagen de contenedor se componen juntas utilizando metadatos de imagen y se fusionan en una única vista del sistema de archivos. Este resultado se puede lograr de muchas maneras, pero como se anticipó en el capítulo anterior, el enfoque más común hoy en día es mediante el uso de sistemas de archivos de unión: combinando dos sistemas de archivos y proporcionando una vista única y unificada. Finalmente, cuando se ejecuta un contenedor, se crea una nueva capa efímera de lectura/escritura encima de la imagen, que se perderá tras destruir el contenedor.

Como dijimos anteriormente en este capítulo, las imágenes de contenedores y su distribución fueron la característica clave de los contenedores Docker. Por lo tanto, en la siguiente sección, veamos el elemento clave para la distribución de contenedores: los registros de Docker.

#### Registros de Docker (*Docker registries*)

Un registro de Docker es simplemente un repositorio de imágenes de contenedores de Docker que contiene los metadatos y las capas de las imágenes de contenedores para que estén disponibles para varios demonios de Docker.

Un demonio de Docker actúa como cliente para un registro de Docker a través de una API HTTP, enviando (*push*) y descargando (*pull*) imágenes de contenedores dependiendo de la acción que indique el cliente de Docker.

El uso de un registro de contenedores puede facilitar el uso de contenedores en muchas máquinas independientes. Estas máquinas se pueden configurar para descargar imágenes de contenedores desde un registro si no están presentes en la caché local del demonio de Docker. El registro predeterminado que está preconfigurado en los ajustes del demonio de Docker es Docker Hub, un registro de contenedores de Software como Servicio (SaaS) alojado por la empresa Docker en la nube. Sin embargo, Docker Hub no es el único registro; muchos otros registros de contenedores han aparecido en los últimos años.

Casi todas las empresas o comunidades que trabajan con contenedores han creado su propio registro de contenedores con una interfaz web diferente. Uno de los servicios alternativos gratuitos a Docker Hub es Quay.io, un registro de contenedores de Software como Servicio alojado por Red Hat.

Una gran alternativa a los servicios en la nube es el registro de Docker local (*on-premises*), que se puede crear a través de un contenedor en una máquina que ejecute el demonio de Docker con un solo comando:

```bash
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

No es el objetivo de este libro repasar las diversas opciones y configuraciones de Docker, pero si deseas obtener más información sobre el registro de Docker, puedes consultar la documentación principal de Docker en [https://docs.docker.com/registry/deploying/](https://docs.docker.com/registry/deploying/).

Hemos visto muchas cosas hasta ahora: la API de Docker, el cliente, el demonio, las imágenes y, finalmente, el registro; pero, como mencionamos anteriormente, todo depende del uso correcto del demonio de Docker, que siempre debe estar saludable y en funcionamiento. Por lo tanto, exploremos ahora qué sucede en caso de que deje de funcionar.

#### ¿Cómo se ve una arquitectura de Docker en ejecución?

El demonio de Docker es el elemento central clave de toda la arquitectura de Docker. Exploraremos en esta sección cómo se ven un demonio de Docker y un grupo de contenedores en ejecución.

No profundizaremos en los pasos necesarios para instalar y configurar el demonio de Docker; en su lugar, analizaremos directamente un sistema operativo preconfigurado con él:

```bash
# systemctl status docker
```

```text
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Tue 2024-09-03 20:01:46 UTC; 4min 0s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 3970 (dockerd)
      Tasks: 11
     Memory: 171.4M (peak: 237.3M swap: 4.0K swap peak: 4.0K)
        CPU: 4.062s
     CGroup: /system.slice/docker.service
             └─3970 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

Como puedes ver en el comando anterior, acabamos de verificar que el demonio de Docker está en funcionamiento, pero no es el único servicio de contenedores que se ejecuta en el sistema. El demonio de Docker tiene un compañero que omitimos en la parte anterior para mantener la descripción fácil de entender: **containerd**.

Para comprender mejor el flujo de trabajo, echa un vistazo al siguiente diagrama:

*Figura 2.2 – Ejecución de un contenedor Docker*

containerd es el proyecto que desacopla la gestión de contenedores (incluida la interacción con el kernel) del demonio de Docker, y también se adhiere al estándar OCI utilizando `runc` como runtime de contenedores.

Entonces, verifiquemos el estado de containerd en nuestro sistema operativo preconfigurado:

```bash
# systemctl status containerd
```

```text
● containerd.service - containerd container runtime
     Loaded: loaded (/usr/lib/systemd/system/containerd.service; disabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Tue 2024-09-03 20:01:45 UTC; 8min ago
       Docs: https://containerd.io
    Process: 3960 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 3962 (containerd)
      Tasks: 18
     Memory: 93.4M (peak: 111.9M swap: 3.0M swap peak: 5.6M)
        CPU: 359ms
     CGroup: /system.slice/containerd.service
             ├─3962 /usr/bin/containerd
             └─4231 /usr/bin/containerd-shim-runc-v2 -namespace moby -id a478c626278a1aa388732501c14fe9f4b4dfab70ad8d87266cdcc770adf3e93b -address /run/containerd/containerd.sock
```

Como puedes ver en la salida de consola anterior, el servicio está en funcionamiento y ha iniciado un proceso hijo: `/usr/bin/containerd-shim-runc-v2`. ¡Esto coincide perfectamente con lo que acabamos de ver en la Figura 2.2!

Ahora, revisemos nuestros contenedores en ejecución interactuando con la interfaz de línea de comandos de Docker:

```bash
# docker ps
```

```text
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
7433c0613412 centos/httpd-24-centos7:latest "container-entrypoin…" 26 minutes ago Up 26 minutes 8080/tcp, 8443/tcp funny_goodall
```

Como puedes ver, el cliente Docker confirma que tenemos un contenedor en ejecución en nuestro sistema, iniciado a través del runtime de contenedores `runc`, administrado por el servicio del sistema `containerd` y configurado a través de un demonio Docker.

Ahora que hemos introducido este nuevo elemento, containerd, veámoslo con más profundidad en la siguiente sección.

#### Arquitectura de containerd (*containerd architecture*)

La arquitectura de containerd está compuesta por varios componentes organizados en subsistemas. Los componentes que vinculan diferentes subsistemas también se denominan módulos en la arquitectura de containerd, como se puede ver en el siguiente diagrama:

*Figura 2.3 – Arquitectura de containerd*

Los dos subsistemas principales disponibles son los siguientes:

- El servicio de paquetes (*bundle service*), que extrae paquetes de las imágenes de disco.
- El servicio de ejecución (*runtime service*), que ejecuta los paquetes, creando los contenedores en tiempo de ejecución.

Los módulos principales que hacen que la arquitectura sea completamente funcional son los siguientes:

- El módulo **Executor**, que implementa el runtime de contenedores representado en la arquitectura anterior como el bloque *Runtimes*.
- El módulo **Supervisor**, que monitoriza e informa el estado del contenedor, formando parte del bloque *Containers* en la arquitectura anterior.
- El módulo **Snapshot**, que gestiona las instantáneas del sistema de archivos.
- El módulo **Events**, que recopila y consume eventos.
- El módulo **Metrics**, que exporta varias métricas a través de la API de métricas.

Los pasos necesarios por containerd para colocar un contenedor en estado de ejecución son demasiado complejos para describirlos en esta sección, pero podemos resumirlos de la siguiente manera:

1. Descargar metadatos y contenido a través de un controlador de distribución (*distribution controller*).
2. Utilizar el controlador de paquetes (*bundle controller*) para descomprimir los datos recuperados, creando instantáneas que compondrán los paquetes (*bundles*).
3. Ejecutar el contenedor a través del paquete recién creado mediante el controlador de runtime (*runtime controller*):

*Figura 2.4 – Diagrama de flujo de datos de containerd*

En esta sección, hemos descrito las características clave y los principios de diseño del motor de contenedores Docker, con su enfoque centrado en el demonio. Ahora podemos pasar a analizar la arquitectura sin demonio de Podman.

---

### Arquitectura sin demonio (*daemonless*) de Podman

**Podman** (abreviatura de *POD MANager*) es un motor de contenedores sin demonio que permite a los usuarios gestionar contenedores, imágenes y sus recursos asociados, como volúmenes de almacenamiento o recursos de red. Los usuarios que instalan Podman por primera vez pronto se dan cuenta de que no hay ningún servicio que iniciar una vez completada la instalación. ¡No se requiere ningún demonio en ejecución en segundo plano para ejecutar contenedores con Podman!

Una vez instalado, el binario de Podman actúa tanto como interfaz de línea de comandos (CLI) como motor de contenedores que orquesta la ejecución del runtime de contenedores. Las siguientes subsecciones explorarán los detalles del comportamiento y los bloques de construcción de Podman.

#### Comandos de Podman y API REST

La CLI de Podman proporciona un conjunto creciente de comandos. La lista seleccionada está disponible en [https://docs.podman.io/en/latest/Commands.html](https://docs.podman.io/en/latest/Commands.html).

La siguiente lista explora un subconjunto de los comandos más utilizados:

- `build`: Construir una imagen a partir de un Containerfile o Dockerfile
- `cp`: Copiar archivos/carpetas entre un contenedor y el sistema de archivos local
- `exec`: Ejecutar un comando en un contenedor en ejecución
- `events`: Mostrar eventos de Podman
- `generate`: Generar datos estructurados como YAML de Kubernetes o unidades de systemd
- `images`: Listar imágenes en la caché local
- `inspect`: Devolver información de bajo nivel sobre contenedores o imágenes
- `kill`: Matar uno o más contenedores en ejecución
- `load`: Cargar una imagen desde un archivo TAR de contenedor o stdin
- `login`: Iniciar sesión en un registro de contenedores
- `logs`: Obtener los registros de un contenedor
- `pod`: Gestionar pods
- `ps`: Listar contenedores en ejecución
- `pull`: Descargar una imagen o un repositorio desde un registro
- `push`: Subir una imagen o un repositorio a un registro
- `restart`: Reiniciar uno o más contenedores
- `rm`: Eliminar uno o más contenedores
- `rmi`: Eliminar una o más imágenes
- `run`: Ejecutar un comando en un contenedor nuevo
- `save`: Guardar una o más imágenes en un archivo TAR (transmitido a stdout por defecto)
- `start`: Iniciar uno o más contenedores detenidos
- `stop`: Detener uno o más contenedores en ejecución
- `system`: Administrar Podman (uso de disco, migración de contenedores, servicios de API REST, gestión de almacenamiento y limpieza/pruning)
- `tag`: Crear una etiqueta `TARGET_IMAGE` que haga referencia a `SOURCE_IMAGE`
- `unshare`: Ejecutar un comando en un namespace de usuario modificado
- `volume`: Gestionar volúmenes de contenedores (listar, limpiar, crear, inspeccionar)

En los próximos capítulos del libro, cubriremos los comandos anteriores con mayor detalle y comprenderemos cómo usarlos para gestionar el ciclo de vida completo del contenedor.

Los usuarios que ya hayan trabajado con Docker detectarán inmediatamente los mismos comandos que solían ejecutar con la CLI de Docker. Los comandos de la CLI de Podman son compatibles con los de Docker para facilitar una transición fluida entre las dos herramientas.

A diferencia de Docker, Podman no necesita un demonio de Docker en ejecución que escuche en un socket Unix para ejecutar los comandos anteriores. Sin embargo, los usuarios aún pueden optar por ejecutar un servicio de Podman y hacer que escuche en un socket Unix para exponer API REST nativas.

Al ejecutar el siguiente comando, Podman creará un endpoint de socket en la ruta de preferencia y escuchará llamadas a la API:

```bash
$ podman system service --time 0 unix://tmp/podman.sock
```

Si no se proporciona, el endpoint de socket predeterminado es `unix://run/podman/podman.sock` para servicios rootful y `unix://run/user/<UID>/podman/podman.sock`, que se supone que debe ser accedido por usuarios no root.

Como resultado, los usuarios pueden realizar llamadas a la API REST al endpoint del socket. El siguiente ejemplo consulta a Podman sobre las imágenes locales disponibles:

```bash
$ curl --unix-socket /tmp/podman.sock \
http://d/v3.0.0/libpod/images/json | jq .
```

El proyecto Podman mantiene documentación compatible con OpenAPI de las llamadas a la API REST disponibles en [https://docs.podman.io/en/latest/_static/api.html](https://docs.podman.io/en/latest/_static/api.html).

El comando entubado `jq` en el ejemplo anterior es útil para producir una salida JSON formateada y más legible. Exploraremos la API REST de Podman y la activación basada en sockets de systemd con mayor detalle en la sección de personalización posterior a la instalación del [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3). Describamos ahora los bloques de construcción de Podman con mayor detalle.

#### Bloques de construcción de Podman (*Podman building blocks*)

Podman tiene como objetivo adherirse a los estándares abiertos tanto como sea posible; por lo tanto, la mayoría de los componentes de runtime, compilación, almacenamiento y redes se basan en proyectos y estándares comunitarios. Los componentes descritos en la siguiente lista pueden verse como los principales bloques de construcción de Podman:

- El ciclo de vida del contenedor se gestiona con la biblioteca **libpod**, ya incluida en el repositorio principal de Podman: [https://github.com/containers/podman/tree/main/libpod](https://github.com/containers/podman/tree/main/libpod)
- El runtime de contenedores se basa en las especificaciones de OCI implementadas por runtimes compatibles con OCI, como `crun` y `runc`. Veremos en este capítulo cómo funcionan los runtimes de contenedores y la principal diferencia entre los mencionados anteriormente.
- Al mismo tiempo, la gestión de imágenes implementa la biblioteca **containers/image** ([https://github.com/containers/container-libs/tree/main/image](https://github.com/containers/container-libs/tree/main/image)). Este es un conjunto de bibliotecas de Go utilizadas tanto por motores de contenedores como por registros de contenedores.
- El almacenamiento de contenedores e imágenes se implementa adoptando la biblioteca **containers/storage** ([https://github.com/containers/container-libs/tree/main/storage](https://github.com/containers/container-libs/tree/main/storage)), otra biblioteca de Go para gestionar capas del sistema de archivos, imágenes de contenedores y volúmenes de contenedores en tiempo de ejecución.
- Las construcciones de imágenes se implementan con **Buildah** ([https://github.com/containers/buildah](https://github.com/containers/buildah)), que es tanto una herramienta binaria como una biblioteca para construir imágenes OCI. Cubriremos Buildah más adelante en este libro.
- La monitorización del runtime de contenedores y la comunicación con el motor se implementan con **Conmon**, una herramienta que monitoriza los runtimes OCI y es utilizada tanto por Podman como por CRI-O ([https://github.com/containers/conmon](https://github.com/containers/conmon)).
- El soporte de red de contenedores se implementa a través de una pila de red de contenedores basada en Rust llamada **Netavark**. Netavark se introdujo en Podman 4.0 junto con Container Network Interface (CNI) como el nuevo estándar predeterminado. En Podman 5.0, CNI fue declarado obsoleto. De forma predeterminada, Podman utiliza el controlador de puente básico (*bridge driver*) para la red predeterminada. Actualmente, se admiten los controladores `bridge`, `macvlan` e `ipvlan`. El enfoque orientado a plugins permite a los usuarios implementar sus propios plugins de Netavark utilizando la API de plugins de Netavark: [https://github.com/containers/netavark/blob/main/plugin-API.md](https://github.com/containers/netavark/blob/main/plugin-API.md).

Como se indicó anteriormente, Podman orquesta el ciclo de vida del contenedor gracias a la biblioteca `libpod`, descrita en la siguiente subsección.

#### La biblioteca libpod (*The libpod library*)

Las bases centrales de Podman se basan en la biblioteca `libpod`. Esta biblioteca contiene toda la lógica necesaria para orquestar el ciclo de vida del contenedor, y podemos afirmar con seguridad que el desarrollo de esta biblioteca fue la clave para el nacimiento del proyecto Podman tal como lo conocemos hoy.

La biblioteca está escrita en Go y, por lo tanto, se accede a ella como un paquete de Go y está destinada a implementar todas las funcionalidades de alto nivel del motor. Según la documentación de libpod y Podman, su alcance incluye lo siguiente:

- Gestión del formato de imagen del contenedor, que incluye imágenes OCI y Docker. Esto incluye la gestión completa del ciclo de vida de la imagen, desde la autenticación y descarga desde un registro de contenedores, el almacenamiento local de las capas de imagen y metadatos, hasta la construcción de nuevas imágenes y la subida a registros remotos.
- Gestión del ciclo de vida del contenedor: desde la creación del contenedor (con todos los pasos preliminares necesarios involucrados) y la ejecución del contenedor hasta todas las demás funcionalidades de tiempo de ejecución, como detener, matar, reanudar y eliminar, ejecución de procesos en contenedores en ejecución y registro de logs.
- Gestión de contenedores simples y pods, que son grupos de contenedores aislados que comparten namespaces juntos (en particular UTC, IPC, red y, opcionalmente, PID como una característica reciente) y también se gestionan juntos como un todo.
- Soporte para contenedores y pods sin root (*rootless*) que pueden ser ejecutados por usuarios estándar sin necesidad de escalada de privilegios.
- Gestión del aislamiento de recursos del contenedor. Esto se logra a bajo nivel con cgroups, pero los usuarios de Podman pueden interactuar utilizando opciones de CLI durante la ejecución del contenedor para gestionar la reserva de memoria y CPU o limitar la tasa de lectura/escritura en un dispositivo de almacenamiento.
- Soporte para una interfaz de CLI que se puede utilizar como una alternativa compatible con Docker. La mayoría de los comandos de Podman son iguales a los de la CLI de Docker.
- Proporcionar una API REST compatible con Docker con sockets Unix locales (no habilitada por defecto). Las API REST de Libpod proporcionan todas las funcionalidades ofrecidas por la CLI de Podman.

El paquete `libpod` interactúa, a un nivel inferior, con los runtimes de contenedores, Conmon y paquetes como `container/storage`, `container/image`, `Buildah` y `Netavark`. En la siguiente sección, nos centraremos en la ejecución del runtime de contenedores.

#### Los entornos de ejecución de contenedores OCI runc y crun (*The runc and crun OCI container runtimes*)

Como se ilustró en el capítulo anterior, un motor de contenedores se encarga de la orquestación de alto nivel del ciclo de vida del contenedor, mientras que las acciones de bajo nivel necesarias para crear y ejecutar el contenedor las proporciona un runtime de contenedores.

En los últimos años ha surgido un estándar de la industria, con la ayuda de los principales contribuyentes del entorno de contenedores: la Especificación de Runtime de OCI (*OCI Runtime Specification*). La especificación completa está disponible en [https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec).

Desde este repositorio, el documento *Runtime and Lifecycle* proporciona una descripción completa de cómo el runtime de contenedores debe manejar la creación y ejecución del contenedor: [https://github.com/opencontainers/runtime-spec/blob/master/runtime.md](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md).

`runc` ([https://github.com/opencontainers/runc](https://github.com/opencontainers/runc)) es actualmente el runtime de contenedores OCI más ampliamente adoptado. Su historia se remonta a 2015, cuando Docker anunció la escisión de toda su infraestructura técnica en un proyecto dedicado llamado `runc`.

`runc` es totalmente compatible con los contenedores de Linux y las especificaciones de runtime de OCI. El repositorio del proyecto incluye el paquete `libcontainer`, que es un paquete de Go para crear contenedores con namespaces, cgroups, capacidades y controles de acceso al sistema de archivos. `libcontainer` era un proyecto independiente de Docker anteriormente y, cuando se creó el proyecto runc, se trasladó dentro de su repositorio principal en aras de la coherencia y la claridad.

El paquete `libcontainer` define la lógica interna y la interacción del sistema de bajo nivel para inicializar un contenedor desde cero, desde el aislamiento inicial de los namespaces hasta la ejecución como PID 1 del programa binario dentro del propio contenedor.

El runtime recurre a la biblioteca `libcontainer` para realizar las siguientes tareas:

- Consumir el punto de montaje del contenedor y los metadatos del contenedor proporcionados por Podman.
- Interactuar con el kernel para iniciar el contenedor y ejecutar el proceso aislado mediante las llamadas al sistema `clone()` y `unshare()`.
- Configurar las reservas de recursos de cgroup.
- Configurar las políticas de SELinux, Seccomp y las reglas de AppArmor.

Junto con los procesos en ejecución, `libcontainer` maneja la inicialización de namespaces y descriptores de archivos, la creación del `rootFS` del contenedor y los montajes vinculados (*bind mounts*), la exportación de logs de los procesos del contenedor, la gestión de restricciones de seguridad con seccomp, SELinux y AppArmor, y la creación y mapeo de usuarios y grupos.

La arquitectura de `libcontainer` es un tema bastante complejo para este libro y obviamente necesita más investigación para comprender mejor su funcionamiento interno.

Los métodos para el sistema operativo Linux que implementan la interfaz están definidos en [https://github.com/opencontainers/runc/blob/master/libcontainer/container_linux.go](https://github.com/opencontainers/runc/blob/master/libcontainer/container_linux.go).

La ejecución de bajo nivel de las llamadas al sistema `clone()` y `unshare()` para aislar los namespaces de los procesos la maneja el paquete `nsenter`, más precisamente la función `nsexec()`. Esta es una función en C integrada en el código Go gracias al uso de `cgo`.

El código de `nsexec()` se puede encontrar aquí:
[https://github.com/opencontainers/runc/blob/master/libcontainer/nsenter/nsexec.c](https://github.com/opencontainers/runc/blob/master/libcontainer/nsenter/nsexec.c)

Junto a `runc`, se han creado muchos otros runtimes de contenedores. Un runtime alternativo que discutiremos en este libro es **crun** ([https://github.com/containers/crun](https://github.com/containers/crun)), un runtime de contenedores OCI rápido y de bajo consumo de memoria escrito completamente en C. La idea detrás de `crun` era proporcionar un runtime OCI mejorado que pudiera aprovechar el enfoque de diseño en C para lograr un runtime más limpio y ligero. Dado que ambos son runtimes OCI, un motor de contenedores puede utilizar `runc` y `crun` indistintamente.

Por ejemplo, en 2019, el proyecto Fedora tomó una decisión audaz y eligió lanzar Fedora 31 con cgroup v2 por defecto ([https://www.redhat.com/sysadmin/fedora-31-control-group-v2](https://www.redhat.com/sysadmin/fedora-31-control-group-v2)). En el momento de esta elección, `runc` aún no era capaz de gestionar contenedores bajo cgroup v2.

En consecuencia, la versión de Podman para Fedora adoptó `crun` como runtime predeterminado, ya que ya era capaz de gestionar tanto cgroup v1 como v2. Este cambio fue prácticamente transparente para los usuarios finales, quienes continuaron usando Podman de la misma manera con los mismos comandos y comportamientos. Más tarde, `runc` finalmente introdujo soporte para cgroup v2, a partir de v1.0.0-rc93, y ahora se puede utilizar en distribuciones más nuevas sin problemas. Sin embargo, el tema de cgroups no fue el único diferenciador entre `runc` y `crun`.

`crun` proporciona algunas ventajas interesantes frente a `runc`, tales como las siguientes:

- **Binario más pequeño**: Una compilación de `crun` es aproximadamente 50 veces más pequeña que una de `runc`.
- **Ejecución más rápida**: `crun` es más rápido al instrumentar el contenedor que `runc` bajo las mismas condiciones de ejecución.
- **Menor uso de memoria**: `crun` consume menos de la mitad de la memoria de `runc`. Un menor consumo de memoria es sumamente útil cuando se trata de despliegues masivos de contenedores o dispositivos IoT. También permite establecer límites de recursos muy bajos en los contenedores (4 MB y posiblemente menores).

`crun` también se puede utilizar como biblioteca e integrarse en otros proyectos compatibles con OCI. Tanto `crun` como `runc` proporcionan una interfaz de línea de comandos, pero no están diseñados para ser utilizados manualmente por los usuarios finales, quienes deben utilizar un motor de contenedores como Podman o Docker para gestionar el ciclo de vida del contenedor.

¿Qué tan fácil es alternar entre los dos runtimes en Podman? Veamos los siguientes ejemplos. Ambos ejemplos ejecutan un contenedor utilizando la opción `--runtime` para proporcionar una ruta de binario de runtime OCI. El primero ejecuta el contenedor usando `runc`:

```bash
$ podman --runtime /usr/bin/runc run --rm fedora echo "Hello World"
```

La segunda línea ejecuta el mismo contenedor con el binario `crun`:

```bash
$ podman --runtime /usr/bin/crun run --rm fedora echo "Hello World"
```

Los ejemplos asumen que ambos runtimes ya están instalados en el sistema.

Tanto `crun` como `runc` son compatibles con eBPF y CRIU.

**eBPF** significa *Extended Berkeley Packet Filter* y es una tecnología basada en el kernel que permite la ejecución de programas definidos por el usuario en el kernel de Linux para agregar capacidades adicionales al sistema sin la necesidad de recompilar el kernel o cargar módulos adicionales. Todos los programas eBPF se ejecutan dentro de una máquina virtual en un entorno aislado (*sandbox*), y su ejecución es segura por diseño. Hoy en día, eBPF está ganando impulso y atrayendo el interés de la industria, lo que lleva a una amplia adopción en diferentes casos de uso, especialmente en redes, seguridad, observabilidad y rastreo.

**Checkpoint Restore in Userspace (CRIU)** es una pieza de software que permite a los usuarios congelar un contenedor en ejecución y guardar su estado en el disco para su posterior reanudación. Las estructuras de datos guardadas en la memoria se vuelcan y se restauran en consecuencia.

Otro componente arquitectónico importante utilizado por Podman es **Conmon**, una herramienta para monitorizar el estado del runtime de contenedores. Investiguemos esto con más detalle en la siguiente subsección.

#### Conmon

Es posible que todavía tengamos algunas preguntas sobre la ejecución del runtime.

¿Cómo interactúan entre sí Podman (el motor de contenedores) y runc/crun (el runtime de contenedores OCI)? ¿Quién es responsable de lanzar el proceso de runtime de contenedores? ¿Hay alguna forma de monitorizar la ejecución del contenedor?

Presentemos el proyecto **Conmon** ([https://github.com/containers/conmon](https://github.com/containers/conmon)). Conmon es una herramienta de monitorización y comunicación que se sitúa entre el motor de contenedores y el runtime. Cada vez que se crea un nuevo contenedor, se inicia una nueva instancia de Conmon. Se desacopla (*detaches*) del proceso del gestor de contenedores y se ejecuta demonizada, iniciando el runtime del contenedor como un proceso hijo.

Si conectamos una herramienta de rastreo a un contenedor de Podman, podemos ver el orden de ejecución:

1. El motor de contenedores ejecuta el proceso Conmon, que se desacopla y se demoniza a sí mismo.
2. El proceso Conmon ejecuta una instancia de runtime de contenedores que inicia el contenedor y sale.
3. El proceso Conmon continúa ejecutándose para proporcionar una interfaz de monitorización, mientras que el proceso del gestor/motor ha salido o se ha desacoplado.

El siguiente diagrama muestra el flujo de trabajo lógico, desde la ejecución de Podman hasta el contenedor en ejecución:

*Figura 2.5 – Ejecución de un contenedor Podman*

En un sistema con muchos contenedores en ejecución, los usuarios encontrarán muchas instancias del proceso Conmon, una para cada contenedor creado. En otras palabras, Conmon actúa como un pequeño demonio dedicado al contenedor.

Veamos el siguiente ejemplo, donde se utiliza un bucle de shell simple para crear tres contenedores nginx idénticos:

```bash
# for i in {1..3}; do podman run -d --rm docker.io/library/nginx; done
592f705cc31b1e47df18f71ddf922ea7e6c9e49217f00d1af8cf18c8e5557bde
4b1e44f512c86be71ad6153ef1cdcadcdfa8bcfa8574f606a0832c647739a0a2
4ef467b7d175016d3fa024d8b03ba44b761b9a75ed66b2050de3fec28232a8a7
```

```bash
# ps aux | grep conmon
root 4272 0.0 0.5 9856 2456 ? Ss 20:22 0:00 /usr/bin/conmon --api-version 1 -c 775cb306658941f414298be6d185274dd8fc9e39e1a7bbe950b0ed87d40f749f -u 775cb306658941f414298be6d185274dd8fc9e39e1a7bbe950b0ed87d40f749f -r /usr/bin/crun [..omitted output]
root 4341 0.0 0.5 9856 2464 ? Ss 20:22 0:00 /usr/bin/conmon --api-version 1 -c 5abde0da3e6f5c6a6f7ca4d7c4698124c8bc9c297efc835d7806e9a16281abd6 -u 5abde0da3e6f5c6a6f7ca4d7c4698124c8bc9c297efc835d7806e9a16281abd6 -r /usr/bin/crun [..omitted output]
root 4409 0.0 0.5 9856 2460 ? Ss 20:22 0:00 /usr/bin/conmon --api-version 1 -c 9cd5e4461c9ab4d22baeabe0bfff3ab239d4d09b99ccd0eddf4d3c2cd2d16f48 -u 9cd5e4461c9ab4d22baeabe0bfff3ab239d4d09b99ccd0eddf4d3c2cd2d16f48 -r /usr/bin/crun [..omitted output]
```

Después de ejecutar los contenedores, un patrón simple de expresión regular aplicado a la salida del comando `ps aux` muestra tres instancias del proceso Conmon.

Incluso si el proceso inicial de Podman ya no se está ejecutando (dado que no hay demonio), todavía es posible conectarse al proceso Conmon y adjuntarse al contenedor. Al mismo tiempo, Conmon expone una forma de adjuntarse al contenedor y reenviar los logs del contenedor a archivos de registro o al diario de `systemd`.

Conmon es un proyecto ligero escrito en C. También proporciona enlaces para el lenguaje Go para pasar estructuras de configuración entre el gestor y el runtime.

#### Contenedores sin root (*Rootless containers*)

Una de las características más interesantes de Podman es la capacidad de ejecutar contenedores sin privilegios de root (*rootless*), lo que significa que los usuarios sin privilegios elevados pueden ejecutar sus propios contenedores.

Los contenedores rootless proporcionan un mejor aislamiento de seguridad y permiten que diferentes usuarios ejecuten sus propias instancias de contenedores de forma independiente. Y, gracias a `fork/exec`, un enfoque sin demonio adoptado por Podman, los contenedores rootless son sorprendentemente fáciles de gestionar. Un usuario estándar ejecuta simplemente un contenedor rootless con los comandos y argumentos habituales, como en el siguiente ejemplo:

```bash
$ podman run -d --rm docker.io/library/nginx
```

Cuando se emite este comando, Podman crea un nuevo namespace de usuario y asigna los UID entre los dos namespaces mediante un archivo `uid_map` (consulte `man user_namespaces`). Este método le permite tener, por ejemplo, un usuario root dentro del contenedor asignado a un usuario común en el host.

Los datos de imágenes y contenedores rootless se almacenan en el directorio de inicio del usuario, normalmente en `$HOME/.local/share/containers/storage`.

Podman gestiona la conectividad de red para contenedores rootless de una manera diferente a los contenedores rootful. Una comparación técnica en profundidad entre contenedores rootless y rootful, especialmente desde el punto de vista de las redes y la seguridad, se tratará más adelante en este libro, en el [Capítulo 10](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/10).

Después de un análisis en profundidad del flujo de trabajo de ejecución, resulta útil proporcionar una descripción general de las especificaciones de imagen OCI utilizadas por Podman.

#### Imágenes OCI (*OCI images*)

Podman y el paquete `container/image` implementan la Especificación del Formato de Imagen de OCI (*OCI Image Format Specification*). La especificación completa está disponible en GitHub en el siguiente enlace y se combina con la especificación de runtime de OCI: [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec).

Una imagen OCI se compone de los siguientes elementos:

- Manifiesto (*Manifest*)
- Un índice de imagen (*Image index*, opcional)
- Un diseño de imagen (*Image layout*)
- Uno o más archivos comprimidos de cambios de capa del sistema de archivos (*changeset archive*) que se descomprimirán para crear el sistema de archivos final
- Un documento de configuración de imagen para definir el orden de las capas, así como los argumentos y variables de entorno de la aplicación
- Un documento de conversión que describe cómo debe ocurrir la traducción
- Un documento de Guía de Artefactos (*Artifacts Guidance*) que describe el uso de la especificación para empaquetar contenido distinto de las imágenes OCI
- Una referencia llamada Descriptor para describir el tipo, los metadatos y la dirección de contenido (*content address*) del contenido referenciado

Veamos en detalle qué tipos de información y datos manejan los elementos más relevantes de los anteriores.

##### Manifiesto (*Manifest*)
Una especificación de manifiesto de imagen debe proporcionar imágenes direccionables por contenido. El manifiesto de la imagen contiene capas de imagen y configuraciones para una arquitectura y un sistema operativo específicos, como Linux x86_64.

Especificación: [https://github.com/opencontainers/image-spec/blob/main/manifest.md](https://github.com/opencontainers/image-spec/blob/main/manifest.md)

##### Índice de imagen (*Image index*)
Un índice de imagen es un objeto que contiene una lista de manifiestos relacionados con diferentes arquitecturas (por ejemplo, `amd64`, `arm64` o `386`) y sistemas operativos, junto con anotaciones personalizadas.

Especificación: [https://github.com/opencontainers/image-spec/blob/main/image-index.md](https://github.com/opencontainers/image-spec/blob/main/image-index.md)

##### Diseño de imagen (*Image layout*)
El diseño de imagen OCI representa la estructura de directorios de los blobs de imagen. El diseño de la imagen también proporciona las referencias de ubicación del manifiesto necesarias, así como el índice de la imagen (en formato JSON) y la configuración de la imagen. El archivo `index.json` de la imagen contiene la referencia al manifiesto de la imagen, almacenado como un blob en el paquete de la imagen OCI.

Especificación: [https://github.com/opencontainers/image-spec/blob/main/image-layout.md](https://github.com/opencontainers/image-spec/blob/main/image-layout.md)

##### Capas del sistema de archivos (*Filesystem layers*)
Dentro de una imagen, se aplican una o más capas una encima de otra para crear un sistema de archivos que el contenedor pueda utilizar.

A bajo nivel, las capas se empaquetan como archivos TAR (con opciones de compresión con gzip y zstd). La capa del sistema de archivos implementa la lógica y el orden de apilamiento de capas y cómo se aplican las capas de cambios (capas que contienen modificaciones de archivos).

Como se describió en el capítulo anterior, un sistema de archivos de copia en escritura (*copy-on-write*) o de unión se ha convertido en un estándar para gestionar el apilamiento en un enfoque similar a un grafo. Para gestionar el apilamiento de capas, Podman utiliza `overlayfs` de forma predeterminada como controlador de grafo (*graph driver*).

Especificación: [https://github.com/opencontainers/image-spec/blob/main/layer.md](https://github.com/opencontainers/image-spec/blob/main/layer.md)

##### Configuración de imagen (*Image configuration*)
Una configuración de imagen define la composición de capas de la imagen y los parámetros de ejecución correspondientes, como puntos de entrada (*entry points*), volúmenes, argumentos de ejecución o variables de entorno, así como metadatos adicionales de la imagen.

El archivo JSON de la imagen que contiene las configuraciones es un objeto inmutable; cambiarlo significa crear una nueva imagen derivada.

Especificación: [https://github.com/opencontainers/image-spec/blob/main/config.md](https://github.com/opencontainers/image-spec/blob/main/config.md)

El siguiente diagrama representa una implementación de imagen OCI, compuesta por capa(s) de imagen, índice de imagen y configuración de imagen:

*Figura 2.6 – Implementación de una imagen OCI*

Inspeccionemos un ejemplo realista de una imagen básica y ligera de `alpine`:

```bash
# tree alpine/
alpine/
├── blobs
│   └── sha256
│       ├── 03014f0323753134bf6399ffbe26dcd75e89c6a7429adfab392d64706649f07b
│       ├── 696d33ca1510966c426bdcc0daf05f75990d68c4eb820f615edccf7b971935e7
│       └── a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e
├── index.json
└── oci-layout
```

El diseño del directorio contiene un archivo `index.json`, con el siguiente contenido:

```json
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:03014f0323753134bf6399ffbe26dcd75e89c6a7429adfab392d64706649f07b",
      "size": 348,
      "annotations": {
        "org.opencontainers.image.ref.name": "latest"
      }
    }
  ]
}
```

El índice contiene una matriz de manifiestos con un solo elemento en su interior. El resumen (*digest*) del objeto es un SHA256 y corresponde al nombre de archivo como uno de los blobs enumerados anteriormente. El archivo es el manifiesto de la imagen y se puede inspeccionar:

```bash
# cat alpine/blobs/sha256/03014f0323753134bf6399ffbe26dcd75e89c6a7429adfab392d64706649f07b | jq
```

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:696d33ca1510966c426bdcc0daf05f75990d68c4eb820f615edccf7b971935e7",
    "size": 585
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e",
      "size": 2814446
    }
  ]
}
```

El manifiesto contiene referencias a la configuración y capas de la imagen. En este caso particular, la imagen tiene una sola capa. Nuevamente, sus resúmenes corresponden a los nombres de archivo de los blobs enumerados anteriormente.

El archivo de configuración muestra los metadatos de la imagen, las variables de entorno y la ejecución del comando. Al mismo tiempo, contiene referencias `DiffID` a las capas utilizadas por la imagen e información sobre la creación de la imagen:

```bash
# cat alpine/blobs/sha256/696d33ca1510966c426bdcc0daf05f75990d68c4eb820f615edccf7b971935e7 | jq
```

```json
{
  "created": "2021-08-27T17:19:45.758611523Z",
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh"
    ]
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:e2eb06d8af8218cfec8210147357a68b7e13f7c485b991c288c2d01dc228bb68"
    ]
  },
  "history": [
    {
      "created": "2021-08-27T17:19:45.553092363Z",
      "created_by": "/bin/sh -c #(nop) ADD file:aad4290d27580cc1a094ffaf98c3ca2fc5d699fe695dfb8e6e9fac20f1129450 in / "
    },
    {
      "created": "2021-08-27T17:19:45.758611523Z",
      "created_by": "/bin/sh -c #(nop) CMD [\"/bin/sh\"]",
      "empty_layer": true
    }
  ]
}
```

La capa de imagen es el tercer archivo blob. Se trata de un archivo TAR que se podría descomprimir e inspeccionar. Por razones de espacio, en este libro, el ejemplo se limita a una inspección del tipo de archivo:

```bash
# file alpine/blobs/sha256/a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e
alpine/blobs/sha256/a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e: gzip compressed data, original size modulo 2^32 5865472
```

El resultado demuestra que el archivo es un archivo TAR comprimido con gzip.

---

### Las principales diferencias entre Docker y Podman

En las secciones anteriores, repasamos las características clave de Docker y Podman, analizando la capa subyacente y descubriendo los proyectos de código abierto complementarios que hicieron que estas dos herramientas fueran únicas en su función de motor de contenedores, pero ahora es el momento de compararlas.

Como vimos anteriormente, la diferencia más significativa entre ambos es que Docker tiene un enfoque centrado en el demonio (*daemon-centric*), mientras que Podman tiene una arquitectura sin demonio (*daemonless*). El binario de Podman actúa como interfaz de línea de comandos y como motor de contenedores, y utiliza Conmon para orquestar y monitorizar el runtime de contenedores.

Al mirar bajo el capó en los aspectos internos de ambos proyectos, también encontraremos muchas otras diferencias, pero, al final, una vez iniciado el contenedor, ambos aprovechan los runtimes de contenedores estándar de OCI, aunque con algunas diferencias: Docker usa `runc` mientras que Podman usa `crun` en la mayoría de las distribuciones, con algunas excepciones; por ejemplo, el valor predeterminado de Podman en Fedora o Red Hat Enterprise Linux 9 es `crun`, mientras que todavía usa `runc` en el más conservador Red Hat Enterprise Linux 8, con `crun` como opción. Red Hat Enterprise Linux 10 solo incluye `crun` como único runtime, ya que se considera completamente estable y listo para producción.

A pesar de las ventajas de rendimiento de `crun` descritas en la sección anterior, no es el objetivo de este libro realizar una comparación de rendimiento detallada entre ambos. Los lectores interesados en el tema encontrarán fácilmente literatura sobre las diferencias de rendimiento entre los dos runtimes.

Otra gran brecha que el equipo de Docker cubrió recientemente fue el contenedor sin root (*rootless*). Podman fue el primer motor de contenedores en incorporar esta excelente característica que aumenta la seguridad y mejora el uso de contenedores en muchos contextos, pero, como mencionamos, esta característica ahora también está disponible en Docker.

Pero seamos más prácticos en las siguientes secciones, comparándolos lado a lado primero a través de la línea de comandos y luego ejecutando un contenedor.

#### Comparación de la interfaz de línea de comandos (*Command-line interface comparison*)

En esta sección, realizaremos una comparación lado a lado, examinando las CLI de Docker y Podman.

Al observar los comandos disponibles para ambas CLI, es fácil detectar las numerosas similitudes. La siguiente tabla fue truncada para mejorar la legibilidad:

| Docker | Podman |
| :--- | :--- |
| `attach` | `attach` |
| `build` | `auto-update` |
| `commit` | `build` |
| `cp` | `commit` |
| `create` | `container` |
| `diff` | `cp` |
| `events` | `create` |
| `exec` | `diff` |
| `export` | `events` |
| `history` | `exec` |
| `images` | `export` |
| `import` | `generate` |
| `info` | `healthcheck` |
| `inspect` | `help` |
| `kill` | `history` |
| `load` | `image` |
| `login` | `images` |
| `logout` | `import` |
| `logs` | `info` |
| `pause` | `init` |
| `port` | `inspect` |
| … | … |

*Tabla 2.1 – Comparación de comandos de Docker y Podman*

Como se mencionó anteriormente, Docker nació en 2013, mientras que Podman llegó 4 años después, en 2017. Podman se creó teniendo en cuenta la experiencia que los administradores de contenedores tenían con el motor de contenedores más famoso disponible en ese momento: Docker. Por esta razón, el equipo de desarrollo de Podman decidió no cambiar demasiado la apariencia de las herramientas de línea de comandos para facilitar la migración de los usuarios de Docker al recién nacido Podman.

De hecho, al comienzo de la distribución de Podman se afirmaba que si tenías scripts existentes que ejecutaban Docker, podías crear un alias y debería funcionar (`alias docker=podman`). También existía un paquete que colocaba un comando Docker simulado en `/usr/bin` que apuntaba al binario de Podman. Por esta razón, si eres un usuario de Docker, puedes esperar una transición fluida a Podman una vez que estés listo.

Otro punto importante es que las imágenes creadas con Docker son compatibles con el estándar OCI, por lo que puedes migrar o descargar fácilmente cualquier imagen que hayas utilizado anteriormente con Docker.

Si analizamos detalladamente las opciones de comando disponibles para Podman, notarás que hay algunos comandos adicionales que no están presentes en Docker, mientras que otros no están disponibles.

Por ejemplo, Podman puede gestionar, junto con los contenedores, **pods** (el nombre Podman resulta bastante prometedor en este aspecto). El concepto de pod se introdujo con Kubernetes y representa la unidad de ejecución más pequeña en un clúster de Kubernetes.

Con Podman, los usuarios pueden crear pods vacíos y luego ejecutar contenedores dentro de ellos fácilmente utilizando el siguiente comando:

```bash
$ podman pod create --name mypod
$ podman run --pod mypod -d docker.io/library/nginx
```

Esto no es tan fácil con Docker, donde los usuarios deben ejecutar primero un contenedor y luego crear otros nuevos adjuntándose al namespace de red del primer contenedor.

Podman tiene características adicionales que podrían ayudar a los usuarios a trasladar sus contenedores a entornos de Kubernetes. Mediante el comando `podman generate kube`, Podman puede crear un archivo YAML de Kubernetes para un contenedor en ejecución que se puede utilizar para crear un pod dentro de un clúster de Kubernetes.

Ejecutar contenedores como servicios de systemd es igualmente fácil anteponiendo el comando `podlet` al comando `podman run` original. Esto toma un contenedor o pod en ejecución y genera un archivo de unidad de systemd que se puede utilizar para iniciar automáticamente el servicio al arrancar el sistema.

He aquí un ejemplo notable: el proyecto OpenStack, una infraestructura de computación en la nube de código abierto, adoptó Podman como gestor predeterminado para sus servicios en contenedores cuando se despliega con TripleO. Todos los servicios son ejecutados por Podman y orquestados por systemd en el plano de control y en los nodos de cómputo.

Habiendo revisado la superficie de estos motores de contenedores y sus líneas de comandos, resumamos las diferencias bajo el capó en la siguiente sección.

#### Ejecución de un contenedor (*Running a container*)

Ejecutar un contenedor en un entorno Docker, como mencionamos anteriormente, consiste en utilizar el cliente de línea de comandos de Docker para comunicarse con el demonio de Docker, que realizará las acciones necesarias para poner el contenedor en funcionamiento. Para resumir los conceptos que explicamos en este capítulo, podemos echar un vistazo al siguiente diagrama:

*Figura 2.7 – Arquitectura simplificada de Docker*

Podman, en cambio, interactúa directamente con el registro de imágenes, el almacenamiento y con el kernel de Linux a través del proceso de runtime de contenedores (no un demonio), con Conmon como proceso de monitorización ejecutado entre Podman y el runtime OCI, como podemos esquematizar en el siguiente diagrama:

*Figura 2.8 – Arquitectura simplificada de Podman*

La diferencia fundamental entre las dos arquitecturas es la visión centrada en el demonio de Docker frente al enfoque `fork/exec` de Podman.

Este libro no entra en los pros y los contras de la arquitectura y las características del demonio de Docker. Sin embargo, es justo decir que un número significativo de usuarios de Docker estaban preocupados por este enfoque centrado en el demonio por muchas razones. He aquí algunos ejemplos:

- El demonio podría ser un punto único de fallo (*single point of failure*).
- Consumo de recursos.
- Pobre soporte para contenedores rootless debido a la necesidad de demonios de Docker y containerd por cada usuario.

A pesar de las diferencias arquitectónicas y las soluciones de alias descritas anteriormente para migrar fácilmente proyectos sin cambiar ningún script, ejecutar un contenedor desde la línea de comandos con Docker o Podman es prácticamente lo mismo para el usuario final:

```bash
$ docker run -d --rm docker.io/library/nginx
$ podman run -d --rm docker.io/library/nginx
```

Por la misma razón, la mayoría de los argumentos de línea de comandos de los comandos de la CLI se han mantenido lo más cercanos posible a la versión original en Docker.

---

### Resumen

En este capítulo, analizamos las principales diferencias entre Podman y Docker, tanto desde el punto de vista arquitectónico como de uso. Describimos los principales bloques de construcción de los dos motores de contenedores y destacamos los diferentes proyectos comunitarios que impulsan el proyecto Podman, especialmente las especificaciones de OCI y los runtimes `runc` y `crun`.

El propósito de este libro no es debatir por qué y cómo Podman podría ser una mejor opción que Docker. Creemos que todos los que trabajan con contenedores deben estar extremadamente agradecidos con la empresa y la comunidad de Docker por el gran trabajo que hicieron al llevar los contenedores a las masas y liberarlos de la adopción de nicho.

Al mismo tiempo, el enfoque evolutivo del software de código abierto facilita el nacimiento de nuevos proyectos que intentan competir para ser adoptados. Desde su nacimiento, el proyecto Podman ha crecido exponencialmente y ha ganado una base de usuarios más amplia día a día.

Comprender el funcionamiento interno del motor sigue siendo una tarea importante de todos modos. Para la resolución de problemas, el ajuste del rendimiento o incluso por simple curiosidad, invertir tiempo en comprender cómo se relacionan los componentes entre sí, leer el código y probar compilaciones es una decisión inteligente que te recompensará algún día.

En los próximos capítulos, descubriremos en detalle las características y el comportamiento de este magnífico motor de contenedores.

---

### Lecturas adicionales

Para obtener más información sobre los temas tratados en este capítulo, puedes consultar lo siguiente:

- Contenedores sin root con Podman: Los conceptos básicos: [https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics](https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics)
- Cómo hacer la transición de Docker a Podman: [https://developers.redhat.com/blog/2020/11/19/transitioning-from-docker-to-podman](https://developers.redhat.com/blog/2020/11/19/transitioning-from-docker-to-podman)
- cgroup v2: [https://github.com/opencontainers/runc/blob/master/docs/cgroup-v2.md](https://github.com/opencontainers/runc/blob/master/docs/cgroup-v2.md)
- Una introducción a crun, un runtime de contenedores rápido y de bajo consumo de memoria: [https://www.redhat.com/sysadmin/introduction-crun](https://www.redhat.com/sysadmin/introduction-crun)
- Documentación de eBPF: [https://ebpf.io/what-is-ebpf/](https://ebpf.io/what-is-ebpf/)
