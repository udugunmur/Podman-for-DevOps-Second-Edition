# Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

## Capítulo 5: Implementación del Almacenamiento para los Datos del Contenedor

En los capítulos anteriores, exploramos cómo ejecutar y administrar nuestros contenedores utilizando Podman, pero pronto nos daremos cuenta en este capítulo de que estas operaciones no son útiles en ciertos escenarios donde las aplicaciones incluidas en nuestros contenedores necesitan almacenar datos de modo persistente. Los contenedores son efímeros por defecto, y esta es una de sus características principales, como describimos en el primer capítulo de este libro; por esta razón, necesitamos una forma de adjuntar almacenamiento persistente a un contenedor en ejecución para preservar los datos importantes del contenedor.

En este capítulo, cubriremos los siguientes temas principales:

- ¿Por qué es importante el almacenamiento para los contenedores?
- Características del almacenamiento de los contenedores
- Copiar archivos hacia y desde un contenedor
- Adjuntar almacenamiento del host a un contenedor

---

### Requisitos técnicos

Antes de continuar con los tutoriales prácticos del capítulo, se requiere una máquina con una instalación de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Por último, una buena comprensión de los temas tratados en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, es útil para asimilar fácilmente conceptos sobre imágenes OCI y la ejecución de contenedores.

---

### ¿Por qué es importante el almacenamiento para los contenedores?

Antes de avanzar en el capítulo y responder a esta interesante pregunta, debemos distinguir entre dos tipos de almacenamiento para contenedores:

- Almacenamiento externo adjunto a los contenedores en ejecución para almacenar datos, haciéndolos persistentes al reinicio del contenedor.
- Almacenamiento subyacente para los sistemas de archivos raíz de nuestros contenedores e imágenes de contenedor.

Hablando de almacenamiento externo, como describimos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*, los contenedores son sin estado (*stateless*), efímeros y a menudo tienen un sistema de archivos de solo lectura. Esto se debe a que la teoría detrás de la tecnología establece que los contenedores deben usarse para generar aplicaciones escalables y distribuidas que tienen que escalar horizontalmente en lugar de verticalmente.

Escalar una aplicación horizontalmente significa que, en caso de que necesitemos recursos adicionales para nuestros servicios en ejecución, no aumentaremos la CPU o la RAM de un solo contenedor en ejecución, sino que iniciaremos un contenedor completamente nuevo que manejará las solicitudes entrantes junto con el contenedor existente. Este es el mismo paradigma bien conocido adoptado en la nube pública. El contenedor, en principio, debe ser efímero porque cualquier copia adicional de la imagen del contenedor existente debe poder ejecutarse cuando sea necesario para respaldar el servicio que se está ejecutando actualmente.

Por supuesto, existen excepciones, y podría suceder que un contenedor en ejecución no se pueda escalar horizontalmente o que simplemente necesite compartir configuraciones, caché o cualquier otro dato relevante con otras copias de las mismas imágenes de contenedor al momento del inicio o durante el tiempo de ejecución.

Comprendamos esto con la ayuda de un ejemplo de la vida real. Usar un servicio de vehículos compartidos (*car-sharing*) para obtener un auto nuevo para cada destino dentro de una ciudad puede ser una forma útil e inteligente de moverse sin preocuparse por las tarifas de estacionamiento, el combustible y otras cosas. Sin embargo, al mismo tiempo, este servicio no puede permitirte almacenar o dejar tus cosas dentro de un auto estacionado. Por lo tanto, al utilizar un servicio de vehículos compartidos, podemos desempacar nuestras cosas una vez que subimos a un auto, pero debemos empacarlas de nuevo antes de salir de ese auto. Lo mismo se aplica de manera similar a los contenedores, donde debemos adjuntarles algo de almacenamiento para permitir que nuestro contenedor escriba datos, pero luego, una vez que nuestro contenedor se detiene, debemos desconectar ese almacenamiento para que un contenedor completamente nuevo pueda usarlo cuando sea necesario.

He aquí otro ejemplo más técnico: consideremos una aplicación estándar de tres capas con un servicio web, un backend y un servicio de base de datos. Cada capa de esta aplicación puede necesitar almacenamiento, que utilizará de diversas maneras. El servicio web puede necesitar un lugar para guardar caché, almacenar páginas web renderizadas, algunas imágenes personalizadas en tiempo de ejecución, etc. El servicio de backend necesitará un lugar para almacenar datos de configuración y sincronización entre los otros servicios de backend en ejecución, si los hubiera, etc. El servicio de base de datos seguramente necesitará un lugar para almacenar los datos.

El almacenamiento a menudo se asocia con infraestructura de bajo nivel, pero en un contenedor, el almacenamiento se vuelve importante incluso para los desarrolladores, quienes deben planificar dónde adjuntar el almacenamiento y las características necesarias para su aplicación.

Si extendemos el tema a la orquestación de contenedores, entonces el almacenamiento hereda un papel estratégico porque debe ser tan elástico y factible como el orquestador de Kubernetes con el que podríamos usarlo. En este caso, el almacenamiento de contenedores debería parecerse más al almacenamiento definido por software (*software-defined storage*), capaz de proporcionar recursos de almacenamiento de forma autoservicio a los desarrolladores y a los contenedores en general.

Aunque este libro hablará sobre el almacenamiento local, es importante tener en cuenta que esto no es suficiente para el orquestador de Kubernetes porque los contenedores deben ser portátiles de un host a otro, según la disponibilidad y las reglas de escalado definidas. ¡Aquí es donde el almacenamiento definido por software podría ser la solución!

Como podemos deducir de los ejemplos anteriores, el almacenamiento externo importa en los contenedores. El uso puede variar según la aplicación que se ejecute dentro de nuestro contenedor, pero es obligatorio. Al mismo tiempo, otro papel clave lo desempeña el almacenamiento de contenedores subyacente, que es responsable de gestionar el almacenamiento correcto de los contenedores y el sistema de archivos raíz de las imágenes de los contenedores. Elegir el almacenamiento local subyacente correcto, estable y de buen rendimiento garantizará una mejor y correcta administración de nuestros contenedores.

Entonces, primero exploremos un poco la teoría del almacenamiento de contenedores y luego analicemos cómo trabajar con él.

---

### Características del almacenamiento de los contenedores

Antes de entrar en un ejemplo real y casos de uso, primero debemos profundizar en las principales diferencias entre el almacenamiento de contenedores y una interfaz de almacenamiento de contenedores (CSI, *Container Storage Interface*).

El almacenamiento de contenedores, anteriormente denominado almacenamiento de contenedores subyacente, es responsable de manejar las imágenes de contenedores en sistemas de archivos de copia en escritura (COW, *copy-on-write*). Las imágenes de contenedores deben administrarse hasta que se le indique a un motor de contenedores que las ejecute, por lo que necesitamos una forma de almacenar la imagen hasta que se ejecute. Esa es la función del almacenamiento de contenedores.

Una vez que comenzamos a usar un orquestador como Kubernetes, CSI se encarga de proporcionar el almacenamiento de bloques o archivos que los contenedores necesitan para escribir datos.

En la siguiente sección, nos centraremos en el almacenamiento de contenedores y su configuración. Más adelante, hablaremos sobre el almacenamiento externo para contenedores y las opciones que tenemos en Podman para exponer el almacenamiento local del host a los contenedores en ejecución.

Una gran innovación introducida con Podman es el proyecto `containers/storage` ([https://github.com/containers/storage](https://github.com/containers/storage)), una excelente manera de compartir un método común subyacente para acceder al almacenamiento de contenedores en un host. Con la llegada de Docker, nos vimos obligados a pasar por el demonio de Docker para interactuar con el almacenamiento de contenedores. Sin otra forma de interactuar directamente con el almacenamiento subyacente, el demonio de Docker simplemente lo ocultaba tanto al usuario como al administrador del sistema.

Con el proyecto `containers/storage`, ahora tenemos una manera fácil de usar múltiples herramientas para analizar, administrar o trabajar con el almacenamiento de contenedores al mismo tiempo. La configuración de esta pieza de software de bajo nivel es muy importante para Podman y sus herramientas complementarias, como Buildah, Skopeo y CRI-O, y se puede inspeccionar o editar a través de su archivo de configuración, disponible en `/etc/containers/storage.conf`.

Al observar el archivo de configuración, podemos descubrir fácilmente que podemos cambiar muchas opciones en términos de cómo interactúan nuestros contenedores con el almacenamiento subyacente. Inspeccionemos la opción más importante: el controlador de almacenamiento (*storage driver*).

#### Controlador de almacenamiento (*Storage driver*)

El archivo de configuración, como una de sus primeras opciones, permite elegir el controlador de almacenamiento de contenedores COW predeterminado. El archivo de configuración en la versión actual, al momento de escribir este libro, admite los siguientes controladores COW:

- `overlay`
- `vfs`
- `btrfs`
- `zfs`

Estos también se denominan a menudo controladores de gráficos (*graph drivers*) porque la mayoría de ellos organizan las capas que manejan en una estructura de grafo.

Cuando se utiliza Fedora 40 y Podman 5.2.2, el archivo de configuración de almacenamiento del contenedor incluye `overlay` configurado como controlador predeterminado.

Antes de sumergirnos en las otras opciones disponibles y, más adelante, en los ejemplos prácticos contenidos en este capítulo, echemos un vistazo más de cerca a cómo funciona uno de estos controladores de sistema de archivos COW.

El sistema de archivos de unión OverlayFS ha estado presente en el kernel de Linux desde la versión 3.18. Por lo general, está habilitado de forma predeterminada y se activa dinámicamente una vez que se inicia un montaje con este sistema de archivos. El mecanismo detrás de este sistema de archivos es realmente simple pero poderoso: permite superponer múltiples árboles de directorios entre sí, almacenando solo las diferencias, pero mostrando el árbol de directorios más reciente, actualizado y aplanado (*squashed*).

Por lo general, en el mundo de los contenedores, comenzamos utilizando un sistema de archivos de solo lectura, agregando una o más capas, de solo lectura nuevamente, hasta que un contenedor en ejecución utilice este conjunto de capas aplanadas como su sistema de archivos raíz. Aquí es donde se creará la última capa de lectura/escritura como una superposición de las demás.

Veamos qué sucede bajo el capó una vez que descargamos una imagen de contenedor completamente nueva con Podman:

> [!NOTE]
> Si deseas continuar probando el siguiente ejemplo en tu máquina de prueba, asegúrate de eliminar los contenedores e imágenes de contenedor en ejecución para que coincidan fácilmente con las capas que Podman descargará para nosotros.

```bash
# podman pull docker.io/library/httpd:latest
Trying to pull docker.io/library/httpd:latest......
Getting image source signatures
Copying blob 6bd9d3710aae done   | 
Copying blob 4f4fb700ef54 skipped: already exists
Copying blob a480a496ba95 done   | 
Copying blob 3a2663e66670 done   | 
Copying blob dbde712f81fb done   | 
Copying blob 867b2ea3628d done   | 
Copying config 1bcf11fa15 done   | 
Writing manifest to image destination
1bcf11fa154f23987201bd92a75bf75e3507fc49f415d5dfe35887d1be3fd596
```

Podemos ver en la salida del comando anterior que se han descargado varias capas. Esto se debe a que la imagen del contenedor que descargamos está compuesta por muchas capas.

Ahora podemos comenzar a inspeccionar solo las capas descargadas. En primer lugar, tenemos que localizar el directorio correcto, que podemos buscar dentro del archivo de configuración. Alternativamente, podemos utilizar una forma más sencilla. Podman tiene un comando dedicado a mostrar su configuración en ejecución y otra información útil: `podman info`. Veamos cómo funciona:

```bash
# podman info | grep -A17 "store:"
store:
  configFile: /usr/share/containers/storage.conf
  containerStore:
    number: 4
    paused: 0
    running: 0
    stopped: 4
  graphDriverName: overlay
  graphOptions:
    overlay.imagestore: /usr/lib/containers/storage
    overlay.mountopt: nodev,metacopy=on
  graphRoot: /var/lib/containers/storage
  graphRootAllocated: 4212109312
  graphRootUsed: 1196261376
  graphStatus:
    Backing Filesystem: btrfs
    Native Overlay Diff: "false"
    Supports d_type: "true": "true"
    Supports shifting: "true"
    Supports volatile: "true"
    Using metacopy: "true": "true"
  imageCopyTmpDir: /var/tmptmp
  imageStore::
    number: 4
  runRoot: /run/containers/storage: /run/containers/storage
  transientStore: false: false
  volumePath: /var/lib/containers/storage/volumes: /var/lib/containers/storage/volumes
```

Para reducir la salida del comando `podman info`, usamos el comando `grep` para que solo coincida con la sección `store:` que contiene la configuración actual vigente para el almacenamiento del contenedor. Como alternativa, también puedes usar la utilidad de comando `jq` para filtrar la salida solicitada en formato JSON con el siguiente comando:

```bash
$ podman info --format json | jq .store
```

Como podemos ver, el controlador utilizado es `overlay`, y el directorio raíz para buscar nuestras capas se reporta como el directorio `graphRoot`: `/var/lib/containers/storage`; para contenedores rootless, el equivalente es `$HOME/.local/share/containers/storage`. También tenemos otras rutas reportadas, pero hablaremos de ellas más adelante en esta sección. La palabra clave *graph* es un término derivado de la categoría de controladores que acabamos de presentar anteriormente.

Echemos un vistazo a ese directorio para ver cuál es el contenido real:

```bash
# cd /var/lib/containers/storage
# ls
db.sql                  defaultNetworkBackend  libpod   overlay  overlay-containers  overlay-images  overlay-layers  secrets  storage.lock  tmp  userns.lock  volumes
```

Tenemos varios directorios disponibles cuyos nombres se explican por sí solos. Los que estamos buscando son los siguientes:

- **overlay-images**: Contiene los metadatos de las imágenes de contenedores descargadas.
- **overlay-layers**: Contiene los archivos comprimidos de todas las capas de cada imagen de contenedor.
- **overlay**: Este es el directorio que contiene las capas descomprimidas de cada imagen de contenedor.

Comprobemos el contenido del primer directorio, `overlay-images`:

```bash
# ls -l overlay-images/
total 8
drwx------. 1 root root  646 Oct 19 13:44 1bcf11fa154f23987201bd92a75bf75e3507fc49f415d5dfe35887d1be3fd596
-rw-------. 1 root root 1505 Oct 19 13:44 images.json
-rw-r--r--. 1 root root   64 Oct 19 13:44 images.lock
```

Como podemos imaginar, en este directorio podemos encontrar los metadatos de la única imagen de contenedor que descargamos y, en el directorio con un ID muy largo, encontraremos el archivo de manifiesto (*manifest*) que describe las capas que componen nuestra imagen de contenedor.

Comprobemos ahora el contenido del segundo directorio, `overlay-layers`:

```bash
# ls -l overlay-layers/
total 388
-rw-------. 1 root root    412 Oct 19 13:44 7555495f10f71578f8bd7904214940724f63e98660a813f4fa6aa856dd22d8b7.tar-split.gz
-rw-------. 1 root root 272894 Oct 19 13:44 98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad.tar-split.gz
-rw-------. 1 root root    327 Oct 19 13:44 a3a542fc439141fb90369c89fdd23875fd4ec423983bd2631c2116b208776e6f.tar-split.gz
-rw-------. 1 root root  50969 Oct 19 13:44 c126300979bb0dfc017decea18ab61d3ea542febea9d4fe90f333e1cb3666db2.tar-split.gz
-rw-------. 1 root root  42672 Oct 19 13:44 c1a2d7288c9a4fc91cbfdc931f742cdb5aed45da08b2e6c780228856f30bb579.tar-split.gz
-rw-------. 1 root root     77 Oct 19 13:44 c1a9d7e93770930ea7a795125d0a2ba54bf7689d7c2b7dbbe7072971f1cd78fd.tar-split.gz
-rw-------. 1 root root   2729 Oct 19 13:44 layers.json
-rw-r--r--. 1 root root     64 Oct 19 13:44 layers.lock
-rw-------. 1 root root      2 Oct 10 09:43 volatile-layers.json
```

Como podemos ver, acabamos de encontrar todos los archivos de capas descargados para nuestra imagen de contenedor, pero ¿dónde se han descomprimido? La respuesta es sencilla: en la tercera carpeta, `overlay`:

```bash
# ls -l overlay/
total 0
drwx------. 1 root root  46 Oct 19 13:44 7555495f10f71578f8bd7904214940724f63e98660a813f4fa6aa856dd22d8b7
drwx------. 1 root root  46 Oct 19 13:44 98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad
drwx------. 1 root root  46 Oct 19 13:44 a3a542fc439141fb90369c89fdd23875fd4ec423983bd2631c2116b208776e6f
drwx------. 1 root root  46 Oct 19 13:44 c126300979bb0dfc017decea18ab61d3ea542febea9d4fe90f333e1cb3666db2
drwx------. 1 root root  46 Oct 19 13:44 c1a2d7288c9a4fc91cbfdc931f742cdb5aed45da08b2e6c780228856f30bb579
drwx------. 1 root root  46 Oct 19 13:44 c1a9d7e93770930ea7a795125d0a2ba54bf7689d7c2b7dbbe7072971f1cd78fd
drwxr-xr-x. 1 root root 312 Oct 19 13:44 l
```

La primera pregunta que podría surgir al mirar el contenido más reciente del directorio es: *¿Cuál es el propósito del directorio `l` (L en minúscula)?*

Para responder a esta pregunta, tenemos que inspeccionar el contenido del directorio de una capa. Podemos comenzar con la primera de la lista:

```bash
# ls -la overlay/7555495f10f71578f8bd7904214940724f63e98660a813f4fa6aa856dd22d8b7/
total 8
drwx------. 1 root root  46 Oct 19 13:44 .
drwx------. 1 root root 806 Oct 19 13:44 ..
dr-xr-xr-x. 1 root root   6 Oct 19 13:44 diff
-rw-r--r--. 1 root root  26 Oct 19 13:44 link
-rw-r--r--. 1 root root 144 Oct 19 13:44 lower
drwx------. 1 root root   0 Oct 19 13:44 merged
drwx------. 1 root root   0 Oct 19 13:44 work
```

Entendamos el propósito de estos archivos y directorios:

- **diff**: Este directorio representa la capa superior (*upper layer*) de la superposición y se utiliza para almacenar cualquier cambio en la capa.
- **lower**: Este archivo reporta todos los montajes de capas inferiores, ordenados de la más alta a la más baja.
- **merged**: Este directorio es aquel sobre el que se monta la superposición (*overlay*).
- **work**: Este directorio se utiliza para operaciones internas.
- **link**: Este archivo contiene una cadena única para la capa.

Ahora, volviendo a nuestra pregunta (*¿Cuál es el propósito del directorio `l` (L en minúscula)?*), debajo del directorio `l` hay enlaces simbólicos con cadenas únicas que apuntan al directorio `diff` para cada capa. Los enlaces simbólicos hacen referencia a capas inferiores en el archivo `lower`. Comprobémoslo:

```bash
# ls -la overlay/l/
total 24
drwxr-xr-x. 1 root root 312 Oct 19 13:44 .
drwx------. 1 root root 806 Oct 19 13:44 ..
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 3HBRITCBL5MXZGZHDNTU7LPC2X -> ../c126300979bb0dfc017decea18ab61d3ea542febea9d4fe90f333e1cb3666db2/diff
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 43LDU3L34EOOEP5NFGNDBLNNNV -> ../98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad/diff
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 5FLKM24KJBZT2FVZI75PYPHEBW -> ../a3a542fc439141fb90369c89fdd23875fd4ec423983bd2631c2116b208776e6f/diff
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 Q3ZVJUXUYIUWYRKBQYSD7R6OXT -> ../c1a2d7288c9a4fc91cbfdc931f742cdb5aed45da08b2e6c780228856f30bb579/diff
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 RGJRWXLMSBUBJF4VR3RDLGFXP2 -> ../7555495f10f71578f8bd7904214940724f63e98660a813f4fa6aa856dd22d8b7/diff
lrwxrwxrwx. 1 root root  72 Oct 19 13:44 YJ4PBSR4UDAJMJVMWVK5P6WP2H -> ../c1a9d7e93770930ea7a795125d0a2ba54bf7689d7c2b7dbbe7072971f1cd78fd/diff
```

Para verificar nuevamente lo que acabamos de aprender, busquemos la primera capa de nuestra imagen de contenedor y comprobemos si hay un archivo `lower` para ella.

Inspeccionemos el archivo de manifiesto para nuestra imagen de contenedor:

```bash
# cat overlay-images/1bcf11fa154f23987201bd92a75bf75e3507fc49f415d5dfe35887d1be3fd596/manifest | head -14
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.oci.image.manifest.v1+json",
   "config": {
      "mediaType": "application/vnd.oci.image.config.v1+json",
      "digest": "sha256:1bcf11fa154f23987201bd92a75bf75e3507fc49f415d5dfe35887d1be3fd596",
      "size": 8015
   },
   "layers": [
      {
         "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
         "digest": "sha256:a480a496ba95a197d587aa1d9e0f545ca7dbd40495a4715342228db62b67c4ba",
         "size": 29126289
      },
```

Luego, debemos comparar la suma de verificación (*checksum*) del archivo comprimido con la lista de todas las capas que descargamos:

> [!NOTE]
> SHA-256 es un algoritmo utilizado para producir un hash criptográfico único que podría usarse para verificar la integridad de un archivo (checksum). Los desarrolladores de Podman están trabajando para admitir otros algoritmos hash, como SHA-512, que ofrecerán propiedades de seguridad superiores.

```bash
# cat overlay-layers/layers.json | jq | grep -B3 -A10 "sha256:a480a"
  {
    "id": "98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad",
    "created": "2024-10-19T13:44:05.710510598Z",
    "compressed-diff-digest": "sha256:a480a496ba95a197d587aa1d9e0f545ca7dbd40495a4715342228db62b67c4ba",
    "compressed-size": 29126289,
    "diff-digest": "sha256:98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad",
    "diff-size": 77832192,
    "compression": 2,
    "uidset": [
      0
    ],
    "gidset": [
      0,
      8,
```

El archivo que acabamos de analizar, `overlay-layers/layers.json`, no estaba sangrado. Por esta razón, usamos la utilidad `jq` para darle formato y hacerlo legible para humanos.

> [!NOTE]
> Si no puedes encontrar la utilidad `jq` en tu sistema, puedes instalarla a través del administrador de paquetes predeterminado del sistema operativo. En Fedora, por ejemplo, puedes ejecutar `dnf install jq`.

Como puedes ver, acabamos de encontrar el ID de nuestra capa raíz aprovechando la utilidad `grep` buscando el principio de la cadena hash de nuestra imagen de contenedor, `"sha256:a480a"`. Ahora, miremos su contenido, inspeccionando el directorio con el mismo ID que encontramos en la salida del comando anterior, `98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad`:

```bash
# ls -l overlay/98b5f35ea9d3eca6ed1881b5fe5d1e02024e1450822879e4c13bb48c9386d0ad
total 4
dr-xr-xr-x. 1 root root 132 Oct 19 13:44 diff
drwx------. 1 root root   0 Oct 19 13:44 empty
-rw-r--r--. 1 root root  26 Oct 19 13:44 link
drwx------. 1 root root   0 Oct 19 13:44 merged
drwx------. 1 root root   0 Oct 19 13:44 work
```

Como podemos verificar, ¡no hay ningún archivo `lower` dentro del directorio de la capa porque esta es la primera capa de nuestra imagen de contenedor!

La diferencia que podríamos notar es la presencia de un directorio llamado `empty`. Esto se debe a que si una capa no tiene un padre, entonces el sistema de superposición creará un directorio inferior ficticio llamado `empty` y omitirá la escritura de un archivo `lower`.

Finalmente, como última etapa de nuestro ejemplo práctico, ejecutemos nuestro contenedor y verifiquemos que se creará una nueva capa `diff`. Esperamos que esta capa contenga solo la diferencia entre las inferiores.

> [!IMPORTANT]
> En el contexto de los controladores de almacenamiento de Podman (particularmente el controlador OverlayFS predeterminado), una capa diff es el directorio real en el disco del host que contiene los cambios específicos del sistema de archivos introducidos por un solo contenedor o capa de imagen.
> Si bien las capas son una forma conceptual de pensar en la construcción de imágenes, diff es la implementación técnica de esa capa en tu almacenamiento local.

Primero, ejecutamos la imagen del contenedor que acabamos de analizar:

```bash
# podman run -d docker.io/library/httpd:latest
9f84f1cbadf2c482dd2c4fa3ad350332bc956a4711e09decd9c66c1759f4f345
```

Como puedes ver, lo iniciamos en segundo plano mediante la opción `-d` para continuar trabajando en el host del sistema. Después de esto, ejecutaremos una nueva shell en el pod para verificar la carpeta raíz del contenedor y crear un nuevo archivo en ella:

```bash
# podman exec -ti 9f84f1cbadf2c482dd2c4fa3ad350332bc956a4711e09decd9c66c1759f4f345 /bin/bash
root@9f84f1cbadf2:/usr/local/apache2# pwd
/usr/local/apache2
root@9f84f1cbadf2:/usr/local/apache2# echo "this is my NOT persistent data" > tempfile.txt
root@9f84f1cbadf2:/usr/local/apache2# ls
bin  build  cgi-bin  conf  error  htdocs  icons  include  logs  modules  tempfile.txt
```

Este nuevo archivo que acabamos de crear será temporal y solo durará durante la vida útil del contenedor. Ahora es el momento de encontrar la capa diff recién creada por el controlador overlay en nuestro sistema host. La forma más sencilla es analizar los puntos de montaje utilizados en el contenedor en ejecución:

```bash
root@9f84f1cbadf2:/usr/local/apache2# mount | grep overlay
overlay on / type overlay (rw,relatime,context="system_u:object_r:container_file_t:s0:c560,c898",lowerdir=/var/lib/containers/storage/overlay/l/RGJRWXLMSBUBJF4VR3RDLGFXP2:/var/lib/containers/storage/overlay/l/3HBRITCBL5MXZGZHDNTU7LPC2X:/var/lib/containers/storage/overlay/l/Q3ZVJUXUYIUWYRKBQYSD7R6OXT:/var/lib/containers/storage/overlay/l/YJ4PBSR4UDAJMJVMWVK5P6WP2H:/var/lib/containers/storage/overlay/l/5FLKM24KJBZT2FVZI75PYPHEBW:/var/lib/containers/storage/overlay/l/43LDU3L34EOOEP5NFGNDBLNNNV,upperdir=/var/lib/containers/storage/overlay/88633bde45bb8c7c25b5f42af4ba35d273fa7bf2c9fe689ae653675845dec9fc/diff,workdir=/var/lib/containers/storage/overlay/88633bde45bb8c7c25b5f42af4ba35d273fa7bf2c9fe689ae653675845dec9fc/work,redirect_dir=on,uuid=on,metacopy=on)
```

Para contenedores rootless, el comando anterior deberá ejecutarse dentro de la shell `podman unshare`.

Como puedes ver, el primer punto de montaje de la lista muestra una línea muy larga llena de rutas de capa divididas por dos puntos. En esta larga línea, podemos encontrar el directorio `upperdir` que estamos buscando:

```text
upperdir=/var/lib/containers/storage/overlay/88633bde45bb8c7c25b5f42af4ba35d273fa7bf2c9fe689ae653675845dec9fc/diff
```

Ahora podemos inspeccionar el contenido de este directorio y navegar a través de las diversas rutas disponibles para encontrar el directorio raíz del contenedor donde escribimos ese archivo en los comandos anteriores:

```bash
# ls -la /var/lib/containers/storage/overlay/88633bde45bb8c7c25b5f42af4ba35d273fa7bf2c9fe689ae653675845dec9fc/diff/usr/local/apache2/
total 4
drwxr-xr-x. 1  33 tape 32 Oct 19 14:05 .
drwxr-xr-x. 1 root root 14 Oct 17 01:19 ..
drwxr-xr-x. 1 root root 18 Oct 19 14:04 logs
-rw-r--r--. 1 root root 31 Oct 19 14:05 tempfile.txt
# cat /var/lib/containers/storage/overlay/88633bde45bb8c7c25b5f42af4ba35d273fa7bf2c9fe689ae653675845dec9fc/diff/usr/local/apache2/tempfile.txt
this is my NOT persistent data
```

Como comprobamos, los datos se almacenan en el sistema operativo host, ¡pero se almacenan en una capa temporal que tarde o temprano se eliminará una vez que se elimine el contenedor!

#### El archivo de configuración storage.conf

Ahora, volviendo al tema original que nos envió a este pequeño viaje bajo el capó del controlador de almacenamiento overlay, estábamos hablando de `/etc/containers/storage.conf`. Este archivo contiene todas las configuraciones para el proyecto `containers/storage`, que es responsable de compartir un método común subyacente para acceder al almacenamiento de contenedores en un host. Las otras opciones disponibles en este archivo están relacionadas con la personalización del controlador de almacenamiento, así como con el cambio de las rutas predeterminadas para los directorios de almacenamiento interno. El principal de ellos es `graphroot` (a menudo denominado la ruta raíz). Esta es la ubicación de almacenamiento persistente donde Podman guarda todas las capas de imágenes descargadas, los metadatos de las imágenes locales y las capas diff que discutimos anteriormente.

A continuación, el directorio `runroot` tiene un propósito diferente. Mientras que `graphroot` maneja el almacenamiento a largo plazo, `runroot` se usa para almacenar datos temporales y volátiles producidos por contenedores activos. Esto incluye archivos de estado, archivos de bloqueo y los puntos de montaje fusionados que existen solo mientras se ejecuta un contenedor. Debido a que estos datos son temporales y a menudo requieren un alto rendimiento, se almacenan con frecuencia en un sistema de archivos respaldado por memoria (como `tmpfs`) para garantizar la velocidad y asegurar que se borren al reiniciar el sistema.

Si inspeccionamos la carpeta en nuestro host en ejecución donde iniciamos el contenedor para el ejemplo anterior, encontraremos que esta es un área de almacenamiento interno general para archivos de contenedor, incluidos archivos PID y archivos de registro internos, así como ciertos archivos que Podman genera automáticamente y se montan en el contenedor:

```bash
# ls -l /run/containers/storage/overlay-containers/9f84f1cbadf2c482dd2c4fa3ad350332bc956a4711e09decd9c66c1759f4f345/userdata/
total 20
-rw-r--r--. 1 root root    4 Oct 19 14:04 conmon.pid
-rw-r--r--. 1 root root   13 Oct 19 14:04 hostname
-rw-r--r--. 1 root root  243 Oct 19 14:04 hosts
-rw-r--r--. 1 root root    0 Oct 19 14:04 oci-log
-rwx------. 1 root root    4 Oct 19 14:04 pidfile
-rw-r--r--. 1 root root   25 Oct 19 14:04 resolv.conf
drwxr-xr-x. 3 root root   60 Oct 19 14:04 run
```

Como puedes ver en la salida anterior, la carpeta del contenedor bajo la ruta `runroot` contiene varios archivos que se han montado directamente en el contenedor para personalizarlo.

Para resumir, en los ejemplos anteriores, analizamos la anatomía de una imagen de contenedor y lo que sucede una vez que ejecutamos un nuevo contenedor a partir de esa imagen. La tecnología detrás de escena es asombrosa y vimos que muchas funciones están relacionadas con las capacidades de aislamiento que ofrece el sistema operativo. Aquí, el almacenamiento ofrece otras funcionalidades importantes que han convertido a los contenedores en la gran tecnología que todos conocemos hoy. Estrechamente ligada al almacenamiento de contenedores está la capacidad de acceder a archivos locales o remotos desde dentro de un contenedor. En la siguiente sección, exploraremos varias técnicas para gestionar este intercambio de datos de manera eficaz.

---

### Copiar archivos hacia y desde un contenedor

Podman permite a los usuarios mover archivos dentro y fuera de un contenedor en ejecución. Existen dos métodos principales para lograr este objetivo:

- Usar el comando `podman cp`
- Montar el sistema de archivos del contenedor

Veamos estas dos técnicas más a fondo con algunos ejemplos.

#### Uso de podman cp

Este resultado se logra mediante el comando `podman cp`, que puede mover archivos y carpetas hacia y desde un contenedor. Su uso es bastante simple y se ilustrará en el siguiente ejemplo.

Primero, iniciemos un nuevo contenedor Alpine:

```bash
$ podman run -d --name alpine_cp_test alpine sleep 1000
```

Ahora, tomemos un archivo del contenedor: hemos elegido el archivo `/etc/os-release`, que proporciona información sobre la distribución y su ID de versión:

```bash
$ podman cp alpine_cp_test:/etc/os-release /tmp
```

El archivo se ha copiado a la carpeta `/tmp` del host y se puede inspeccionar:

```bash
$ cat /tmp/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.20.3
PRETTY_NAME="Alpine Linux v3.20"
HOME_URL=https://alpinelinux.org/
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
```

En la dirección opuesta, podemos copiar archivos o carpetas desde el host al contenedor en ejecución o desde un contenedor a otro:

```bash
$ podman cp /tmp/build_folder alpine_cp_test:/
```

Este ejemplo copia la carpeta `/tmp/build_folder`, y todo su contenido, bajo el sistema de archivos raíz del contenedor Alpine. Luego podemos inspeccionar el resultado del comando de copia usando `podman exec` con el comando de utilidad `ls`.

#### Interacción con overlayfs

Existe otra forma de copiar archivos desde un contenedor al host, que es mediante el comando `podman mount` e interactuando directamente con las superposiciones fusionadas (*merged overlays*).

Para montar el sistema de archivos de un contenedor rootless en ejecución, primero debemos ejecutar el comando `podman unshare`, que permite a los usuarios ejecutar comandos dentro de un namespace de usuario modificado:

```bash
$ podman unshare
```

Este comando abre una shell de root en un nuevo namespace de usuario configurado con UID 0 y GID 0. Ahora es posible ejecutar el comando `podman mount` y obtener la ruta absoluta del punto de montaje:

```bash
# cd $(podman mount alpine_cp_test)
```

El comando anterior utiliza la expansión de shell para cambiar a la ruta de `MergedDir`, que, como sugiere el nombre, fusiona el contenido de `LowerDir` y `UpperDir` para proporcionar una vista unificada de las diferentes capas. A partir de ahora, es posible copiar archivos hacia y desde el sistema de archivos raíz del contenedor. Esta es una excelente manera de profundizar rápidamente en el sistema de archivos montado de un contenedor.

Los ejemplos anteriores se basaron en contenedores rootless, pero la misma lógica se aplica a los contenedores rootful. La práctica de copiar archivos y carpetas desde un contenedor es especialmente útil para solucionar problemas. La acción opuesta de copiarlos dentro de un contenedor en ejecución puede ser útil para actualizar y probar secretos o archivos de configuración. En ese caso, tenemos la opción de persistir esos cambios, como se describe en la siguiente subsección.

#### Persistencia de cambios con podman commit

Los ejemplos anteriores no son un método para personalizar permanentemente los contenedores en ejecución, ya que la naturaleza inmutable de los contenedores implica que las modificaciones persistentes deben pasar por la reconstrucción de la imagen.

Sin embargo, si necesitamos preservar los cambios y producir una nueva imagen sin iniciar una nueva compilación, el comando `podman commit` proporciona una forma de persistir los cambios en un contenedor dentro de una nueva imagen.

El concepto de commit es de primordial importancia en las compilaciones de imágenes de Docker y OCI. De hecho, podemos interpretar los diferentes pasos de un Dockerfile como una serie de confirmaciones (*commits*) aplicadas durante el proceso de compilación.

El siguiente ejemplo muestra cómo persistir un archivo copiado en un contenedor en ejecución y producir una nueva imagen. Supongamos que queremos actualizar la página `index.html` predeterminada de nuestro contenedor nginx:

```bash
$ echo "Hello World!" > /tmp/index.html
$ podman run --name custom_nginx -d -p \
8080:80 docker.io/library/nginx
$ podman cp /tmp/index.html \
custom_nginx:/usr/share/nginx/html/
```

Probemos los cambios aplicados:

```bash
$ curl localhost:8080
Hello World!
```

Ahora, queremos persistir el archivo `index.html` modificado en una nueva imagen, a partir del contenedor en ejecución con `podman commit`:

```bash
$ podman commit -p custom_nginx hello-world-nginx
```

El comando anterior persiste los cambios creando efectivamente una nueva capa de imagen que contiene los archivos y carpetas actualizados. Todas las capas, excepto la más nueva, la que se acaba de crear, se comparten con la imagen de nginx existente. No se utiliza almacenamiento adicional excepto para la nueva capa.

El contenedor anterior ahora se puede detener y eliminar de forma segura antes de probar la nueva imagen personalizada:

```bash
$ podman stop custom_nginx && podman rm custom_nginx
```

Probemos la nueva imagen personalizada e inspeccionemos el archivo `index.html` modificado:

```bash
$ podman run -d -p 8080:80 --name hello_world \
localhost/hello-world-nginx
$ curl localhost:8080
Hello World!
```

En esta sección, aprendimos cómo copiar archivos hacia y desde un contenedor en ejecución y cómo confirmar (*commit*) los cambios sobre la marcha produciendo una nueva imagen.

En la siguiente sección, aprenderemos cómo se adjunta el almacenamiento del host a un contenedor mediante la introducción del concepto de volúmenes y montajes bind (*bind mounts*).

---

### Adjuntar almacenamiento del host a un contenedor

Ya hemos hablado de la naturaleza inmutable de los contenedores. A partir de imágenes precompiladas, cuando ejecutamos un contenedor, instanciamos una capa de lectura/escritura sobre una pila de capas de solo lectura utilizando un enfoque COW.

Los contenedores son objetos efímeros basados en una imagen con estado. Esto implica que los contenedores no están pensados para almacenar datos dentro de ellos. Si un contenedor se bloquea o se elimina, todos los datos se perderían. Necesitamos una forma de almacenar datos en una ubicación separada que esté montada dentro del contenedor en ejecución, preservada cuando se elimine el contenedor y lista para ser reutilizada por un nuevo contenedor.

Hay otra advertencia importante que no debe olvidarse: los secretos y los archivos de configuración. Cuando creamos una imagen, podemos pasar todos los archivos y carpetas que necesitamos dentro de ella. Sin embargo, sellar secretos como certificados o claves dentro de una compilación no es una buena práctica. Si necesitamos, por ejemplo, rotar un certificado, tendremos que reconstruir toda la imagen desde cero. De la misma manera, cambiar un archivo de configuración que reside dentro de una imagen implica una nueva reconstrucción cada vez que cambiamos una configuración.

Por estas razones, las especificaciones OCI admiten volúmenes y montajes bind para administrar el almacenamiento adjunto a un contenedor. En las siguientes secciones, aprenderemos cómo funcionan los volúmenes y los montajes bind y cómo adjuntarlos a un contenedor.

#### Administración y conexión de montajes bind a un contenedor

Comencemos con los montajes bind (*bind mounts*), ya que aprovechan una característica nativa de Linux. Según las páginas de manual oficiales de Linux, un montaje bind es una forma de volver a montar una parte de la jerarquía del sistema de archivos en otro lugar. Esto significa que utilizando montajes bind, podemos replicar la vista de un directorio bajo otro punto de montaje en el host.

Antes de aprender cómo los contenedores usan los montajes bind, veamos un ejemplo básico en el que simplemente montamos el directorio `/etc` bajo el directorio `/mnt`:

```bash
$ sudo mount --bind /etc /mnt
```

Después de ejecutar este comando, veremos el contenido exacto de `/etc` bajo `/mnt`. Para desmontarlo, simplemente ejecuta el siguiente comando:

```bash
$ sudo umount /mnt
```

El mismo concepto se puede aplicar a los contenedores. Podman puede montar directorios del host dentro de un contenedor y ofrece opciones de CLI dedicadas para simplificar el proceso de montaje.

Podman ofrece dos opciones que se pueden utilizar para realizar montajes bind: `-v|--volume` y `--mount`. Cubramos esto con más detalle.

##### La opción -v|--volume

Esta opción utiliza un argumento compacto de un solo campo para definir el directorio del host de origen y el punto de montaje del contenedor con el patrón `/DIRECTORIO_HOST:/DIRECTORIO_CONTENEDOR`. El siguiente ejemplo monta el directorio `/host_files` en el punto de montaje `/mnt` dentro del contenedor:

```bash
$ podman run -v /host_files:/mnt docker.io/library/nginx
```

Es posible pasar argumentos adicionales para definir el comportamiento del montaje, por ejemplo, para montar el directorio del host como de solo lectura:

```bash
$ podman run -v /host_files:/mnt:ro \
docker.io/library/nginx
```

Otras opciones viables para montajes bind utilizando la opción `-v|--volume` se pueden encontrar en la página de manual del comando run (`man podman-run`).

##### La opción --mount

Esta opción es más detallada (*verbose*) ya que utiliza una sintaxis `clave=valor` para definir el origen y los destinos, así como el tipo de montaje y los argumentos adicionales. Esta opción acepta diferentes tipos de montaje (`bind` (montaje bind), `volume`, `tmpfs`, `image` y `devpts`) en el patrón `type=TIPO,source=DIR_HOST,destination=DIR_CONTENEDOR`. Las claves `source` y `destination` se pueden reemplazar por las más cortas `src` y `dst`, respectivamente. El ejemplo anterior se puede reescribir de la siguiente manera:

```bash
$ podman run \
--mount type=bind,src=/host_files,dst=/mnt \
docker.io/library/nginx
```

> [!IMPORTANT]
> Cuando se utilizan montajes bind en sistemas con SELinux habilitado (como Fedora, RHEL o CentOS), es posible que se le niegue el acceso al contenedor a los archivos del host debido a las etiquetas de seguridad. Para resolver esto, Podman te permite agregar una opción de reetiquetado al montaje:
> - `type=bind,src=...,dst=...,relabel=shared` (o `:z`): Esto le dice a Podman que el contenido se compartirá entre varios contenedores. Aplica una etiqueta de seguridad SELinux compartida a los archivos del host.
> - `type=bind,src=...,dst=...,relabel=private` (o `:Z`): Esto le dice a Podman que el contenido es privado y no compartido. Aplica una etiqueta de seguridad exclusiva y única a los archivos del host, asegurando que solo ese contenedor específico pueda acceder a ellos.
> 
> **Advertencia**: Ten cuidado al utilizar estos indicadores en directorios críticos del sistema (como `/etc` o `/home`), ya que volver a etiquetar estos archivos puede impedir que otros procesos del host o el sistema operativo accedan a ellos correctamente.

También podemos pasar una opción adicional agregando una coma extra, por ejemplo, para montar el directorio del host como de solo lectura:

```bash
$ podman run \
--mount type=bind,src=/host_files,dst=/mnt,ro=true \
docker.io/library/nginx
```

A pesar de ser muy sencillos de usar y comprender, los montajes bind tienen algunas limitaciones que podrían afectar el ciclo de vida del contenedor en algunos casos. Los archivos y directorios del host deben existir antes de ejecutar los contenedores, y los permisos deben configurarse en consecuencia para que se puedan leer o escribir. Otra advertencia importante a tener en cuenta es que un montaje bind siempre ofusca el punto de montaje subyacente en el contenedor si está poblado por archivos o directorios. Una alternativa útil a los montajes bind son los volúmenes, descritos en la siguiente subsección.

#### Administración y conexión de volúmenes a un contenedor

Un volumen es un directorio creado y administrado directamente por el motor de contenedores y montado en un punto de montaje dentro del contenedor. Ofrecen una excelente solución para persistir los datos generados por un contenedor; en lugar de administrar los datos y el directorio tú mismo, Podman lo hace por ti.

Los volúmenes se pueden administrar usando el comando `podman volume`, que se puede usar para listar, inspeccionar, crear y eliminar volúmenes en el sistema. Comencemos con un ejemplo básico, con un volumen creado automáticamente por Podman sobre la raíz de documentos (*document root*) de nginx:

```bash
$ podman run -d -p 8080:80 --name nginx_volume1 -v /usr/share/nginx/html docker.io/library/nginx
```

Esta vez, la opción `-v` tiene un argumento con un solo elemento: el directorio raíz del documento. En este caso, Podman crea automáticamente un volumen y lo monta mediante bind en el punto de montaje de destino.

Para demostrar que se ha creado un nuevo volumen, podemos inspeccionar el contenedor:

```bash
$ podman inspect nginx_volume1
[...omitted output...]
        "Mounts": [
            {
                "Type": "volume",
                "Name": "2ed93716b7ad73706df5c6f56bda262920accec59e7b6642d36f938e936d36d9",
                "Source": "/home/packt/.local/share/containers/storage/volumes/2ed93716b7ad73706df5c6f56bda262920accec59e7b6642d36f938e936d36d9/_data",
                "Destination": "/usr/share/nginx/html",
                "Driver": "local",
                "Mode": "",
                "Options": [
                    "nosuid",
                    "nodev",
                    "rbind"
                ],
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
[…omitted output]
```

En la sección `Mounts`, tenemos una lista de objetos montados en el contenedor. El único elemento es un objeto del tipo `volume`, con un UID generado como su nombre y un campo `Source` que representa su ruta en el host, mientras que el campo `Destination` es el punto de montaje dentro del contenedor.

Podemos verificar la existencia del volumen con el comando `podman volume ls`:

```bash
$ podman volume ls
DRIVER      VOLUME NAME
local       2ed93716b7ad73706df5c6f56bda262920accec59e7b6642d36f938e936d36d9
```

Podemos inspeccionar la carpeta local inspeccionando su ubicación y luego mirando dentro de la ruta de origen. Como podemos ver, encontraremos los archivos predeterminados en la raíz del documento del contenedor:

```bash
$ podman volume inspect $ID | grep Mountpoint
$ ls -al /home/packt/.local/share/containers/storage/volumes/2ed93716b7ad73706df5c6f56bda262920accec59e7b6642d36f938e936d36d9/_data
total 16
drwxr-xr-x. 2 packt packt 4096 Sep  9 20:26 .
drwx------. 3 packt packt 4096 Oct 16 22:41 ..
-rw-r--r--. 1 packt packt  497 Sep  7 17:21 50x.html
-rw-r--r--. 1 packt packt  615 Sep  7 17:21 index.html
```

Esto demostró que cuando se crea un volumen vacío, se completa con el contenido del punto de montaje de destino. Cuando un contenedor se detiene, el volumen se conserva junto con todos los datos y se puede reutilizar cuando otro contenedor reinicie el contenedor. Al ingresar al mundo de los contenedores, notarás que las imágenes de contenedores populares ya usan volúmenes de forma predeterminada para almacenar su configuración y datos de manera estructurada.

El ejemplo anterior muestra un volumen con un UID generado, pero es posible elegir el nombre del volumen adjunto, como en el siguiente ejemplo:

```bash
$ podman run -d -p 8080:80 --name nginx_volume2 -v nginx_vol:/usr/share/nginx/html docker.io/library/nginx
```

En el ejemplo anterior, Podman crea un nuevo volumen llamado `nginx_vol` y lo almacena en el directorio de volúmenes predeterminado. Cuando se crea un volumen con nombre, Podman no necesita generar un UID.

El directorio de volúmenes predeterminado tiene rutas diferentes para contenedores rootless y rootful:

- Para contenedores rootless, la ruta de almacenamiento de volumen predeterminada es `<USER_HOME>/.local/share/containers/storage/volumes`
- Para contenedores rootful, la ruta de almacenamiento de volumen predeterminada es `/var/lib/containers/storage/volumes`

Alternativamente, Podman ofrece los comandos `podman volume mount` y `podman volume unmount` para acceder fácilmente a los archivos de un solo volumen.

Los volúmenes creados en esas rutas persisten después de que se destruye el contenedor y pueden ser reutilizados por otros contenedores.

Para eliminar manualmente un volumen, utiliza el comando `podman volume rm` si ningún contenedor está utilizando ese volumen:

```bash
$ podman volume rm nginx_vol
```

En caso de que haya contenedores usando un determinado volumen, puedes usar el siguiente comando, que elimina el volumen y cualquier contenedor que lo use:

```bash
$ podman volume rm --force nginx_vol
```

Cuando se trabaja con varios volúmenes, el comando `podman volume prune` elimina todos los volúmenes no utilizados. El siguiente ejemplo elimina todos los volúmenes en el almacenamiento de volúmenes predeterminado del usuario (el utilizado por contenedores rootless):

```bash
$ podman volume prune
```

Los mismos comandos funcionan para root y rootless.

> [!IMPORTANT]
> No olvides monitorear los volúmenes que se acumulan en el host, ya que consumen espacio en disco que podría recuperarse, y elimina periódicamente los volúmenes no utilizados para evitar saturar el almacenamiento del host. Además, los contenedores basados en ciertas imágenes pueden crear automáticamente volúmenes que no siempre se eliminan cuando se elimina el contenedor, lo que genera una acumulación inesperada de volúmenes.

Los usuarios también pueden crear y poblar volúmenes de forma preliminar antes de ejecutar el contenedor. El siguiente ejemplo utiliza el comando `podman volume create` para crear el volumen montado en la raíz de documentos de nginx y luego lo puebla con un archivo `index.html` de prueba:

```bash
$ podman volume create custom_nginx
$ podman unshare
# podman volume mount custom_nginx
/home/packt/.local/share/containers/storage/volumes/custom_nginx/_data
# echo "Hello World!" >> /home/packt/.local/share/containers/storage/volumes/custom_nginx/_data/index.html
```

Ahora podemos ejecutar un nuevo contenedor nginx utilizando el volumen prepoblado:

```bash
$ podman run -d -p 8080:80 --name nginx_volume3 -v custom_nginx:/usr/share/nginx/html docker.io/library/nginx
```

La prueba HTTP muestra el contenido actualizado:

```bash
$ curl localhost:8080
Hello World!
```

Esta vez, el volumen, que no estaba vacío al principio, ofuscó el directorio de destino del contenedor con su contenido.

Al igual que con los montajes bind, podemos elegir libremente entre las opciones `-v|--volume` y `--mount`. Veamos cómo.

#### Montaje de volúmenes con la opción --mount

El siguiente ejemplo ejecuta un contenedor nginx utilizando el indicador `--mount`:

```bash
$ podman run -d -p 8080:80 --name nginx_volume4 --mount type=volume,src=custom_nginx,dst=/usr/share/nginx/html docker.io/library/nginx
```

Si bien la opción `-v|--volume` es compacta y ampliamente adoptada, la ventaja de la opción `--mount` es una sintaxis más clara y expresiva, junto con una declaración exacta del tipo de montaje; sin embargo, la opción `-v|--volume` se utiliza estadísticamente más debido a su sintaxis más simple.

#### Controladores de volumen (*Volume drivers*)

Los ejemplos de volúmenes anteriores se basan todos en el mismo controlador de volumen `local`, que se utiliza para administrar volúmenes en el sistema de archivos local del host. Se pueden configurar controladores de volumen adicionales en el archivo `/usr/share/containers/containers.conf` en la sección `[engine.volume_plugins]` pasando el nombre del complemento seguido de la ruta del archivo o socket. Ciertos proveedores proporcionan complementos de volumen que se pueden utilizar con este sistema, pero generalmente no es necesario a menos que tengas una solución de almacenamiento que ofrezca dicho complemento.

El controlador de volumen local también se puede utilizar para montar recursos compartidos NFS en el host que ejecuta el contenedor. El siguiente ejemplo muestra cómo crear un volumen respaldado por un recurso compartido NFS y montarlo dentro de un contenedor MongoDB en su directorio `/data/db`:

```bash
$ sudo podman volume create --driver local --opt type=nfs --opt o=addr=nfs-host.example.com,rw,context="system_u:object_r:container_file_t:s0" --opt device=:/opt/nfs-export nfs-volume
$ sudo podman run -d -v nfs-volume:/data/db docker.io/library/mongo
```

Un requisito previo del ejemplo anterior es la configuración preliminar del servidor NFS, que debe ser accesible por el host que ejecuta el contenedor. Es importante tener en cuenta que esta característica no está disponible para contenedores rootless.

#### Volúmenes en compilaciones (*Volumes in builds*)

Los volúmenes se pueden predefinir durante el proceso de compilación de la imagen. Esto permite a los mantenedores de imágenes definir qué directorios de contenedores se adjuntarán automáticamente a los volúmenes. Para comprender este concepto, inspeccionemos este Dockerfile mínimo:

```dockerfile
FROM docker.io/library/nginx:latest
VOLUME /usr/share/nginx/html
```

El único cambio realizado en la imagen `docker.io/library/nginx` es una directiva `VOLUME`, que define qué directorio se debe montar externamente como un volumen anónimo en el host. Esta declaración es simplemente un metadato y el volumen se creará solo en tiempo de ejecución cuando se inicie un contenedor a partir de esta imagen.

Si compilamos la imagen y ejecutamos un contenedor basado en el Dockerfile de ejemplo, podemos ver un volumen anónimo creado automáticamente:

```bash
$ podman build -t my_nginx .
$ podman run -d --name volumes_from_build my_nginx
$ podman inspect volumes_from_build --format "{{ .Mounts }}"
[{volume 4d6ac7edcb4f01add205523b7733d61ae4a5772786eacca68e4972b20fd1180c /home/packt/.local/share/containers/storage/volumes/4d6ac7edcb4f01add205523b7733d61ae4a5772786eacca68e4972b20fd1180c/_data /usr/share/nginx/html local [nodev exec nosuid rbind] true rprivate}]
```

Sin una opción explícita de creación de volumen, Podman ya ha creado y montado el volumen del contenedor. Esta definición automática de volumen en el momento de la compilación es una práctica común en todos los contenedores que se espera que persistan datos, como las bases de datos. Ten en cuenta que, en la mayoría de los casos, los volúmenes creados con el contenedor y a los que no se les asigna un nombre, como este, se eliminan automáticamente junto con el contenedor que los creó.

Por ejemplo, inspeccionemos la imagen oficial de MongoDB: la imagen `docker.io/library/mongo` ya está configurada para crear dos volúmenes, uno para `/data/configdb` y otro para `/data/db`. El mismo comportamiento se puede identificar en las bases de datos más comunes, incluidas PostgreSQL, MariaDB o MySQL.

Es posible definir cómo se deben montar los volúmenes anónimos predefinidos cuando se inicia un contenedor. De forma predeterminada, los volúmenes se crean y se montan mediante bind en el contenedor (denominado opción `bind`). Sin embargo, también puedes optar por montarlos como `tmpfs` o ignorarlos por completo utilizando la opción `--image-volume`. El siguiente ejemplo inicia un contenedor MongoDB con sus volúmenes predeterminados montados como `tmpfs`:

```bash
$ podman run -d --image-volume tmpfs docker.io/library/mongo
```

En el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), cubriremos el proceso de compilación con mayor detalle. Ahora cerramos esta subsección con un ejemplo de cómo montar volúmenes en múltiples contenedores.

#### Montaje de volúmenes entre contenedores (*Mounting volumes across containers*)

Una de las mayores ventajas de los volúmenes es su flexibilidad. Por ejemplo, un contenedor puede montar volúmenes desde un contenedor que ya se está ejecutando para compartir los mismos datos. Para lograr este resultado, podemos usar la opción `--volumes-from`. El siguiente ejemplo inicia un contenedor MongoDB y luego monta sus volúmenes en un contenedor Fedora:

```bash
$ podman run -d --name mongodb01 docker.io/library/mongo
$ podman run -it --volumes-from=mongodb01 docker.io/library/fedora
```

El segundo contenedor abre una shell interactiva de root que podemos usar para inspeccionar el contenido del sistema de archivos:

```bash
[root@c10420016687 /]# ls -al /data
total 20
drwxr-xr-t.  4 root root 4096 Oct 17 15:36 .
dr-xr-xr-x. 19 root root 4096 Oct 17 15:36 ..
drwxr-xr-x.  2  999  999 4096 Sep 20 22:20 configdb
drwxr-xr-x.  4  999  999 4096 Oct 17 15:36 db
```

Como era de esperar, podemos encontrar los volúmenes de MongoDB montados en el contenedor Fedora. Si detenemos e incluso eliminamos el contenedor `mongodb01`, los volúmenes permanecen activos y montados dentro del contenedor Fedora. Podemos lograr fácilmente el mismo resultado utilizando el comando `podman inspect`.

Por defecto, Podman monta los volúmenes con los mismos permisos con los que están montados en el contenedor de origen. Es posible cambiarlos agregando la opción `rw` o `ro` separada por comas. El siguiente ejemplo adjunta el volumen del contenedor MongoDB como un volumen de solo lectura:

```bash
$ podman run -it --volumes-from=mongodb01,ro docker.io/library/fedora
```

Hasta ahora, hemos visto casos de uso básicos sin una segregación específica entre contenedores o recursos montados. Si el host tiene SELinux habilitado y en modo `enforcing`, se deben aplicar algunas consideraciones adicionales.

#### Consideraciones de SELinux para montajes

SELinux aplica etiquetas de forma recursiva a archivos y directorios para definir su contexto. Esas etiquetas generalmente se almacenan como atributos extendidos del sistema de archivos. SELinux utiliza contextos para administrar políticas y definir qué procesos pueden acceder a un recurso específico.

El comando `ls` se utiliza para ver el contexto de tipo de un recurso:

```bash
$ ls -alZ /etc/passwd
-rw-r--r--. 1 root root system_u:object_r:passwd_file_t:s0 2965 Jul 28 21:00 /etc/passwd
```

En el ejemplo anterior, la etiqueta `passwd_file_t` define el contexto de tipo del archivo `/etc/passwd`. Según el contexto de tipo, un programa puede o no acceder a un archivo mientras SELinux se ejecuta en modo `enforcing`.

Los procesos también tienen su contexto de tipo: los contenedores se ejecutan con la etiqueta `container_t` y tienen acceso de lectura/escritura a archivos y directorios etiquetados con el contexto de tipo `container_file_t`, y acceso de lectura/ejecución a recursos etiquetados con `container_share_t`.

Otros directorios de host accesibles de forma predeterminada son `/etc` como solo lectura y `/usr` como lectura/ejecución. Además, los recursos bajo `/var/lib/containers/overlay/` están etiquetados como `container_share_t`.

¿Qué sucede si intentamos montar un directorio que no está correctamente etiquetado?

Podman aún ejecuta el contenedor sin quejarse del etiquetado incorrecto, pero el proceso que se ejecuta dentro de los contenedores, que están etiquetados con el tipo de contexto `container_t`, no podrá acceder al directorio o archivo montado. El siguiente ejemplo intenta montar una raíz de documentos personalizada para un contenedor nginx sin respetar las restricciones de etiquetado:

```bash
$ mkdir ~/custom_docroot
$ echo "Hello World!" > ~/custom_docroot/index.html
$ podman run -d \
--name custom_nginx \
-p 8080:80 \
-v ~/custom_docroot:/usr/share/nginx/html \
docker.io/library/nginx
```

Aparentemente, todo salió bien: el contenedor se inició correctamente y los procesos dentro de él se están ejecutando, pero si intentamos contactar con el servidor NGINX, vemos el error:

```bash
$ curl localhost:8080
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.21.3</center>
</body>
</html>
```

`403 Forbidden` muestra que el proceso de nginx no puede acceder a la página `index.html`. Para corregir este error, tenemos dos opciones: poner SELinux en modo permisivo (*permissive*) o volver a etiquetar los recursos montados. Al poner SELinux en modo permisivo, continúa rastreando las infracciones sin bloquearlas. De todos modos, esta no es una buena práctica y debe usarse solo cuando no podamos solucionar correctamente los problemas de acceso y necesitemos dejar a SELinux fuera de la ecuación.

El siguiente comando establece SELinux en modo permisivo:

```bash
$ sudo setenforce 0
```

> [!NOTE]
> El modo permisivo no equivale a deshabilitar SELinux por completo. Al trabajar en este modo, SELinux aún registra las denegaciones de la caché de vectores de acceso (AVC, *Access Vector Cache*) sin bloquearlas. Los administradores del sistema pueden cambiar inmediatamente entre el modo permisivo y el modo de aplicación (*enforcing*) sin reiniciar. Deshabilitarlo, por otro lado, implica un reinicio completo del sistema.
> Alternativamente, puedes deshabilitar SELinux para un solo contenedor usando `--security-opt label=disable`.

La segunda opción, y preferida, es simplemente volver a etiquetar los recursos que necesitamos montar. Para lograr este resultado, podríamos usar herramientas de línea de comandos de SELinux. Como atajo, Podman ofrece una forma más sencilla: los sufijos `:z` y `:Z` aplicados a los argumentos de montaje del volumen. La diferencia entre los dos sufijos es sutil:

- El sufijo `:z` le indica a Podman que vuelva a etiquetar los recursos montados para permitir que todos los contenedores lean y escriban en el almacenamiento. Funciona tanto con volúmenes como con montajes bind.
- El sufijo `:Z` le indica a Podman que vuelva a etiquetar los recursos montados para permitir que solo el contenedor actual lea y escriba en el almacenamiento de forma exclusiva. Esto también funciona tanto con volúmenes como con montajes bind.

Ten cuidado al utilizar estos indicadores en directorios críticos del sistema (como `/etc` o `/home`), ya que volver a etiquetar estos archivos puede impedir que otros procesos del host o el sistema operativo accedan a ellos correctamente.

Para probar la diferencia, intentemos ejecutar el contenedor nuevamente con el sufijo `:z` y veamos qué sucede:

```bash
$ podman run -d \
--name custom_nginx \
-p 8080:80 \
-v ~/custom_docroot:/usr/share/nginx/html:z \
docker.io/library/nginx
```

Ahora, las llamadas HTTP devuelven los resultados esperados ya que el proceso pudo acceder al archivo `index.html` sin ser bloqueado por SELinux:

```bash
$ curl localhost:8080
Hello World!
```

Veamos el contexto de archivo de SELinux aplicado automáticamente al directorio montado:

```bash
$ ls -alZ ~/custom_docroot
total 20
drwxrwxr-x.  2 packt packt system_u:object_r:container_file_t:s0   4096 Oct 16 15:53 .
drwxrwxr-x. 74 packt packt unconfined_u:object_r:user_home_dir_t:s0 12288 Oct 16 16:32 ..
-rw-rw-r--.  1 packt packt system_u:object_r:container_file_t:s0     13 Oct 16 15:53 index.html
```

Centrémonos en la etiqueta `system_u:object_r:container_file_t:s0`. El campo final `s0` es un nivel de sensibilidad de Seguridad Multinivel (MLS, *Multi-Level Security*), lo que significa que todos los procesos con el mismo nivel de sensibilidad tendrán acceso de lectura/escritura al recurso. Por lo tanto, otros contenedores que se ejecutan con el nivel de sensibilidad `s0` podrán montar el recurso con privilegios de acceso de lectura/escritura. Esto también representa un problema de seguridad ya que un contenedor malicioso en el mismo host podría atacar a otros contenedores robando o sobrescribiendo datos.

La solución a este problema se llama Seguridad Multicategoría (MCS, *Multi-Category Security*). SELinux utiliza MCS para configurar categorías adicionales, que son etiquetas de texto sin formato aplicadas a los recursos junto con las otras etiquetas de SELinux. Los objetos etiquetados con MCS son accesibles solo para los procesos que tienen asignadas las mismas categorías.

Cuando se inicia un contenedor, los procesos dentro de él se etiquetan con categorías MCS, siguiendo el patrón `cXXX,cYYY`, donde `XXX` e `YYY` son números enteros seleccionados aleatoriamente.

Podman aplica automáticamente categorías MCS a los recursos montados cuando se pasa `Z` (mayúscula). Para probar este comportamiento, ejecutemos nuevamente el contenedor nginx con el sufijo `:Z`:

```bash
$ podman run -d \
--name custom_nginx \
-p 8080:80 \
-v ~/custom_docroot:/usr/share/nginx/html:Z \
docker.io/library/nginx
```

Podemos ver inmediatamente que la carpeta montada se ha vuelto a etiquetar con categorías MCS:

```bash
$ ls -alZ ~/custom_docroot
total 20
drwxrwxr-x.  2 packt packt system_u:object_r:container_file_t:s0:c16,c898  4096 Oct 16 15:53 .
drwxrwxr-x. 74 packt packt unconfined_u:object_r:user_home_dir_t:s0       12288 Oct 16 21:12 ..
-rw-rw-r--.  1 packt packt system_u:object_r:container_file_t:s0:c16,c898    13 Oct 16 15:53 index.html
```

Una simple prueba devolverá el texto `Hello World!` esperado, lo que demuestra que los procesos dentro del contenedor pueden acceder a los recursos de destino:

```bash
$ curl localhost:8080
Hello World!
```

¿Qué sucede si ejecutamos un segundo contenedor con el mismo enfoque, aplicando `:Z` nuevamente al mismo montaje bind?

```bash
$ podman run -d \
--name custom_nginx2 \
-p 8081:80 \
-v ~/custom_docroot:/usr/share/nginx/html:Z \
docker.io/library/nginx
```

Esta vez, ejecutamos la prueba HTTP en el puerto 8081 y el GET HTTP sigue funcionando correctamente:

```bash
$ curl localhost:8081
Hello World!
```

Sin embargo, si probamos una vez más el contenedor mapeado al puerto 8080, obtendremos un mensaje inesperado `403 Forbidden`:

```bash
$ curl localhost:8080
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.21.3</center>
</body>
</html>
```

Como era de esperar, el segundo contenedor se ejecutó con el sufijo `:Z` y volvió a etiquetar el directorio con un nuevo par de categorías MCS, haciendo que el primer contenedor no pudiera acceder al contenido previamente disponible.

> [!IMPORTANT]
> Los ejemplos anteriores se realizaron con montajes bind, pero siguen aplicándose a los volúmenes de la misma manera. Utiliza estas técnicas con precaución para evitar cambios de etiquetas no deseados en directorios del sistema o del directorio home montados por bind. Además de eso, no debes usar `:z` y `:Z` con volúmenes; Podman maneja automáticamente su etiquetado SELinux.

En esta subsección, demostramos el poder de SELinux para administrar contenedores y el aislamiento de recursos. Concluyamos este capítulo con una descripción general de otros tipos de almacenamiento que se pueden adjuntar a los contenedores.

---

### Conectar otros tipos de almacenamiento a un contenedor

Junto con los montajes bind y los volúmenes, es posible adjuntar otros tipos de almacenamiento a los contenedores, más específicamente, de los tipos `tmpfs`, `image` y `devpts`.

#### Conexión de almacenamiento tmpfs (*Attaching tmpfs storage*)

A veces, necesitamos adjuntar almacenamiento a contenedores que no están diseñados para ser persistentes (por ejemplo, para uso de caché). El uso de volúmenes o montajes bind saturaría el disco local del host (o cualquier otro backend si se utilizan diferentes controladores de almacenamiento). En esos casos particulares, podemos usar un volumen `tmpfs`.

`tmpfs` es un sistema de archivos de memoria virtual, lo que significa que todo su contenido se crea dentro de la memoria virtual del host. Un beneficio de `tmpfs` es que proporciona una E/S más rápida ya que todas las operaciones de lectura/escritura ocurren principalmente en la memoria RAM.

Para adjuntar un volumen `tmpfs` a un contenedor, podemos usar la opción `--mount` o la opción `--tmpfs`.

La bandera `--mount` tiene la gran ventaja de ser más descriptiva y expresiva con respecto al tipo de almacenamiento, origen, destino y opciones de montaje adicionales. El siguiente ejemplo ejecuta un contenedor httpd con un volumen `tmpfs` adjunto al contenedor:

```bash
$ podman run -d -p 8080:80 \
--name tmpfs_example1 \
--mount type=tmpfs,tmpfs-size=512M,destination=/tmp \
docker.io/library/httpd
```

El comando anterior crea un volumen `tmpfs` de 512 MB y lo monta en la carpeta `/tmp` del contenedor. Podemos probar la correcta creación del montaje ejecutando el comando mount dentro del contenedor:

```bash
$ podman exec -it tmpfs_example1 mount | grep '\/tmp'
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,relatime,context="system_u:object_r:container_file_t:s0:c375,c804",size=524288k,uid=1000,gid=1000,inode64
```

Esto demuestra que el sistema de archivos `tmpfs` se ha montado correctamente dentro del contenedor. Detener el contenedor descartará automáticamente `tmpfs`:

```bash
$ podman stop tmpfs_example1
```

El siguiente ejemplo monta un volumen `tmpfs` utilizando la opción `--tmpfs`:

```bash
$ podman run -d -p 8080:80 \
--name tmpfs_example2 \
--tmpfs /tmp:rw,size=524288k,mode=1777 \
docker.io/library/httpd
```

Este ejemplo proporciona los mismos resultados que el anterior: un contenedor en ejecución con un volumen `tmpfs` de 512 MB montado en el directorio `/tmp` en modo lectura/escritura y permisos 1777.

De forma predeterminada, `tmpfs` se monta dentro del contenedor con las siguientes opciones de montaje: `rw`, `noexec`, `nosuid` y `nodev`.

Otra característica interesante es el etiquetado MCS automático de SELinux. Esto proporciona una segregación automática del sistema de archivos y evita que cualquier otro contenedor acceda a los datos en la memoria.

#### Conexión de imágenes (*Attaching images*)

Las imágenes OCI son la base que proporciona capas y metadatos para iniciar contenedores, pero también se pueden adjuntar al sistema de archivos de un contenedor en tiempo de ejecución. Esto puede resultar útil para solucionar problemas, para adjuntar binarios que están disponibles en una imagen externa o para realizar análisis de seguridad en imágenes. Cuando se monta una imagen OCI dentro de un contenedor, se crea una superposición adicional. Esto implica que incluso cuando la imagen se monta con permisos de lectura/escritura, los usuarios nunca alteran la imagen original sino solo la superposición superior.

El siguiente ejemplo monta una imagen de busybox con permisos de lectura/escritura dentro de un contenedor Alpine:

```bash
$ podman run -it \
--mount type=image,src=docker.io/library/busybox,dst=/mnt,rw=true \
alpine
```

> [!IMPORTANT]
> Cualquier imagen que pretendas montar debe estar disponible localmente en el host primero. A diferencia del comando estándar `podman run`, que busca automáticamente una imagen base faltante, Podman requiere que las imágenes montables se descarguen de antemano. Ejecutar un comando `podman pull` preliminar garantiza que la imagen esté almacenada en caché y lista para usar.

#### Conexión de devpts (*Attaching devpts*)

Esta opción es útil para conectar un pseudoesclavo de terminal (PTS, *pseudo terminal slave*) a un contenedor. Esta característica se introdujo en Podman 2.1.0 para admitir contenedores que necesitan montar `/dev/` desde el host en el contenedor, mientras continúan creando una terminal. El pseudosistema de archivos `/dev` del host permite a los contenedores obtener acceso directo a los dispositivos físicos o virtuales de la máquina.

Para crear un contenedor con el sistema de archivos `/dev` y un dispositivo `devpts` adjunto, ejecuta el siguiente comando:

```bash
$ sudo podman run -it \
-v /dev/:/dev:rslave \
--mount type=devpts,destination=/dev/pts \
docker.io/library/fedora
```

Para comprobar el resultado de la opción de montaje, necesitamos una herramienta adicional dentro del contenedor. Por esta razón, podemos instalarla con el siguiente comando:

```bash
[root@034c8a61a4fc /]# dnf install -y toolbox
```

El contenedor resultante tiene un dispositivo `devpts` adicional, no aislado, montado en `/dev/pts`:

```bash
# mount | grep '\/dev\/pts'
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,context="system_u:object_r:container_file_t:s0:c299,c741",gid=5,mode=620,ptmxmode=666)
```

La salida anterior se extrajo ejecutando el comando `mount` dentro del contenedor.

---

### Resumen

En este capítulo, hemos completado un recorrido sobre el almacenamiento de contenedores y las características que ofrece Podman para manipularlo. El material de este capítulo es crucial para comprender cómo Podman gestiona los datos tanto efímeros como persistentes y proporciona mejores prácticas a los usuarios para manipular sus datos.

En la primera sección, aprendimos por qué es importante el almacenamiento de contenedores y cómo debe administrarse correctamente tanto en entornos de un solo host como en entornos orquestados de múltiples hosts.

En la segunda sección, profundizamos en las características del almacenamiento de contenedores y los controladores de almacenamiento, con un enfoque especial en `overlayfs`.

En la tercera sección, aprendimos cómo copiar archivos hacia y desde un contenedor. También vimos cómo los cambios podían confirmarse (*commit*) en una nueva imagen.

La cuarta sección describió los diferentes escenarios posibles de almacenamiento adjunto a un contenedor, cubriendo montajes bind, volúmenes, `tmpfs`, imágenes y `devpts`. Esta sección también fue ideal para analizar la interacción de SELinux en la gestión del almacenamiento y ver cómo podemos usarlo para aislar los recursos de almacenamiento entre contenedores en el mismo host.

En el próximo capítulo, aprenderemos sobre un tema muy importante tanto para los desarrolladores como para los equipos de operaciones, que es cómo compilar imágenes OCI tanto con Podman como con Buildah, una herramienta de compilación de imágenes avanzada y especializada.

---

### Lecturas adicionales

Consulta los siguientes recursos para obtener más información:

- Página del proyecto Containers/storage: [https://github.com/containers/storage](https://github.com/containers/storage)
- Etiquetado de contenedores: [https://danwalsh.livejournal.com/81269.html](https://danwalsh.livejournal.com/81269.html)
- Por qué deberías usar Multi-Category Security para tus contenedores Linux: [https://www.redhat.com/en/blog/why-you-should-be-using-multi-category-security-your-linux-containers](https://www.redhat.com/en/blog/why-you-should-be-using-multi-category-security-your-linux-containers)
- Udica: Generar políticas de SELinux: [https://github.com/containers/udica](https://github.com/containers/udica)
- Código fuente de Overlay: [https://github.com/containers/storage/blob/main/drivers/overlay/overlay.go](https://github.com/containers/storage/blob/main/drivers/overlay/overlay.go)
