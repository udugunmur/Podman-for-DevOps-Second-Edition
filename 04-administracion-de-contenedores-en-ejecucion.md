# Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

## Capítulo 4: Administración de Contenedores en Ejecución

En el capítulo anterior, aprendimos a configurar el entorno para ejecutar contenedores con Podman, cubriendo la instalación binaria para las principales distribuciones, los archivos de configuración del sistema y una primera ejecución de un contenedor de ejemplo para verificar que nuestra configuración fuera correcta. Este capítulo ofrecerá una visión general más detallada de la ejecución de contenedores, cómo administrar e inspeccionar contenedores en ejecución y cómo agrupar contenedores en pods. Este capítulo es importante para adquirir el conocimiento y la experiencia adecuados para comenzar nuestra experiencia como administradores de sistemas en tecnologías de contenedores.

En este capítulo, cubriremos los siguientes temas principales:

- Gestión de imágenes de contenedores
- Operaciones con contenedores en ejecución
- Inspección de la información de los contenedores
- Captura de logs de los contenedores
- Ejecución de procesos en un contenedor en ejecución
- Ejecución de contenedores en pods

---

### Requisitos técnicos

Antes de continuar con este capítulo y sus ejercicios, se requiere una máquina con una instancia de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40, pero se pueden reproducir en el sistema operativo de tu elección.

Por último, una buena comprensión de los temas tratados en los capítulos anteriores es útil para asimilar fácilmente los conceptos relativos a las imágenes OCI y la ejecución de contenedores.

---

### Gestión de imágenes de contenedores

En esta sección, veremos cómo encontrar y descargar (*pull*) una imagen en el sistema local, así como cómo inspeccionar su contenido. Cuando se crea y se ejecuta un contenedor por primera vez, Podman se encarga de descargar la imagen relacionada automáticamente. Sin embargo, poder descargar e inspeccionar imágenes con anticipación brinda algunas ventajas valiosas, la primera de las cuales es que un contenedor se ejecuta más rápido cuando las imágenes ya están disponibles en el almacenamiento local de la máquina.

Como dijimos en los capítulos anteriores, los contenedores son una forma de aislar procesos en un entorno aislado (*sandbox*) con namespaces y asignación de recursos independientes.

El sistema de archivos montado en el contenedor lo proporciona la imagen OCI descrita en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*.

Las imágenes OCI se almacenan y distribuyen mediante servicios especializados denominados registros de contenedores (*container registries*). Un registro de contenedores almacena imágenes y metadatos y expone servicios de API REST simples para permitir a los usuarios subir (*push*) y descargar (*pull*) imágenes.

Básicamente existen dos tipos de registros: públicos y privados. Un registro público es accesible como un servicio público (con o sin autenticación). Los principales registros públicos, como `docker.io`, `gcr.io` o `quay.io`, también se utilizan como repositorios de imágenes de proyectos de código abierto más grandes. La mayoría de estos registros también ofrecerán alojamiento privado por una tarifa.

Los registros privados se despliegan y administran dentro de una organización y pueden centrarse más en la seguridad y el filtrado de contenido. Los principales proyectos de registro de contenedores en la actualidad están graduados bajo la CNCF ([https://landscape.cncf.io/card-mode?category=container-registry&grouping=category](https://landscape.cncf.io/card-mode?category=container-registry&grouping=category)) y ofrecen funciones empresariales avanzadas para gestionar el multi-inquilino (*multitenancy*), la autenticación y el control de acceso basado en roles (RBAC), así como el escaneo de vulnerabilidades de imágenes y la firma de imágenes.

En el [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*, proporcionaremos más detalles y ejemplos de interacción con registros de contenedores.

La mayor parte de los registros públicos y privados exponen la Docker Registry HTTP API V2 ([https://docs.docker.com/registry/spec/api/](https://docs.docker.com/registry/spec/api/)). Al ser una API REST basada en HTTP, los usuarios pueden interactuar con el registro con un simple comando `curl` o diseñar sus propios clientes personalizados.

Podman ofrece una interfaz de línea de comandos (CLI) para interactuar con registros de contenedores públicos y privados, administrar inicios de sesión cuando se requiere autenticación en el registro, buscar repositorios de imágenes pasando un patrón de texto y administrar imágenes almacenadas en la caché local.

#### Búsqueda de imágenes (*Searching for images*)

El primer comando que aprenderemos a utilizar para buscar imágenes en múltiples registros es el comando `podman search`. El siguiente ejemplo muestra cómo buscar una imagen de nginx:

```bash
# podman search nginx
```

El comando anterior producirá una salida con muchas entradas de todos los registros en lista blanca (consulta la sección *Preparación del entorno | Personalización de la lista de búsqueda de registros de contenedores* del [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*). La salida será un poco desordenada, con muchas entradas de repositorios desconocidos y no confiables. También vale la pena señalar que la mayoría de los registros públicos tienen una búsqueda basada en web que es de una calidad significativamente mayor que los resultados disponibles a través de `podman search`.

En general, el comando `podman search` acepta el siguiente patrón:

```text
podman search [options] TERM
```

Aquí, `TERM` es el argumento de búsqueda. La salida resultante de una búsqueda tiene los siguientes campos:

- **INDEX**: El registro que indexa la imagen.
- **NAME**: El nombre completo de la imagen, incluido el nombre del registro y los namespaces asociados.
- **DESCRIPTION**: Una breve descripción del rol de la imagen.
- **STARS**: El número de estrellas otorgadas por los usuarios (disponible solo en registros que admiten esta función, como `docker.io`).
- **OFFICIAL**: Un valor booleano para especificar si la imagen es oficial.
- **AUTOMATED**: Un campo establecido en `OK` si la compilación de la imagen está automatizada.

> [!IMPORTANT]
> Nunca confíes en repositorios desconocidos y prefiere siempre imágenes oficiales. Al descargar imágenes de un proyecto de nicho, intenta comprender el contenido de la imagen antes de ejecutarla. Recuerda que un atacante podría ocultar código malicioso que podría ejecutarse dentro de los contenedores.
> Incluso los repositorios confiables pueden verse comprometidos en algunos casos. En escenarios empresariales, implementa la verificación de firmas de imágenes para evitar la manipulación de las mismas.

Es posible aplicar filtros a la búsqueda y refinar la salida. Por ejemplo, para refinar la búsqueda e imprimir solo imágenes oficiales, podemos agregar la siguiente opción de filtrado, que solo imprime imágenes con la marca `is-official`:

```bash
# podman search nginx --filter=is-official
```

Este comando imprimirá una línea que apunta a `docker.io/library/nginx:latest`. Esta imagen oficial es mantenida por la comunidad de nginx y se puede utilizar con mayor confianza.

Los usuarios pueden refinar el formato de salida del comando. El siguiente ejemplo muestra cómo imprimir solo el registro de la imagen y el nombre de la imagen:

```bash
# podman search fedora \
--filter is-official \
--format "table {{.Index}} {{.Name}}"
INDEX       NAME
docker.io   docker.io/library/fedora
```

El nombre de la imagen de salida tiene un patrón de nomenclatura estándar que merece una descripción detallada. El formato estándar se muestra aquí:

```text
<registry>[:<port>]/[<namespace>/]<name>:<tag>
```

Describamos los campos anteriores en detalle, de la siguiente manera:

- **registry**: Contiene el registro en el que está almacenada la imagen. La imagen de nginx de nuestro ejemplo se almacena en el registro público `docker.io`. Opcionalmente, es posible especificar un número de puerto personalizado para el registro. Por defecto, los registros exponen el puerto 5000 del Protocolo de Control de Transmisión (TCP).
- **namespace**: Este campo puede ser opcional y proporciona una estructura jerárquica que es útil para distinguir el contexto de la imagen del proveedor. El espacio de nombres puede representar la organización matriz, el nombre de usuario del propietario del repositorio o la función de la imagen.
- **name**: Contiene el nombre del repositorio de imágenes privado/público donde se almacenan todas las etiquetas. A menudo se le denomina con el nombre de la aplicación (es decir, `nginx`).
- **tag**: Cada imagen almacenada en el registro tiene una etiqueta única, asignada a un resumen del Algoritmo de Hash Seguro 256 (SHA256). La etiqueta genérica `:latest` se puede omitir en el nombre de la imagen.

La búsqueda genérica oculta las etiquetas de imagen de forma predeterminada. Para mostrar todas las etiquetas disponibles para un repositorio determinado, podemos usar la opción `--list-tags` para un nombre de imagen determinado, como se muestra a continuación:

```bash
# podman search quay.io/prometheus/prometheus --list-tags
NAME                             TAG
quay.io/prometheus/prometheus    0.19.0
quay.io/prometheus/prometheus    0.19.1
quay.io/prometheus/prometheus    0.19.2
quay.io/prometheus/prometheus    0.19.3
quay.io/prometheus/prometheus    0.20.0
quay.io/prometheus/prometheus    latest
quay.io/prometheus/prometheus    main
quay.io/prometheus/prometheus    master
quay.io/prometheus/prometheus    v1.0.0
[...output omitted...]
```

Esta opción es realmente útil para encontrar una etiqueta de imagen específica en el registro, a menudo asociada con una versión de lanzamiento de la aplicación/runtime.

> [!IMPORTANT]
> El uso de la etiqueta `:latest` puede provocar problemas de control de versiones de la imagen, ya que no es una etiqueta descriptiva. Además, generalmente se espera que apunte a la última versión de la imagen. Desafortunadamente, esto no siempre es cierto, ya que una imagen sin etiquetar podría retener la etiqueta `:latest` mientras que la última imagen enviada podría tener una etiqueta diferente. Depende del mantenedor del repositorio aplicar las etiquetas correctamente. Si el repositorio utiliza versiones semánticas (*semantic versioning*), la mejor opción es descargar la etiqueta de versión más reciente. Además de eso, otro riesgo podría ser que los mantenedores envíen actualizaciones incompatibles a `:latest` y rompan cualquier despliegue existente.

#### Descarga y visualización de imágenes (*Pulling and viewing images*)

Una vez que hemos encontrado la imagen deseada, se puede descargar usando el comando `podman pull`, de la siguiente manera:

```bash
# podman pull docker.io/library/nginx:latest
```

Observa el usuario root para ejecutar el comando Podman. En este caso, estamos descargando la imagen como root, y sus capas y metadatos se almacenan en la ruta `/var/lib/containers/storage`.

Podemos ejecutar el mismo comando como un usuario estándar ejecutando el comando en la shell de un usuario estándar, de esta forma:

```bash
$ podman pull docker.io/library/nginx:latest
```

En este caso, la imagen se descargará en el directorio de inicio del usuario en `$HOME/.local/share/containers/storage/` y estará disponible para ejecutar contenedores sin privilegios de root (*rootless*) para el usuario que ejecutó el comando.

> [!NOTE]
> También puedes intentar descargar imágenes de contenedores mediante alias. Por ejemplo, el comando `podman pull nginx` sin especificar el registro podría intentar resolver algunas de las imágenes y, en caso de que existan dudas, aparecerá un mensaje preguntando al usuario de qué registro descargar.

Los usuarios pueden inspeccionar todas las imágenes almacenadas en la caché local con el comando `podman images`, como se ilustra aquí:

```bash
# podman images
REPOSITORY                 TAG         IMAGE ID      CREATED     SIZE
docker.io/library/nginx    latest      7f553e8bbc89  6 days ago  196 MB
[...omitted output...]
```

La salida muestra el nombre del repositorio de imágenes, su etiqueta, el identificador de imagen (ID), la fecha de creación y el tamaño de la imagen. Es muy útil para mantener una vista actualizada de las imágenes disponibles en el almacén local y comprender cuáles están obsoletas.

El comando `podman images` también admite muchas opciones (hay una lista completa disponible ejecutando el comando `man podman-images`). Una de las opciones más interesantes es `--sort`, que se puede utilizar para ordenar imágenes por tamaño, fecha, ID, repositorio o etiqueta. Por ejemplo, podríamos imprimir imágenes ordenadas por fecha de creación para descubrir las más obsoletas, de la siguiente manera:

```bash
# podman images --sort=created
```

Otro par de opciones muy útiles son las opciones `--all` (o `-a`) y `--quiet` (o `-q`). Juntas, se pueden combinar para imprimir solo los IDs de imagen de todas las imágenes almacenadas localmente, incluso las capas de imagen intermedias.

> [!NOTE]
> En el contexto de las imágenes de la Open Container Initiative (OCI), las capas intermedias son las instantáneas individuales de solo lectura del sistema de archivos creadas durante el proceso de compilación.
> Cuando compilas una imagen (usando un Dockerfile o Containerfile), cada instrucción, como `RUN`, `COPY` o `ADD`, crea una nueva capa. Estas capas se apilan unas sobre otras para formar la imagen final.

El comando imprimirá una salida similar al siguiente ejemplo:

```bash
# podman images -qa
ad4c705f24d3
a56f85702a94
b5c5125e3fee
4d7fc5917f3e
625707533167
f881f1aa4d65
96ab2a326180
```

¡Listar y mostrar las imágenes ya descargadas en un sistema no es la parte más interesante del trabajo! Descubramos cómo inspeccionar imágenes con su configuración y contenidos en la siguiente sección.

#### Inspección de configuraciones y contenidos de imágenes (*Inspecting images' configurations and contents*)

Para inspeccionar la configuración de una imagen descargada, el comando `podman image inspect` (o el más corto `podman inspect`) viene a ayudarnos, como se ilustra aquí:

```bash
# podman inspect docker.io/library/nginx:latest
```

La salida impresa será un objeto con formato JSON que contiene la configuración de la imagen, la arquitectura, las capas, las etiquetas (*labels*), las anotaciones y el historial de compilación de la imagen.

El historial de la imagen muestra el historial de creación de cada capa y es muy útil para comprender cómo se construyó la imagen cuando el Dockerfile o el Containerfile no están disponibles.

Dado que la salida es un objeto JSON, podemos extraer campos individuales para recopilar datos específicos o utilizarlos como parámetros de entrada para otros comandos.

El siguiente ejemplo imprime el comando ejecutado cuando se crea un contenedor sobre esta imagen:

```bash
# podman inspect docker.io/library/nginx:latest \
--format "{{ .Config.Cmd }}"
[nginx -g daemon off;]
```

Observa que la salida formateada se gestiona como una plantilla de Go (*Go template*).

A veces, especialmente al solucionar problemas de contenedores, la inspección de una imagen debe ir más allá de una simple verificación de configuración. En ocasiones, necesitamos inspeccionar el contenido del sistema de archivos de una imagen. Para lograr este resultado, Podman ofrece el útil comando `podman image mount`.

El siguiente ejemplo monta la imagen e imprime su ruta de montaje:

```bash
# podman image mount docker.io/library/nginx
/var/lib/containers/storage/overlay/11174639fa910c6d29b6f169964a8388815d85b7908b8825946e276e7242aea5/merged
```

Si ejecutamos un comando `ls` simple en la ruta proporcionada, veremos el sistema de archivos de la imagen, compuesto por sus diversas capas fusionadas, como se muestra a continuación:

```bash
# ls -al /var/lib/containers/storage/overlay/11174639fa910c6d29b6f169964a8388815d85b7908b8825946e276e7242aea5/merged/
total 20
dr-xr-xr-x. 1 root root   38 Oct  9 10:04 .
drwx------. 1 root root   46 Oct  9 10:04 ..
lrwxrwxrwx. 1 root root    7 Sep 26 00:00 bin -> usr/bin
drwxr-xr-x. 1 root root    0 Aug 14 16:10 boot
drwxr-xr-x. 1 root root    0 Sep 26 00:00 dev
drwxr-xr-x. 1 root root   54 Oct  3 00:58 docker-entrypoint.d
-rwxr-xr-x. 1 root root 1620 Oct  3 00:57 docker-entrypoint.sh
drwxr-xr-x. 1 root root  366 Oct  3 00:58 etc
drwxr-xr-x. 1 root root    0 Aug 14 16:10 home
lrwxrwxrwx. 1 root root    7 Sep 26 00:00 lib -> usr/lib
lrwxrwxrwx. 1 root root    9 Sep 26 00:00 lib64 -> usr/lib64
drwxr-xr-x. 1 root root    0 Sep 26 00:00 media
drwxr-xr-x. 1 root root    0 Sep 26 00:00 mnt
drwxr-xr-x. 1 root root    0 Sep 26 00:00 opt
drwxr-xr-x. 1 root root    0 Aug 14 16:10 proc
drwx------. 1 root root   30 Sep 26 00:00 root
drwxr-xr-x. 1 root root    8 Sep 26 00:00 run
lrwxrwxrwx. 1 root root    8 Sep 26 00:00 sbin -> usr/sbin
drwxr-xr-x. 1 root root    0 Sep 26 00:00 srv
drwxr-xr-x. 1 root root    0 Aug 14 16:10 sys
drwxrwxrwt. 1 root root    0 Sep 26 00:00 tmp
drwxr-xr-x. 1 root root   40 Sep 26 00:00 usr
drwxr-xr-x. 1 root root   22 Sep 26 00:00 var
```

Para desmontar la imagen, simplemente ejecuta el comando `podman image unmount`, de la siguiente manera:

```bash
# podman image unmount docker.io/library/nginx
```

Montar imágenes en modo rootless es un poco diferente ya que este modo de ejecución solo admite el montaje manual del controlador de almacenamiento del sistema de archivos virtual (VFS). Dado que estamos trabajando con un controlador de almacenamiento OverlayFS predeterminado, los comandos mount/unmount no funcionarían directamente. El requisito es ejecutar primero el comando `podman unshare`. Este ejecuta un nuevo proceso de shell dentro de un nuevo namespace donde el ID de usuario (UID) y el ID único global (GID) actuales se asignan a UID 0 y GID 0, respectivamente. A partir de ahora, tenemos privilegios elevados para ejecutar el comando `podman mount`. Veamos un ejemplo aquí:

```bash
$ podman unshare
# podman image mount docker.io/library/nginx:latest
.local/share/containers/storage/overlay/11174639fa910c6d29b6f169964a8388815d85b7908b8825946e276e7242aea5/merged
```

Observa que el punto de montaje se encuentra ahora en el directorio de inicio de `<username>`.

Para desmontar, simplemente ejecuta el comando `podman unmount`, de la siguiente manera:

```bash
# podman image unmount docker.io/library/nginx:latest
7f553e8bbc897571642d836b31eaf6ecbe395d7641c2b24291356ed28f3f2bd0
# exit
```

El comando `exit` es necesario para salir del namespace temporal no compartido (*unshared*).

#### Eliminación de imágenes (*Deleting images*)

Para eliminar una imagen del almacén local, podemos usar el comando `podman rmi`. El siguiente ejemplo elimina la imagen de nginx descargada previamente:

```bash
# podman rmi docker.io/library/nginx:latest
Untagged: docker.io/library/nginx:latest
Deleted: 7f553e8bbc897571642d836b31eaf6ecbe395d7641c2b24291356ed28f3f2bd0
```

El mismo comando funciona en modo rootless cuando lo ejecuta un usuario estándar sobre el almacén local de su directorio home.

Para eliminar todas las imágenes almacenadas en caché, utiliza el siguiente ejemplo, que se basa en la expansión de comandos de shell para obtener una lista completa de IDs de imágenes:

```bash
# podman rmi $(podman images -qa)
```

Observa el símbolo de almohadilla (`#`) al principio de la línea, que nos indica que el comando se ejecuta como root.

El siguiente comando elimina todas las imágenes de la caché local de un usuario regular (observa el símbolo de dólar al principio de la línea):

```bash
$ podman rmi $(podman images -qa)
```

> [!IMPORTANT]
> El comando `podman rmi` falla al intentar eliminar imágenes que están actualmente en uso por un contenedor. Primero, detén y elimina los contenedores que usan las imágenes bloqueadas y luego ejecuta el comando nuevamente. Alternativamente, podemos usar la opción `--force`, que eliminará las imágenes que están en uso, pero también eliminará todos los contenedores que estén usando la imagen.

Podman también ofrece una forma más sencilla de limpiar imágenes colgantes o no utilizadas: el comando `podman image prune`. Elimina todas las imágenes que no están en uso, dejando solo aquellas asociadas con contenedores, en ejecución o detenidos.

El siguiente ejemplo elimina todas las imágenes no utilizadas sin solicitar confirmación:

```bash
$ sudo podman image prune -af
```

El mismo comando se aplica en modo rootless, eliminando únicamente las imágenes del almacén local del usuario en su home. Con esto, hemos aprendido a administrar imágenes de contenedores en nuestra máquina. Ahora aprendamos a manejar y verificar contenedores en ejecución.

---

### Operaciones con contenedores en ejecución

En el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*, aprendimos en la sección *Ejecución de tu primer contenedor* cómo ejecutar un contenedor con ejemplos básicos, que involucraban la ejecución de un proceso Bash dentro de un contenedor Fedora y un servidor httpd, lo que también fue útil para aprender a exponer contenedores externamente.

Ahora exploraremos un conjunto de comandos utilizados para monitorear y verificar nuestros contenedores en ejecución y obtener información sobre su comportamiento.

#### Visualización y gestión del estado de los contenedores

Comencemos ejecutando un contenedor simple y exponiéndolo en el puerto 8080 para que sea accesible externamente, de la siguiente manera:

```bash
$ podman run -d -p 8080:80 docker.io/library/nginx
```

El ejemplo anterior se ejecuta en modo rootless, pero se puede aplicar lo mismo como usuario root anteponiendo `sudo` al comando. En este caso, simplemente no era necesario que un contenedor se ejecutara de esa manera.

> [!NOTE]
> Los contenedores sin root proporcionan una ventaja de seguridad adicional. Si un proceso malicioso rompe el aislamiento del contenedor, tal vez aprovechando una vulnerabilidad en el host, en el mejor de los casos obtendrá los privilegios del usuario que inició el contenedor sin root.

Ahora que nuestro contenedor está en funcionamiento y listo para servir, podemos probarlo ejecutando un comando `curl` en localhost, lo que debería producir una salida HTML predeterminada como esta:

```bash
$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
```

Obviamente, un servidor nginx vacío sin contenido que servir es inútil, pero aprenderemos a servir contenido personalizado mediante volúmenes o creando imágenes personalizadas más adelante, en los siguientes capítulos.

El primer comando que podemos usar para verificar nuestro contenedor es `podman ps`. Este simplemente imprime información útil de los contenedores en ejecución, con la opción de personalizar y ordenar la salida. Ejecutemos el comando en nuestro host y veamos qué se imprime, de la siguiente manera:

```bash
$ podman ps
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS                                 NAMES
d19dbb81a0f3  docker.io/library/nginx:latest  nginx -g daemon o...  34 seconds ago  Up 34 seconds  0.0.0.0:8080->80/tcp, 80/tcp          admiring_bouman
```

La salida produce información interesante sobre los contenedores en ejecución, como se detalla aquí:

- **CONTAINER ID**: Cada contenedor nuevo obtiene un ID hexadecimal único. El ID completo tiene una longitud de 64 caracteres y en la salida de `podman ps` se imprime una porción abreviada de 12 caracteres.
- **IMAGE**: La imagen utilizada por el contenedor.
- **COMMAND**: El comando ejecutado dentro del contenedor.
- **CREATED**: La fecha de creación del contenedor.
- **STATUS**: El estado actual del contenedor.
- **PORTS**: Los puertos de red abiertos en el contenedor. Cuando se aplica un mapeo de puertos, podemos ver uno o más pares `ip:puerto` de host asignados a los puertos del contenedor con un signo de flecha. Por ejemplo, la cadena `0.0.0.0:8080->80/tcp` significa que el puerto de host `8080/tcp` está expuesto en todas las interfaces de escucha y está asignado al puerto de contenedor `80/tcp`.
- **NAMES**: El nombre del contenedor. Este puede ser asignado por el usuario o generado aleatoriamente por el motor de contenedores.

> [!TIP]
> Observa el nombre generado aleatoriamente en la última columna de la salida. Podman continúa la tradición de Docker de generar nombres aleatorios utilizando adjetivos en la parte izquierda del nombre y científicos y hackers notables en la parte derecha. De hecho, Podman todavía usa el mismo paquete Docker `github.com/docker/docker/namesgenerator`, incluido en el directorio `vendor` del proyecto.

Para obtener una lista completa de contenedores tanto en ejecución como detenidos, podemos agregar la opción `-a` al comando. Para demostrar esto, primero presentamos el comando `podman stop`. Esto cambia el estado del contenedor a detenido y envía una señal SIGTERM a los procesos que se ejecutan dentro del contenedor. Si el contenedor no responde, envía una señal SIGKILL después de un tiempo de espera determinado de 10 segundos.

Intentemos detener el contenedor anterior y verificar su estado ejecutando el siguiente código:

```bash
$ podman stop d19dbb81a0f3
$ podman ps
```

Esta vez, `podman ps` produjo una salida vacía. Esto se debe a que el estado del contenedor es detenido. Para obtener una lista completa de contenedores en ejecución y detenidos, ejecuta el siguiente comando:

```bash
$ podman ps -a
CONTAINER ID  IMAGE                           COMMAND               CREATED          STATUS                      PORTS                         NAMES
d19dbb81a0f3  docker.io/library/nginx:latest  nginx -g daemon o...  About a minute ago  Exited (0) 20 seconds ago   0.0.0.0:8080->80/tcp, 80/tcp  admiring_bouman
```

Observa el estado del contenedor, que indica que el contenedor ha salido con un código de salida 0.

El contenedor detenido se puede reanudar ejecutando el comando `podman start`, de la siguiente manera:

```bash
$ podman start d19dbb81a0f3
```

Este comando simplemente vuelve a iniciar el contenedor que detuvimos anteriormente.

Si ahora verificamos el estado del contenedor nuevamente, veremos que está en funcionamiento, como se indica aquí:

```bash
$ podman ps
CONTAINER ID  IMAGE                           COMMAND               CREATED          STATUS         PORTS                         NAMES
d19dbb81a0f3  docker.io/library/nginx:latest  nginx -g daemon o...  About a minute ago  Up 13 seconds  0.0.0.0:8080->80/tcp, 80/tcp  admiring_bouman
```

Podman conserva la configuración, el almacenamiento y los metadatos del contenedor mientras esté en estado detenido. De todos modos, cuando reanudamos el contenedor, iniciamos un nuevo proceso dentro de él.

Para ver más opciones, consulta la página de manual correspondiente (`man podman-start`).

Si simplemente necesitamos reiniciar un contenedor en ejecución, podemos usar el comando `podman restart`, de la siguiente manera:

```bash
$ podman restart <Container_ID_or_Name>
```

Este comando tiene el efecto de reiniciar inmediatamente el contenedor, utilizando los mismos datos y proceso que el anterior. Si bien reutilizamos el sistema de archivos y la configuración, todos los namespaces serán diferentes, lo que puede solucionar muchos problemas de red, ya que todas las configuraciones se realizarán nuevamente. El comando `podman start` también se puede utilizar para iniciar contenedores que se hayan creado previamente pero que no se hayan ejecutado. Para crear un contenedor sin iniciarlo, utiliza el comando `podman create`. El siguiente ejemplo crea un contenedor pero no lo inicia:

```bash
$ podman create -p 8080:80 docker.io/library/nginx
```

Para iniciarlo, ejecuta `podman start` sobre el ID o nombre del contenedor creado, de la siguiente manera:

```bash
$ podman start <Container_ID_or_Name>
```

Este comando es muy útil para preparar un entorno sin ejecutarlo o para montar un sistema de archivos de contenedor para un usuario normal (no root), como en el siguiente ejemplo:

```bash
$ podman unshare
$ podman container mount <Container_ID_or_Name>
/home/<username>/.local/share/containers/storage/overlay/bf9d8df299436d80dece200a23e1b8b957f987a254a656ef94cdc56669823b5c/merged
```

> [!NOTE]
> El comando `podman container mount` (o el más corto `podman mount`) se utiliza para montar el sistema de archivos raíz de un contenedor en una ubicación de tu máquina host. Esto te permite explorar, leer y modificar los archivos dentro de un contenedor directamente desde la terminal o GUI del host, sin necesidad de ejecutar una shell dentro del contenedor en sí.

Presentemos ahora un comando de uso muy frecuente: `podman rm`. Como su nombre indica, se utiliza para eliminar contenedores del host. De forma predeterminada, elimina contenedores detenidos, pero se puede forzar para eliminar contenedores en ejecución con la opción `-f`.

Usando el contenedor del ejemplo anterior, si lo detenemos nuevamente y ejecutamos el comando `podman rm`, como se ilustra en los siguientes comandos, se descartarán todo el almacenamiento, las configuraciones y los metadatos del contenedor:

```bash
$ podman stop d19dbb81a0f3
$ podman rm d19dbb81a0f3
```

Si ahora ejecutamos un comando `podman ps` nuevamente, incluso con la opción `-a`, obtendremos una lista vacía, como se ilustra aquí:

```bash
$ podman ps -a
CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
```

Para obtener más detalles, inspecciona la página de manual del comando (`man podman-rm`).

A veces, resulta útil (al igual que con las imágenes) imprimir únicamente el ID del contenedor con la opción `-q`. Y cuando se combina con la opción `-a`, puede imprimir una lista de todos los contenedores detenidos y en ejecución en el host. Probemos otro ejemplo aquí:

```bash
$ for i in {1..5}; do podman run -d docker.io/library/nginx; done
```

Es interesante observar que hemos utilizado un bucle de shell para iniciar cinco contenedores idénticos, esta vez sin ningún mapeo de puertos: simplemente contenedores nginx planos. Podemos inspeccionar sus IDs con el siguiente comando:

```bash
$ podman ps -qa
b38ebfed5921
6204efc6d6b2
762967d87657
269f1affb699
1161072ec559
```

¿Cómo podemos detener y eliminar todos nuestros contenedores en ejecución rápidamente? Podemos utilizar la expansión de shell para combinarlo con otros comandos y alcanzar el resultado deseado. La expansión de shell es una herramienta poderosa que ejecuta el comando dentro de paréntesis redondos y nos permite pasar la cadena de salida como argumentos al comando externo, como se ilustra en los siguientes comandos:

```bash
$ podman stop $(podman ps -qa)
$ podman rm $(podman ps -qa)
```

Los dos comandos detuvieron todos los contenedores en ejecución, identificados por sus IDs, y los eliminaron del host.

Alternativamente, podemos usar un alias para todos los contenedores, `-a`, que admiten la mayoría de los comandos, como `podman start`, `podman stop` y `podman rm`.

El comando `podman ps` permite a los usuarios refinar su salida aplicando filtros específicos. Una lista completa de todos los filtros aplicables está disponible en la página de manual `podman-ps`. Una aplicación simple pero útil es el filtro `status`, que permite a los usuarios imprimir solo contenedores en una condición específica. Los estados posibles son `created`, `exited`, `paused`, `running` y `unknown`.

El siguiente ejemplo solo imprime contenedores en estado `exited`:

```bash
$ podman ps --filter status=exited
```

Nuevamente, podemos aprovechar el poder de la expansión de shell para eliminar solo los contenedores que han salido (*exited*), de la siguiente manera:

```bash
$ podman rm $(podman ps -qa --filter status=exited)
```

Se puede lograr un resultado similar con el comando más fácil de recordar `podman container prune` que se muestra aquí, que elimina (*prunes*) todos los contenedores detenidos del host:

```bash
$ podman container prune
```

La ordenación es otra opción útil para producir una salida ordenada al listar contenedores. El siguiente ejemplo muestra cómo ordenar por ID de contenedor:

```bash
$ podman ps -q --sort id
```

El comando `podman ps` admite el formateo mediante una plantilla de Go para producir una salida personalizada. El siguiente ejemplo imprime solo los IDs de contenedor y los comandos ejecutados dentro de ellos:

```bash
$ podman ps -a --format "{{.ID}} {{.Command}}" --no-trunc
```

Además, observa que se agrega la opción `--no-trunc` para evitar truncar la salida del comando. Esto no es obligatorio, pero es útil cuando tenemos comandos largos ejecutados dentro de los contenedores.

Si simplemente deseamos extraer el PID de host del proceso que se ejecuta dentro de los contenedores en ejecución, podemos ejecutar el siguiente ejemplo:

```bash
$ podman ps --format "{{ .Pid }}"
```

En cambio, si necesitamos obtener información sobre los namespaces aislados, `podman ps` puede imprimir detalles sobre los namespaces clonados de los contenedores en ejecución. Este es un punto de partida útil para la resolución de problemas y la inspección avanzada. Puedes ver el comando ejecutándose aquí:

```bash
$ podman ps --namespace
CONTAINER ID  NAMES                  PID     CGROUPNS    IPC         MNT         NET         PIDNS       USERNS      UTS
f2666ed4a46a  unruffled_hofstadter   437764  4026533088  4026533086  4026533083  4026532948  4026533087  4026532973  4026533085
```

Esta subsección cubrió muchas operaciones comunes para controlar y ver el estado de los contenedores. En la siguiente sección, aprenderemos a pausar y reanudar contenedores en ejecución.

#### Pausar y reanudar contenedores (*Pausing and unpausing containers*)

Esta breve sección cubre los comandos `podman pause` y `podman unpause`. A pesar de ser una sección relacionada con el manejo del estado del contenedor, es interesante comprender cómo Podman y el runtime de contenedores aprovechan los cgroups para lograr propósitos específicos.

En pocas palabras, los comandos `pause` y `unpause` tienen el propósito de pausar y reanudar los procesos de un contenedor en ejecución. Ahora, comprendamos la diferencia entre los comandos `pause` y `stop` en Podman.

Mientras que el comando `podman stop` simplemente envía una señal SIGTERM/SIGKILL al proceso principal en el contenedor, el comando `podman pause` usa cgroups para pausar el proceso sin terminarlo. Cuando el contenedor se reanuda (*unpaused*), el mismo proceso se reanuda de manera transparente.

> [!TIP]
> La lógica de bajo nivel de pause/unpause se implementa en el runtime de contenedores; para los más curiosos, esta era la implementación en `crun` al momento de escribir este texto:
> [https://github.com/containers/crun/blob/7ef74c9330033cb884507c28fd8c267861486633/src/libcrun/cgroup.c#L1894-L1936](https://github.com/containers/crun/blob/7ef74c9330033cb884507c28fd8c267861486633/src/libcrun/cgroup.c#L1894-L1936)

El siguiente ejemplo demuestra los comandos `podman pause` y `podman unpause`. Primero, iniciemos un contenedor Fedora que imprima una cadena de fecha y hora cada 2 segundos en un bucle sin fin, de la siguiente manera:

```bash
$ podman run --name timer docker.io/library/fedora bash -c "while true; do echo $(date); sleep 2; done"
```

Dejamos intencionalmente el contenedor ejecutándose en una ventana y abrimos una nueva ventana/pestaña para administrar su estado. Antes de emitir el comando pause, inspeccionemos el PID ejecutando el siguiente comando:

```bash
$ podman ps --format "{{ .Pid }}" --filter name=timer
816807
```

Ahora, pausemos el contenedor en ejecución con el siguiente comando:

```bash
$ podman pause timer
```

Si volvemos al contenedor timer, vemos que la salida simplemente se pausó, pero el contenedor no ha salido. La acción unpause que se ve aquí lo devolverá a la vida:

```bash
$ podman unpause timer
```

Después de la acción unpause, el contenedor timer comenzará a imprimir salidas de fecha nuevamente. Al observar el PID aquí, nada ha cambiado, como se esperaba:

```bash
$ podman ps --format "{{ .Pid }}" --filter name=timer
816807
```

Podemos verificar el estado de los cgroups del contenedor pausado/reanudado. En una tercera pestaña, abre una terminal con una shell de root y accede a la jerarquía del controlador cgroupfs después de reemplazar el ID de contenedor correcto, de la siguiente manera:

```bash
$ sudo -i
$ cd /sys/fs/cgroup/user.slice/user-1000.slice/user@$UID.service/user.slice/libpod-<CONTAINER_ID>.scope/container
```

Ahora, observa el contenido del archivo `cgroup.freeze`. Este archivo contiene un valor booleano y su estado cambia a medida que pausamos/reanudamos el contenedor de 0 a 1 y viceversa. Intenta pausar y reanudar el contenedor nuevamente para probar los cambios.

> [!TIP]
> Dado que el bucle echo se emitió con un comando `bash -c`, debemos enviar una señal SIGKILL al proceso. Para hacer esto, podemos detener el contenedor y esperar el tiempo de espera de 10 segundos, o simplemente ejecutar un comando `podman kill`, de la siguiente manera:
> ```bash
> $ podman kill timer
> ```
> Alternativamente, incluso podemos omitir el tiempo de espera de 10 segundos con la opción `--time 0`.

En esta subsección, cubrimos en detalle los comandos más comunes para observar y modificar el estado de un contenedor. Ahora podemos pasar a inspeccionar los procesos que se ejecutan dentro de los contenedores en ejecución.

#### Inspección de procesos dentro de los contenedores (*Inspecting processes inside containers*)

Cuando un contenedor se está ejecutando, los procesos dentro de él están aislados a nivel de namespace, pero los usuarios aún tienen control total de los procesos en ejecución y pueden inspeccionar su comportamiento. Hay muchos niveles de complejidad en la inspección de procesos, pero Podman ofrece herramientas que pueden acelerar esta tarea.

Comencemos con el comando `podman top`: este proporciona una vista completa de los procesos que se ejecutan dentro de un contenedor. El siguiente ejemplo muestra los procesos que se ejecutan dentro de un contenedor nginx:

```bash
$ podman top f2666ed4a46a
USER   PID   PPID   %CPU    ELAPSED             TTY   TIME   COMMAND
root   1     0      0.000   3m26.540290427s     ?     0s     nginx: master process nginx -g daemon off;
nginx  26    1      0.000   3m26.540547429s     ?     0s     nginx: worker process
nginx  27    1      0.000   3m26.540788803s     ?     0s     nginx: worker process
nginx  28    1      0.000   3m26.540914386s     ?     0s     nginx: worker process
nginx  29    1      0.000   3m26.541040023s     ?     0s     nginx: worker process
nginx  30    1      0.000   3m26.541161213s     ?     0s     nginx: worker process
nginx  31    1      0.000   3m26.541297546s     ?     0s     nginx: worker process
nginx  32    1      0.000   3m26.54141773s      ?     0s     nginx: worker process
nginx  33    1      0.000   3m26.541564289s     ?     0s     nginx: worker process
nginx  34    1      0.000   3m26.541685475s     ?     0s     nginx: worker process
nginx  35    1      0.000   3m26.541808977s     ?     0s     nginx: worker process
nginx  36    1      0.000   3m26.541932099s     ?     0s     nginx: worker process
nginx  37    1      0.000   3m26.54205111s      ?     0s     nginx: worker process
```

El resultado es muy similar a la salida del comando `ps` en lugar de la interactiva producida por el comando `top` de Linux.

Es posible aplicar un formato personalizado a la salida. El siguiente ejemplo solo imprime PIDs, comandos y argumentos:

```bash
$ podman top f2666ed4a46a pid comm args
PID   COMMAND   COMMAND
1     nginx     nginx: master process nginx -g daemon off;
26    nginx     nginx: worker process
27    nginx     nginx: worker process
28    nginx     nginx: worker process
29    nginx     nginx: worker process
30    nginx     nginx: worker process
31    nginx     nginx: worker process
32    nginx     nginx: worker process
33    nginx     nginx: worker process
34    nginx     nginx: worker process
35    nginx     nginx: worker process
36    nginx     nginx: worker process
37    nginx     nginx: worker process
```

Es posible que necesitemos inspeccionar los procesos del contenedor con mayor detalle. Como comentamos anteriormente, en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*, una vez que se inicia un contenedor nuevo, comenzará a asignar PIDs desde el número 0, mientras que bajo el capó, el motor de contenedores asignará los PIDs de este contenedor con los reales en el host. Por lo tanto, podemos usar la salida del comando `podman ps --namespace` para extraer el PID original del proceso en el host para un contenedor determinado. Con esa información, podemos realizar análisis avanzados. El siguiente ejemplo muestra cómo adjuntar el comando `strace`, utilizado para inspeccionar las llamadas al sistema (*syscalls*) de los procesos, al proceso que se ejecuta dentro del contenedor:

```bash
$ sudo strace -p <PID>
```

Los detalles sobre el uso del comando `strace` quedan fuera del alcance de este libro. Consulta `man strace` para ver ejemplos más avanzados y una explicación más profunda de las opciones del comando.

Alternativamente, la forma más fácil de obtener el PID es usar el comando `podman top` como se describe en el siguiente bloque de código:

```bash
$ podman top $CONTAINER_ID pid hpid
```

La opción `hpid` imprimirá el PID del proceso en el host.

Otro comando útil que se puede aplicar fácilmente a los procesos que se ejecutan dentro de un contenedor es `pidstat`. Una vez obtenido el PID, podemos inspeccionar el uso de recursos de esta manera:

```bash
$ pidstat -p <PID> [<interval> <count>]
```

Los números enteros aplicados al final representan, respectivamente, el intervalo de ejecución del comando y el número de veces que debe imprimir las estadísticas de uso. Consulta `man pidstat` para obtener más opciones de uso.

Cuando un proceso en un contenedor deja de responder, es posible gestionar su terminación abrupta con el comando `podman kill`. De forma predeterminada, envía una señal SIGKILL al proceso dentro del contenedor. El siguiente ejemplo crea un contenedor httpd y luego lo mata (*kills*):

```bash
$ podman run --name custom-webserver -d docker.io/library/httpd
$ podman kill custom-webserver
```

Opcionalmente podemos enviar señales personalizadas (como SIGTERM o SIGHUP) con la opción `--signal`. Ten en cuenta que un contenedor terminado con kill no se elimina del host, sino que continúa existiendo, está detenido y se encuentra en estado de salida (*exited*).

En el [Capítulo 11](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/11), *Resolución de Problemas y Monitoreo de Contenedores*, volveremos a ocuparnos de la resolución de problemas de contenedores y aprenderemos a usar herramientas avanzadas como `nsenter` para inspeccionar procesos de contenedores. Ahora pasaremos a los comandos básicos de estadísticas de contenedores que pueden resultar útiles para supervisar el uso general de recursos por parte de todos los contenedores que se ejecutan en un sistema.

#### Monitoreo de estadísticas del contenedor (*Monitoring container stats*)

Cuando se ejecutan varios contenedores en el mismo host, es crucial monitorear la cantidad de recursos de CPU, memoria, disco y red que consumen en un intervalo de tiempo determinado. El primer comando, más simple, que un administrador puede usar es el comando `podman stats`, que se muestra aquí:

```bash
$ podman stats
```

Sin ninguna opción, el comando abrirá una ventana de actualización automática tipo `top` con las estadísticas de todos los contenedores en ejecución. Los valores impresos por defecto se enumeran aquí:

- **ID**: El ID del contenedor en ejecución.
- **NAME**: El nombre del contenedor en ejecución.
- **CPU %**: El uso total de CPU como porcentaje.
- **MEM USAGE / LIMIT**: Uso de memoria frente a un límite dado (dictado por las capacidades del sistema o por límites impuestos por cgroups).
- **MEM %**: El uso total de memoria como porcentaje.
- **NET IO**: Operaciones de entrada/salida (I/O) de red.
- **BLOCK IO**: Operaciones de I/O de disco.
- **PIDS**: El número de PIDs dentro del contenedor.
- **CPU TIME**: Tiempo total de CPU consumido.
- **AVG CPU %**: Uso promedio de CPU como porcentaje.

En caso de que se necesite una redirección, es posible evitar la transmisión de una salida que se actualiza automáticamente con la opción `--no-stream`, de la siguiente manera:

```bash
$ podman stats --no-stream
```

De todos modos, tener una salida estática de este tipo no es muy útil para su análisis o ingestión. Un mejor enfoque es aplicar un formateador de plantillas Go o JSON. El siguiente ejemplo imprime estadísticas en formato JSON:

```bash
$ podman stats --format=json
[
  {
    "id": "d19dbb81a0f3",
    "name": "admiring_bouman",
    "cpu_time": "16.773ms",
    "cpu_percent": "0.01%",
    "avg_cpu": "0.01%",
    "mem_usage": "1.884MB / 475.4MB",
    "mem_percent": "0.40%",
    "net_io": "0B / 796B",
    "block_io": "0B / 0B",
    "pids": "2"
  }
]
```

De manera similar, es posible personalizar los campos de salida utilizando una plantilla de Go.

> [!NOTE]
> La opción de línea de comandos para Podman `--format=json` funciona en la mayoría de los comandos. Por ejemplo, `podman ps --format=json` produce una salida legible por máquina.

El siguiente ejemplo solo imprime el ID del contenedor, el porcentaje de uso de CPU, el uso total de memoria en bytes y los PIDs:

```bash
$ podman stats -a --no-stream --format "{{ .ID }} {{ .CPUPerc }} {{ .MemUsageBytes }} {{ .PIDs }}"
```

En esta sección, hemos aprendido a monitorear contenedores en ejecución y sus procesos aislados. La siguiente sección muestra cómo inspeccionar las configuraciones de los contenedores para su análisis y resolución de problemas.

---

### Inspección de la información de los contenedores

Un contenedor en ejecución expone un conjunto de datos de configuración y metadatos listos para ser consumidos. Podman implementa el comando `podman inspect` para imprimir todas las configuraciones del contenedor y la información en tiempo de ejecución. En su forma más simple, simplemente podemos pasar el ID o nombre del contenedor, de esta forma:

```bash
$ podman inspect <Container_ID_or_Name>
```

Este comando imprime una salida JSON con todas las configuraciones del contenedor. Por motivos de espacio, enumeraremos algunos de los campos más notables aquí:

- **Path**: La ruta del punto de entrada (*entrypoint*) del contenedor. Profundizaremos en los entrypoints más adelante, cuando analicemos los Dockerfiles.
- **Args**: Los argumentos pasados al entrypoint que, junto con Path, forman el comando ejecutado en el contenedor.
- **State**: El estado actual del contenedor, incluida información crucial como el PID ejecutado, el PID de conmon, la versión de OCI y el estado de la comprobación de salud (*health check*).
- **Image**: El ID de la imagen utilizada para ejecutar el contenedor.
- **Name**: El nombre del contenedor.
- **MountLabel**: La etiqueta de montaje del contenedor para SELinux.
- **ProcessLabel**: La etiqueta de proceso del contenedor para SELinux.
- **EffectiveCaps**: Capacidades efectivas (*capabilities*) aplicadas al contenedor.
- **GraphDriver**: El tipo de controlador de almacenamiento (el valor predeterminado es `overlayfs`) y una lista de directorios overlay `upper`, `lower` y `merged`.
- **Mounts**: Los montajes reales especificados por el usuario en el contenedor.
- **NetworkSettings**: La configuración general de la red del contenedor, incluida su dirección IP interna, puertos expuestos y mapeos de puertos.
- **Config**: La configuración de tiempo de ejecución del contenedor, incluidas variables de entorno, nombre de host, comando, directorio de trabajo, etiquetas y anotaciones.
- **HostConfig**: La configuración del host, incluidas las cuotas de cgroups, el modo de red y las capacidades.

Esta es una gran cantidad de información que, la mayoría de las veces, es demasiada para nuestras necesidades. Cuando necesitamos extraer campos específicos, podemos usar la opción `--format` para imprimir solo los seleccionados. El siguiente ejemplo imprime solo el PID vinculado al host del proceso ejecutado dentro del contenedor:

```bash
$ podman inspect <ID or Name> --format "{{ .State.Pid }}"
```

El resultado está en formato de plantilla de Go. Esto permite flexibilidad para personalizar la cadena de salida como deseemos.

El comando `podman inspect` también es útil para comprender el comportamiento del motor de contenedores y para obtener información útil durante las tareas de resolución de problemas.

Por ejemplo, cuando se inicia un contenedor, aprendemos que el archivo `resolv.conf` se monta dentro del contenedor desde una ruta que se define en la clave `{{ .ResolvConfPath }}`. La ruta de destino es `/run/user/<UID>/containers/overlay-containers/<Container_ID>/userdata/resolv.conf` cuando el contenedor se ejecuta en modo rootless y `/var/run/containers/storage/overlay-containers/<Container_ID>/userdata/resolv.conf` cuando se encuentra en modo con privilegios de root (*rootful*).

Otra información interesante es la lista de todas las capas fusionadas administradas por `overlayfs`. Intentemos ejecutar un nuevo contenedor, esta vez en modo rootful, y busquemos información sobre las capas fusionadas, de la siguiente manera:

```bash
# podman run --name logger -d docker.io/library/fedora bash -c "while true; do echo test >> /tmp/test.log; sleep 5; done"
```

Este contenedor ejecuta un bucle simple que escribe una cadena en un archivo de texto cada 5 segundos. Ahora, ejecutemos un comando `podman inspect` para obtener información sobre `MergedDir`, que es el directorio donde `overlayfs` fusiona todas las capas. El comando relacionado se ilustra en el siguiente fragmento:

```bash
# podman inspect logger --format "{{ .GraphDriver.Data.MergedDir }}"
/var/lib/containers/storage/overlay/4372e7c85d552e84e470223cd5aef0bc0fc69ffdf83f687506dff9bd93b64767/merged
```

Dentro de este directorio, podemos encontrar el archivo `/tmp/test.log`, como se indica aquí:

```bash
# cat /var/lib/containers/storage/overlay/4372e7c85d552e84e470223cd5aef0bc0fc69ffdf83f687506dff9bd93b64767/merged/tmp/test.log
test
test
test
test
test
[...]
```

Podemos profundizar más: el directorio `LowerDir` contiene una lista de las capas de imagen base, como se muestra en el siguiente fragmento de código:

```bash
# podman inspect logger \
--format "{{ .GraphDriver.Data.LowerDir}}"
/var/lib/containers/storage/overlay/b0d5c42c12e7b1e896892f1a013ac57b467f2f25545833cf4c5ebc2a5f823845/diff
```

En este ejemplo, la imagen base se compone de una sola capa. ¿Vamos a encontrar el archivo de log aquí? Echemos un vistazo:

```bash
# cat /var/lib/containers/storage/overlay/b0d5c42c12e7b1e896892f1a013ac57b467f2f25545833cf4c5ebc2a5f823845/diff/tmp/test.log
cat: /var/lib/containers/storage/overlay/b0d5c42c12e7b1e896892f1a013ac57b467f2f25545833cf4c5ebc2a5f823845/diff/tmp/test.log: No such file or directory
```

Nos falta el archivo de registro en esta capa. Esto se debe a que en el directorio `LowerDir` no se escribe y representa las capas de imagen de solo lectura. Se fusiona con un directorio `UpperDir` que es la capa de lectura y escritura del contenedor. Con `podman inspect`, podemos averiguar dónde reside, como se ilustra aquí:

```bash
# podman inspect logger --format "{{ .GraphDriver.Data.UpperDir }}"
/var/lib/containers/storage/overlay/27d89046485db7c775b108a80072eafdf9aa63d14ee1205946d74623fc195314/diff
```

El directorio de salida contendrá solo un grupo de archivos y directorios, escritos desde el inicio del contenedor, incluido el archivo `/tmp/test.log`, como se ilustra en el siguiente fragmento de código:

```bash
# cat /var/lib/containers/storage/overlay/4372e7c85d552e84e470223cd5aef0bc0fc69ffdf83f687506dff9bd93b64767/diff/tmp/test.log
test
test
test
test
[...]
```

Ahora podemos detener y eliminar el contenedor logger ejecutando el siguiente comando:

```bash
# podman stop logger && podman rm logger
```

Este ejemplo fue un anticipo del tema de almacenamiento de contenedores que se cubrirá en el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*. Los mecanismos de `overlayfs`, con los conceptos de directorio lower, upper y merged, se analizarán con más detalle.

En esta sección, aprendimos a inspeccionar contenedores en ejecución y a recopilar información y configuraciones en tiempo de ejecución. La siguiente sección cubrirá las mejores prácticas para capturar registros de los contenedores.

---

### Captura de logs de los contenedores

Como se describió anteriormente en este capítulo, los contenedores están formados por uno o más procesos que pueden fallar, imprimiendo errores y describiendo su estado actual en un archivo de registro (*log*). Pero, ¿dónde se almacenan estos registros?

Bueno, por supuesto, un proceso en un contenedor podría escribir sus mensajes de registro dentro de un archivo en algún lugar de un sistema de archivos temporal que el motor de contenedores haya puesto a su disposición (si lo hubiera). Pero ¿qué pasa con un sistema de archivos de solo lectura o cualquier restricción de permisos en el contenedor en ejecución?

Una mejor práctica de los contenedores para exponer registros relevantes fuera del escudo del contenedor en realidad aprovecha el uso de flujos estándar (*standard streams*): salida estándar (STDOUT) y error estándar (STDERR).

> [!NOTE]
> Los flujos estándar son canales de comunicación interconectados a un proceso en ejecución en un sistema operativo. Cuando se ejecuta un programa a través de una shell interactiva, estos flujos se conectan directamente a la terminal en ejecución del usuario para permitir que la entrada, la salida y los errores fluyan entre la terminal y el proceso, y viceversa.

Dependiendo de las opciones que usemos para ejecutar un contenedor completamente nuevo, Podman actuará apropiadamente adjuntando los flujos estándar STDIN, STDOUT y STDERR a un archivo local para almacenar los registros. Para las distribuciones de Linux que usan systemd como administrador de inicio (*init manager*), los registros se almacenan en `journald` de forma predeterminada.

En el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, vimos cómo ejecutar un contenedor en segundo plano, desacoplándonos de un contenedor en ejecución. Usamos la opción `-d` para iniciar un contenedor en modo desacoplado a través del comando `podman run`, como se ilustra aquí:

```bash
$ podman run -d -i -t docker.io/library/httpd
```

Con el comando anterior, le indicamos a Podman que inicie un contenedor en modo desacoplado (`-d`), con un pseudoterminal conectado al flujo STDIN (`-t`), manteniendo abierto el flujo de entrada estándar incluso si aún no hay ningún terminal conectado (`-i`).

El comportamiento estándar de Podman era conectarse a los flujos STDOUT y STDERR y almacenar cualquier dato publicado por el contenedor en un archivo de registro en el sistema de archivos del host.

En la distribución Fedora Linux, a partir de la versión 35, los mantenedores decidieron cambiar de `k8s-file` a `journald`. En este caso, podrías buscar los registros directamente utilizando la utilidad de línea de comandos `journalctl`.

Si deseas echar un vistazo al campo `log_driver` predeterminado, puedes buscar en la siguiente ruta:

```bash
# grep ^log_driver /usr/share/containers/containers.conf
log_driver = "journald"
```

Si estamos trabajando con Podman como usuario root, podemos echar un vistazo al archivo de registro disponible en el sistema host, ejecutando los siguientes pasos:

1. Primero, debemos iniciar nuestro contenedor y tomar nota del nombre asignado por Podman. El código para lograr esto se muestra en el siguiente fragmento:

```bash
# podman run -d -i -t docker.io/library/httpd
a068c7db7ff996fc74499909964c6dc6ce22a0d184659ffe5f650264cb5cb553
```

2. Después de eso, podemos tomar el nombre de nuestro contenedor, de la siguiente manera:

```bash
# podman ps
CONTAINER ID  IMAGE                           COMMAND           CREATED         STATUS         PORTS   NAMES
a068c7db7ff9  docker.io/library/httpd:latest  httpd-foreground  15 minutes ago  Up 15 minutes  80/tcp  cranky_wu
```

3. Finalmente, podemos consultar los registros de nuestro contenedor en ejecución preguntando a `journald` con la opción adecuada, especificando el nombre del contenedor:

```bash
# journalctl --user -t cranky_wu
Oct 10 09:12:42 localhost.localdomain cranky_wu[13287]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.88.0.3. Set the 'ServerName' directiv>
Oct 10 09:12:42 localhost.localdomain cranky_wu[13287]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.88.0.3. Set the 'ServerName' directiv>
Oct 10 09:12:42 localhost.localdomain cranky_wu[13287]: [Thu Oct 10 09:12:42.379839 2024] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.62 (Unix) configured -- resuming normal operat>
Oct 10 09:12:42 localhost.localdomain cranky_wu[13287]: [Thu Oct 10 09:12:42.380944 2024] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
```

¿Significa esto que debemos hacer todo este complejo procedimiento cada vez que necesitamos analizar los registros de nuestros contenedores? ¡Por supuesto que no!

Podman tiene un comando incorporado `podman logs` que puede descubrir, capturar e imprimir fácilmente los registros más recientes del contenedor por nosotros. Teniendo en cuenta el ejemplo anterior, podemos comprobar fácilmente los registros de nuestro contenedor en ejecución ejecutando el siguiente comando:

```bash
# podman logs cranky_wu
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.88.0.3. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.88.0.3. Set the 'ServerName' directive globally to suppress this message
[Thu Oct 10 09:12:42.379839 2024] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.62 (Unix) configured -- resuming normal operations
[Thu Oct 10 09:12:42.380944 2024] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
```

También podemos obtener el ID corto de nuestro contenedor en ejecución y pasar este ID al comando `podman logs`, de la siguiente manera:

```bash
# podman ps
CONTAINER ID  IMAGE                           COMMAND           CREATED         STATUS         PORTS   NAMES
a068c7db7ff9  docker.io/library/httpd:latest  httpd-foreground  19 minutes ago  Up 19 minutes  80/tcp  cranky_wu
# podman logs --tail 2 a068c7db7ff9
[Thu Oct 10 09:12:42.379839 2024] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.62 (Unix) configured -- resuming normal operations
[Thu Oct 10 09:12:42.380944 2024] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
```

En el comando anterior, también utilizamos una excelente opción del comando `podman logs`: la opción `--tail`, que nos permite mostrar solo las últimas filas necesarias del registro del contenedor. En nuestro caso, solicitamos las dos últimas.

> [!NOTE]
> La mayoría de los comandos en Podman aceptan la opción de línea de comandos `--latest`. Esto hace que el comando funcione en el contenedor creado más recientemente, lo cual es útil para situaciones como esta. Si solo tienes un contenedor, `podman logs --latest` siempre dará el resultado correcto sin tener que buscar el ID.

Ahora estamos listos para pasar a la siguiente sección, donde descubriremos otro comando útil.

---

### Ejecución de procesos en un contenedor en ejecución (*Executing processes into a running container*)

En la sección *Arquitectura sin demonio de Podman* del [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*, hablamos sobre el hecho de que Podman, al igual que cualquier otro motor de contenedores, aprovecha la funcionalidad de los namespaces de Linux para aislar correctamente los contenedores en ejecución entre sí y también del host del sistema operativo.

Por lo tanto, dado que Podman crea un namespace completamente nuevo para cada contenedor en ejecución, no debería sorprender que podamos conectarnos al mismo namespace de Linux de un contenedor en ejecución, ejecutando otros procesos tal como en un entorno operativo completo.

Podman nos brinda la capacidad de ejecutar un proceso en un contenedor en ejecución a través del comando `podman exec`.

Una vez ejecutado, este comando encontrará internamente el namespace de Linux correcto al que está conectado el contenedor en ejecución de destino. Habiendo encontrado el namespace de Linux, Podman ejecutará el proceso respectivo, pasado como argumento al comando `podman exec`, conectándolo al namespace de Linux de destino. El proceso final estará en el mismo entorno que el proceso original complementario y podrá interactuar con él; esto preserva la seguridad del namespace y del contenedor.

Para entender cómo funciona esto en la práctica, podemos considerar el siguiente ejemplo, mediante el cual primero ejecutaremos un contenedor y luego ejecutaremos un proceso junto a los procesos existentes:

```bash
# podman run -d -i -t docker.io/library/httpd
e768bf9723687194329b6d6854605849529c79f1d191d3e7139ed5f4e4ae04eb
# podman exec -ti e768bf9723687194329b6d6854605849529c79f1d191d3e7139ed5f4e4ae04eb /bin/bash
root@e768bf972368:/usr/local/apache2# ps aux
bash: ps: command not found
root@e768bf972368:/usr/local/apache2# ls -l /proc/*/exe
[omitted output]
lrwxrwxrwx. 1 root root 0 Oct 10 09:36 /proc/1/exe -> /usr/local/apache2/bin/httpd
...
```

Como puedes ver en los comandos anteriores, tomamos el ID del contenedor proporcionado por Podman una vez que se inició el contenedor y lo pasamos al comando `podman exec` como argumento.

El comando `podman exec` podría ser realmente útil para solucionar problemas, realizar pruebas y trabajar con un contenedor existente. En el ejemplo anterior, conectamos una terminal interactiva que ejecutaba la consola Bash y lanzamos el comando `ps` para inspeccionar los procesos en ejecución disponibles en el namespace de Linux actual asignado al contenedor. Sin embargo, el comando `ps` no estaba disponible, lo que significa que tal vez los desarrolladores que crearon esta imagen de contenedor decidieron no incluir el paquete `procps` para reducir el tamaño de la imagen. Entonces, para superar la ausencia del comando `ps`, simplemente miramos en el sistema de archivos `/proc`; por suerte, todo es un archivo en Linux, ¿no? La alternativa podría ser ejecutar un administrador de paquetes dentro del contenedor e instalar el paquete que contiene el comando `ps`.

El comando `podman exec` tiene muchas opciones disponibles, similares a las proporcionadas por el comando `podman run`. Como viste en el ejemplo anterior, usamos la opción para obtener un pseudoterminal conectado al flujo STDIN (`-t`), manteniendo abierto el flujo de entrada estándar incluso si aún no hay ningún terminal conectado (`-i`).

Para obtener más detalles sobre las opciones disponibles, podemos consultar el manual con el comando respectivo, como se ilustra aquí:

```bash
# man podman exec
```

Estamos avanzando en nuestro viaje por el mundo de la gestión de contenedores, y en la siguiente sección, también echaremos un vistazo a algunas de las capacidades que ofrece Podman para habilitar cargas de trabajo en contenedores en el mundo de la orquestación de contenedores de Kubernetes.

---

### Ejecución de contenedores en pods (*Running containers in pods*)

Como mencionamos en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*, Podman ofrece capacidades para comenzar a adoptar fácilmente algunos conceptos básicos del orquestador de contenedores de facto llamado Kubernetes (también conocido a veces como k8s).

El concepto de pod se introdujo con Kubernetes y representa la unidad de ejecución más pequeña en un clúster de Kubernetes. Con Podman, los usuarios pueden crear pods vacíos y luego ejecutar contenedores dentro de ellos fácilmente.

Agrupar dos o más contenedores dentro de un solo pod puede tener muchos beneficios, tales como los siguientes:

- Compartir el mismo espacio de nombres (*namespace*) de red, incluida la dirección IP
- Compartir los mismos volúmenes de almacenamiento para almacenar datos persistentes
- Compartir las mismas configuraciones
- Administrar los contenedores conjuntamente (iniciar, detener, reiniciar como una sola unidad)

Además, colocar dos o más contenedores en el mismo pod les permitirá compartir el mismo namespace de comunicación entre procesos (IPC) de Linux. Esto podría ser realmente útil para aplicaciones que necesitan comunicarse entre sí mediante memoria compartida.

La forma más sencilla de crear un pod y comenzar a trabajar con él es utilizar este comando:

```bash
# podman pod create --name myhttp
7a566073adb31df12286bfa82586d82b5004396ff10a2235d9d7cf7e7a49cb9f
# podman pod ps
POD ID        NAME    STATUS   CREATED         INFRA ID      # OF CONTAINERS
7a566073adb3  myhttp  Created  13 seconds ago  b5483a84d10b  1
```

Como se muestra en el ejemplo anterior, creamos un nuevo pod llamado `myhttp` y luego verificamos el estado del pod en nuestro sistema host: solo hay un pod en estado creado (`Created`).

Ahora podemos iniciar el pod de la siguiente manera y comprobar qué sucederá:

```bash
# podman pod start myhttp
myhttp
# podman pod ps
POD ID        NAME    STATUS   CREATED             INFRA ID      # OF CONTAINERS
7a566073adb3  myhttp  Running  About a minute ago  b5483a84d10b  1
```

El pod ahora se está ejecutando, pero ¿qué está ejecutando Podman en realidad? ¡Creamos un pod vacío sin contenedores dentro! Echemos un vistazo al contenedor en ejecución ejecutando el comando `podman ps`, de la siguiente manera:

```bash
# podman ps
CONTAINER ID  IMAGE                                      COMMAND  CREATED        STATUS             PORTS  NAMES
b5483a84d10b  localhost/podman-pause:5.2.2-1724198400           2 minutes ago  Up About a minute         7a566073adb3-infra
```

El comando `podman ps` muestra un contenedor en ejecución con una imagen llamada `pause`. Podman ejecuta este contenedor de forma predeterminada como un contenedor de infraestructura (*infra container*). Este tipo de contenedor no hace nada: simplemente mantiene el namespace y permite que el motor de contenedores se conecte a cualquier otro contenedor en ejecución dentro del pod.

Una vez desmitificado el papel de este contenedor especial dentro de nuestros pods, ahora podemos echar un breve vistazo a los pasos necesarios para iniciar un pod de múltiples contenedores.

En primer lugar, comencemos ejecutando un nuevo contenedor dentro del pod existente que creamos en el ejemplo anterior, de la siguiente manera:

```bash
# podman run --pod myhttp -d -i -t docker.io/library/httpd
9c564757d9b34f6c3eb9e14672b9e6d8e639fb91707143a44356cde6ee6a4a4b
```

Luego, podemos verificar si el pod existente ha actualizado la cantidad de contenedores que contiene, como se ilustra en el siguiente fragmento de código:

```bash
# podman pod ps
POD ID        NAME    STATUS   CREATED        INFRA ID      # OF CONTAINERS
7a566073adb3  myhttp  Running  3 minutes ago  b5483a84d10b  2
```

Finalmente, podemos pedirle a Podman una lista de los contenedores en ejecución con el nombre del pod asociado, de la siguiente manera:

```bash
# podman ps -p
CONTAINER ID  IMAGE                                      COMMAND           CREATED         STATUS        PORTS  NAMES               POD ID        PODNAME
b5483a84d10b  localhost/podman-pause:5.2.2-1724198400                     3 minutes ago   Up 2 minutes         7a566073adb3-infra  7a566073adb3  myhttp
9c564757d9b3  docker.io/library/httpd:latest            httpd-foreground  37 seconds ago  Up 37 seconds        goofy_nobel         7a566073adb3  myhttp
```

Como puedes ver, los dos contenedores en ejecución están asociados con el pod llamado `myhttp`.

> [!IMPORTANT]
> Considera la posibilidad de limpiar periódicamente el entorno del laboratorio después de completar todos los ejemplos contenidos en este capítulo. Esto podría ayudarte a ahorrar recursos y evitar errores al pasar a los ejemplos del siguiente capítulo. Los dos comandos que podrían ayudarte aquí son `podman rm --all --force` y `podman system reset`, que en realidad limpian el entorno de Podman y restablecen la configuración de Podman a sus valores predeterminados.
> Por esta razón, puedes consultar el código proporcionado en la carpeta `AdditionalMaterial` en el repositorio de GitHub del libro: [https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition](https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition).

Con el mismo enfoque, podemos agregar más y más contenedores al mismo pod, permitiéndoles compartir todos los datos que describimos anteriormente.

Ten en cuenta que colocar contenedores en el mismo pod puede resultar beneficioso en algunos casos, pero representa un antipatrón para la tecnología de contenedores. De hecho, como se mencionó anteriormente, Kubernetes considera el pod como la unidad informática más pequeña para ejecutarse sobre los nodos distribuidos que forman parte de un clúster. Esto significa que una vez que agrupas dos o más contenedores en el mismo pod, se ejecutarán juntos en el mismo nodo y el orquestador no podrá equilibrar ni distribuir su carga de trabajo en múltiples máquinas.

¡Exploraremos más sobre las características de Podman que te permitirán ingresar al mundo de la orquestación de contenedores a través de Kubernetes en los siguientes capítulos!

---

### Resumen

En este capítulo, comenzamos a desarrollar experiencia en la administración de contenedores, comenzando con imágenes de contenedores y luego trabajando con contenedores en ejecución. Una vez que nuestros contenedores estuvieron en funcionamiento, también exploramos los diversos comandos disponibles en Podman para inspeccionar y verificar los registros y solucionar problemas en nuestros contenedores. Las operaciones necesarias para monitorear y cuidar los contenedores en ejecución son realmente importantes para cualquier administrador de contenedores. Por último, también echamos un breve vistazo a los conceptos de Kubernetes disponibles en Podman que nos permiten agrupar dos o más contenedores bajo el mismo namespace de Linux. Todos los conceptos y ejemplos por los que acabamos de pasar nos ayudarán a iniciar nuestra experiencia como administradores de sistemas para tecnologías de contenedores.

¡Ahora estamos listos para explorar otro tema importante en el próximo capítulo: la gestión del almacenamiento para nuestros contenedores!
