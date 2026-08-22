# Parte 2: Construyendo tu Contenedor desde Cero con Buildah

## Capítulo 9: Publicación de Imágenes en un Registro de Contenedores

En el capítulo anterior, repasamos el concepto fundamental de la imagen base del contenedor. Como vimos, es realmente importante elegir sabiamente la imagen base para nuestros contenedores, utilizando imágenes de contenedores oficiales provenientes de registros de contenedores y comunidades de desarrollo de confianza.

Pero una vez que elegimos la imagen base preferida y luego construimos nuestra imagen de contenedor final, necesitamos una forma de distribuir nuestro trabajo a los diversos hosts de destino en los que planeamos ejecutarlo.

La mejor opción para distribuir una imagen de contenedor es publicarla (*push*) en un registro de contenedores y, después de eso, permitir que todos los hosts de destino descarguen (*pull*) la imagen del contenedor y la ejecuten.

Por esta razón, en este capítulo, vamos a cubrir los siguientes temas principales:

- ¿Qué es un registro de contenedores?
- Registros de contenedores basados en la nube y locales (*on-premises*)
- Administración de imágenes de contenedores con Skopeo
- Ejecución de un registro de contenedores local

---

### Requisitos técnicos

Antes de continuar con el capítulo y sus ejemplos, se requiere una máquina con una instalación de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Una buena comprensión de los temas tratados en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, y el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/8), *Elección de la Imagen Base del Contenedor*, es útil para asimilar fácilmente los conceptos relativos a los registros de contenedores.

---

### ¿Qué es un registro de contenedores?

Un registro de contenedores es una colección de repositorios de imágenes de contenedores, utilizada conjuntamente con sistemas que necesitan descargar (*pull*) y ejecutar imágenes de contenedores de forma dinámica.

Las características principales disponibles en un registro de contenedores son las siguientes:

- Administración de repositorios
- Publicación de imágenes de contenedores (*push*)
- Administración de etiquetas (*tag*)
- Descarga de imágenes de contenedores (*pull*)
- Administración de autenticación

Veamos cada característica en detalle en las siguientes secciones.

#### Administración de repositorios

Una de las características más importantes de los registros de contenedores es la gestión de imágenes de contenedores a través de repositorios. Según la implementación del registro de contenedores que elijamos, seguramente encontraremos una interfaz web o una interfaz de línea de comandos que nos permitirá gestionar la creación de una especie de carpeta que actuará como repositorio para nuestras imágenes de contenedores.

De acuerdo con la Especificación de Distribución de la Open Container Initiative (OCI) [1], las imágenes de contenedores se organizan en un repositorio identificado por un nombre. El nombre de un repositorio generalmente se compone del nombre de usuario/organización y el nombre de la imagen del contenedor de esta manera: `miorganizacion/miimagendecontenedor`. Debe respetar la siguiente comprobación de expresión regular:

```text
[a-z0-9]+([._-][a-z0-9]+)*(/[a-z0-9]+([._-][a-z0-9]+)*)*
```

> [!NOTE]
> Una expresión regular (*regex*) es un patrón de búsqueda definido por una secuencia de caracteres. Esta definición de patrón aprovecha varias notaciones que permiten al usuario definir en detalle la palabra clave, la línea o las múltiples líneas de destino que se desean encontrar en un documento de texto.

Una vez que hayamos creado un repositorio en nuestro registro de contenedores, deberíamos poder comenzar a publicar (*push*), descargar (*pull*) y administrar diferentes versiones (identificadas mediante una etiqueta o *tag*) para nuestras imágenes de contenedores.

#### Publicación de imágenes de contenedores

El acto de publicar imágenes de contenedores en un registro de contenedores es gestionado por la herramienta de contenedores que estemos utilizando, la cual respeta la Especificación de Distribución OCI.

En este proceso, los blobs, que son la forma binaria del contenido, se cargan primero y, por lo general, al final se carga el manifiesto. Este orden no es estricto ni obligatorio según la especificación, pero un registro puede rechazar un manifiesto que haga referencia a blobs que desconoce.

Al utilizar una herramienta de administración de contenedores para publicar una imagen de contenedor en un registro, debemos especificar nuevamente el nombre del repositorio en el formato mostrado anteriormente y la etiqueta de la imagen de contenedor que deseamos cargar.

#### Administración de etiquetas

Como se presentó en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, las imágenes de contenedores se identifican mediante un nombre y una etiqueta (*tag*). Gracias al mecanismo de etiquetas, podemos almacenar varias versiones diferentes de las imágenes de contenedores en la caché local de un sistema o en un registro de contenedores.

El registro de contenedores debe poder exponer la función de descubrimiento de contenido, proporcionando la lista de etiquetas de las imágenes de contenedores al cliente que lo solicite. Esta función puede brindar la oportunidad a los usuarios del registro de contenedores de elegir la imagen de contenedor adecuada para descargar y ejecutar en los sistemas de destino.

#### Descarga de imágenes de contenedores

En el proceso de descarga (*pull*) de imágenes de contenedores, el cliente primero solicita el manifiesto para identificar los blobs, que son la forma binaria del contenido, que se deben descargar para obtener la imagen final del contenedor. El orden es estricto porque, sin descargar y analizar el archivo de manifiesto de la imagen del contenedor, el cliente no podría saber qué datos binarios tiene que solicitar al registro.

Al utilizar una herramienta de administración de contenedores para descargar una imagen de contenedor de un registro, debemos utilizar el nombre completo (*fully-qualified name*) de la imagen, que incluye tanto la etiqueta como el repositorio.

#### Administración de autenticación

Todas las operaciones anteriores pueden requerir autenticación. En muchos casos, los registros públicos de contenedores pueden permitir la descarga anónima y el descubrimiento de contenido, pero para publicar imágenes de contenedores, requieren una autenticación válida.

Según el registro de contenedores elegido, podemos encontrar funciones básicas o avanzadas para autenticarnos en un registro de contenedores. Esto permite que nuestro cliente almacene un token y luego lo use para cada operación que lo requiera.

Esto concluye nuestra breve profundización en la teoría del registro de contenedores. Si deseas obtener más información sobre la Especificación de Distribución OCI, puedes consultar la URL [1] disponible al final de este capítulo en la sección *Lecturas adicionales*.

> [!TIP]
> La Especificación de Distribución OCI también define un conjunto de pruebas de conformidad que cualquiera puede ejecutar contra un registro de contenedores para verificar si esa implementación en particular respeta todas las reglas definidas en la especificación: [https://github.com/opencontainers/distribution-spec/tree/main/conformance](https://github.com/opencontainers/distribution-spec/tree/main/conformance).

Las diversas implementaciones de un registro de contenedores disponibles en la web, además de las funciones básicas que describimos en capítulos anteriores, ofrecen características adicionales que descubriremos en la siguiente sección.

---

### Registros de contenedores basados en la nube y locales (*on-premises*)

Como presentamos en las secciones anteriores, OCI definió un estándar al que adherirse para los registros de contenedores. Esta iniciativa permitió la aparición de muchos otros registros de contenedores además del Docker Registry inicial y su servicio en línea, Docker Hub.

Podemos agrupar los registros de contenedores disponibles en dos categorías principales:

- Registros de contenedores basados en la nube
- Registros de contenedores locales (*on-premises*)

Veamos estas dos categorías en detalle en las siguientes subsecciones.

#### Registros de contenedores locales (*on-premises*)

Los registros de contenedores locales a menudo se utilizan para crear un repositorio privado para fines empresariales. Los principales casos de uso incluyen los siguientes:

- Distribuir imágenes en una red privada o aislada
- Implementar una nueva imagen de contenedor a gran escala en varias máquinas
- Mantener cualquier dato confidencial en nuestro propio centro de datos
- Mejorar la velocidad de descarga y publicación de imágenes utilizando una red interna

Por supuesto, ejecutar un registro local requiere varias habilidades para garantizar la disponibilidad, la supervisión, el registro y la seguridad.

Esta es una lista no exhaustiva de los registros de contenedores disponibles que podemos instalar de forma local:

- **Docker Registry**: El proyecto de Docker, que actualmente se encuentra en la versión 2, proporciona todas las características básicas descritas en las secciones anteriores, y aprenderemos cómo ejecutarlo en la última sección de este capítulo, *Ejecución de un registro de contenedores local*.
- **Harbor**: Este es un proyecto de código abierto de VMware que proporciona alta disponibilidad, auditoría de imágenes e integración con sistemas de autenticación.
- **GitLab Container Registry**: Está fuertemente integrado con el producto GitLab, por lo que requiere una configuración mínima, pero depende del proyecto principal.
- **JFrog Artifactory**: Gestiona más que solo contenedores; proporciona administración para cualquier artefacto.
- **Quay**: Esta es la distribución de código abierto del producto de Red Hat llamado Quay. Este proyecto ofrece una interfaz de usuario web completa, un servicio para el escaneo de vulnerabilidades de imágenes, almacenamiento de datos y protección.

No entraremos en todos los detalles de estos registros de contenedores. Lo que podemos sugerir con certeza es prestar atención y elegir el producto o proyecto que se adapte bien a tus casos de uso y necesidades de soporte. Muchos de estos productos tienen planes de soporte o ediciones empresariales (se requiere licencia) que podrían salvarte en caso de un desastre.

Veamos ahora cuáles son los registros de contenedores basados en la nube, que podrían hacernos la vida más fácil al ofrecer un servicio administrado completo con el cual nuestras tareas operativas podrían reducirse a cero.

#### Registros de contenedores basados en la nube

Como se anticipó en la sección anterior, los registros de contenedores basados en la nube podrían ser la forma más rápida de comenzar a trabajar con imágenes de contenedores a través de un registro.

Como se describe en el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/8), *Elección de la Imagen Base del Contenedor*, hay varios servicios de registro de contenedores basados en la nube en la web. Nos concentraremos solo en un pequeño subconjunto, dejando fuera del análisis los proporcionados por un proveedor de nube pública y los ofrecidos por la distribución de Linux, que generalmente solo están disponibles para descargar imágenes precargadas por los mantenedores de la distribución.

Echemos un vistazo a estos registros de contenedores en la nube:

- **Docker Hub**: Esta es la solución de registro alojada por Docker Inc. Este registro también aloja repositorios oficiales e imágenes verificadas por seguridad para algunos proyectos populares de código abierto.
- **Quay.io**: Esta es la solución de registro alojada nacida bajo la empresa CoreOS, ahora parte de Red Hat. Ofrece repositorios públicos y privados, escaneo automatizado para fines de seguridad, compilaciones de imágenes e integración con repositorios públicos populares de Git.

#### Registro en la nube Docker Hub

El registro en la nube Docker Hub nació junto con el proyecto Docker y representó una de las características clave agregadas a este proyecto, brindando a los contenedores en general la atención que merecían.

Hablando de características, Docker Hub tiene planes gratuitos y de pago:

- **Acceso anónimo**: Solo 10 descargas de imágenes por dirección IP por hora
- **Una cuenta de usuario registrado con el nivel gratuito**: 40 descargas de imágenes por hora
- **Cuentas Pro, Team y Business**: Miles de descargas de imágenes por día, compilaciones automatizadas, soporte, etc.

Como acabamos de señalar, si intentamos iniciar sesión con una cuenta de usuario registrada en el nivel gratuito, solo podemos crear repositorios públicos y un repositorio privado. Esto podría ser suficiente para comunidades o desarrolladores individuales, pero una vez que comienzas a usarlo a nivel empresarial, es posible que necesites las funciones adicionales proporcionadas por los planes de pago.

Para evitar una limitación significativa en términos de descargas de imágenes, deberíamos al menos usar una cuenta de usuario registrada e iniciar sesión tanto en el portal web como en el registro de contenedores utilizando nuestro motor de contenedores, Podman. Veremos en las siguientes secciones cómo autenticarnos en un registro e interactuar con él para publicar y descargar imágenes.

#### Registro en la nube Red Hat Quay.io

El registro en la nube Quay.io es el registro local de Red Hat, pero ofrecido como software como servicio (SaaS).

El registro en la nube Quay.io, al igual que Docker Hub, también ofrece planes de pago para desbloquear funciones adicionales.

Pero la buena noticia es que el nivel gratuito de Quay incluye muchas funciones:

- Compilación desde un Dockerfile cargado manualmente o incluso vinculado a través de GitHub/Bitbucket/GitLab o cualquier repositorio Git
- Escaneos de seguridad para imágenes enviadas al registro
- Registros de uso/auditoría
- Cuentas de usuario/tokens robot para integrar cualquier software externo
- Cinco repositorios privados
- Sin límite en las descargas de imágenes

Por otro lado, los planes de pago desbloquearán más repositorios privados y permisos basados en equipos.

Veamos el registro en la nube Quay.io creando un repositorio público y vinculándolo a un repositorio de GitHub en el que enviamos un Dockerfile para compilar nuestra imagen de contenedor de destino:

1. Primero, debemos registrarnos o iniciar sesión en el portal de Quay.io en [https://quay.io](https://quay.io/).
2. Después de eso, podemos hacer clic en el botón **+ Create New Repository** en la esquina superior derecha:

*Figura 9.1 – Botón Create New Repository de Quay*

3. Una vez hecho esto, el portal web solicitará información básica sobre el nuevo repositorio que queremos crear:
   - Un nombre
   - Una descripción
   - Público o privado (estamos usando una cuenta gratuita, así que público está bien)
   - Cómo inicializar el repositorio:

*Figura 9.2 – Página Create New Repository*

4. Acabamos de definir un nombre para nuestro repositorio, `ubi8-httpd`, y elegimos vincular este repositorio a un push en un repositorio de GitHub.
5. Una vez confirmado, el portal en la nube del registro Quay.io nos redirigirá a GitHub para permitir la autorización y luego nos pedirá que seleccionemos la organización y el repositorio de GitHub correctos para vincular:

*Figura 9.3 – Selección del repositorio de GitHub para vincular con nuestro repositorio de contenedores*

6. Acabamos de seleccionar la organización predeterminada y el repositorio Git que creamos, que contiene nuestro Dockerfile. El repositorio Git se llama `ubi8-httpd` y está disponible aquí: [https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition/tree/main/Chapter09/ubi8-httpd](https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition/tree/main/Chapter09/ubi8-httpd)

> [!NOTE]
> El repositorio utilizado en este ejemplo pertenece al proyecto del autor. Puedes hacer lo mismo bifurcando (*forking*) el repositorio en GitHub y creando tu propia copia con permisos de lectura/escritura para poder realizar cambios y experimentar con confirmaciones (*commits*) y compilaciones automatizadas.

7. Finalmente, nos pedirá que configuremos aún más el disparador (*trigger*):

*Figura 9.4 – Personalización del disparador de compilación*

8. Dejamos la opción predeterminada, que activará una nueva compilación cada vez que se realice un push en el repositorio Git para cualquier rama y etiqueta.
9. Una vez hecho esto, seremos redirigidos a la página principal del repositorio:

*Figura 9.5 – Página principal del repositorio*

Una vez creado, el repositorio está vacío, sin información ni actividad, por supuesto.

En la barra izquierda, podemos acceder fácilmente a la sección **Build**. Es el cuarto ícono empezando desde arriba. En la siguiente figura, acabamos de ejecutar dos pushes en nuestro repositorio Git, lo que activó dos compilaciones diferentes:

*Figura 9.6 – Sección de compilación de imágenes de contenedores*

Si intentamos hacer clic en una de las compilaciones, el registro en la nube mostrará los detalles de la compilación:

*Figura 9.7 – Detalles de la compilación de la imagen del contenedor*

Como podemos ver, la compilación funcionó como se esperaba: se conectó al repositorio de GitHub, descargó el Dockerfile, ejecutó la compilación y, finalmente, envió la imagen al registro de contenedores, todo de forma automatizada. El Dockerfile contiene solo unos pocos comandos para instalar un servidor httpd en una imagen base UBI8, como aprendimos en el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/8), *Elección de la Imagen Base del Contenedor*.

Finalmente, la última sección que vale la pena mencionar es la funcionalidad de escaneo de seguridad incluida. Se puede acceder a esta función haciendo clic en el ícono **Tag**, el segundo desde arriba en el panel izquierdo:

*Figura 9.8 – Página de etiquetas de la imagen del contenedor*

Como notarás, hay una columna **SECURITY SCAN** (la tercera) que informa el estado del escaneo ejecutado en esa imagen de contenedor en particular asociada con el nombre de la etiqueta reportado en la primera columna. Al hacer clic en el valor de esa columna (en la captura de pantalla anterior, es *Passed*), podemos obtener más detalles.

Acabamos de obtener algo de experiencia aprovechando un registro de contenedores ofrecido como un servicio administrado. Esto podría facilitarnos la vida al reducir nuestras tareas operativas, pero no siempre es la mejor opción para nuestros proyectos o empresas.

En la siguiente sección, exploraremos en detalle cómo administrar imágenes de contenedores con Skopeo, el compañero de Podman, y luego aprenderemos cómo configurar y ejecutar un registro de contenedores de forma local.

---

### Administración de imágenes de contenedores con Skopeo

Hasta ahora, hemos aprendido sobre muchos conceptos de registros de contenedores, incluidas las diferencias entre registros públicos y privados, su cumplimiento con las especificaciones de imagen OCI y cómo consumir imágenes con Podman y Buildah para compilar y ejecutar contenedores.

Sin embargo, a veces necesitamos implementar tareas sencillas de manipulación de imágenes, como mover una imagen de un registro a un espejo, inspeccionar una imagen remota sin necesidad de descargarla localmente o incluso firmar imágenes.

La comunidad que dio origen a Podman y Buildah desarrolló una tercera herramienta increíble, Skopeo ([https://github.com/containers/skopeo](https://github.com/containers/skopeo)), que implementa exactamente las características descritas anteriormente.

Skopeo se diseñó como una herramienta de manipulación de registros e imágenes para equipos de DevOps y no está diseñada para ejecutar contenedores (la función principal de Podman) ni para compilar imágenes OCI (la función principal de Buildah). En su lugar, ofrece una interfaz de línea de comandos mínima y sencilla con comandos básicos de manipulación de imágenes que resultarán extremadamente útiles en diferentes contextos.

Inspeccionemos las características más interesantes en las siguientes subsecciones.

#### Instalación de Skopeo

Skopeo es una herramienta binaria en Go que ya está empaquetada y disponible para muchas distribuciones. También se puede compilar e instalar directamente desde el código fuente.

Esta sección proporciona una lista no exhaustiva de ejemplos de instalación en las principales distribuciones. En aras de la claridad, es importante reiterar que los entornos de laboratorio del libro se basaron todos en Fedora 40:

- **Fedora, RHEL 8/9, CentOS 8 y CentOS Stream 8/9**: Para instalar Skopeo en sistemas tipo RHEL, ejecuta el siguiente comando `dnf`:

```bash
$ sudo dnf -y install skopeo
```

- **Debian**: Para instalar Skopeo en Debian Bullseye, Testing e Unstable (Sid), ejecuta los siguientes comandos `apt-get`:

```bash
$ sudo apt-get update
$ sudo apt-get -y install skopeo
```

- **Ubuntu**: Para instalar Skopeo en Ubuntu 20.10 y versiones posteriores, ejecuta el siguiente comando:

```bash
$ sudo apt-get -y update
$ sudo apt-get -y install skopeo
```

- **Arch Linux**: Para instalar Skopeo en Arch Linux, ejecuta el siguiente comando `pacman`:

```bash
$ sudo pacman -S skopeo
```

- **openSUSE**: Para instalar Skopeo en openSUSE, ejecuta el siguiente comando `zypper`:

```bash
$ sudo zypper install skopeo
```

- **macOS**: Para instalar Skopeo en macOS, ejecuta el siguiente comando `brew`:

```bash
$ brew install skopeo
```

- **Compilación desde el código fuente**: Skopeo también se puede compilar desde el código fuente. Al igual que con Buildah, para los propósitos de este libro, mantendremos el enfoque en métodos de implementación simples, pero si tienes curiosidad, puedes encontrar una sección de instalación dedicada en el repositorio principal del proyecto que ilustra cómo compilar Skopeo desde el código fuente: [https://github.com/containers/skopeo/blob/main/install.md#building-from-source](https://github.com/containers/skopeo/blob/main/install.md#building-from-source). El enlace anterior muestra ejemplos de compilaciones en contenedores y no en contenedores.
- **Ejecución de Skopeo en un contenedor**: Skopeo también se publica como una imagen de contenedor que se puede ejecutar con Podman. Para descargar y ejecutar la última versión de Skopeo como un contenedor, usa el siguiente comando `podman`:

```bash
$ podman run quay.io/skopeo/stable:latest <command> <options>
```

- **Windows**: Al momento de escribir este libro, no hay una compilación disponible para Microsoft Windows. Sin embargo, puedes instalarlo en Windows Subsystem for Linux (WSL). Dado que WSL es esencialmente un entorno Linux, el proceso de instalación de Skopeo depende completamente de la distribución que hayas instalado (Fedora, Ubuntu, etc.).

Skopeo utiliza los mismos archivos de configuración locales y del sistema descritos para Podman y Buildah; por lo tanto, podemos centrarnos inmediatamente en la verificación de la instalación y el análisis de los casos de uso más comunes.

#### Verificación de la instalación

Para verificar la instalación correcta, simplemente ejecuta el comando `skopeo` con la opción `-h` o `--help` para ver todos los comandos disponibles, como en el siguiente ejemplo:

```bash
$ skopeo -h
```

La salida esperada mostrará, entre las opciones de utilidades, todos los comandos disponibles, cada uno con una descripción del alcance del comando. La lista completa de comandos es la siguiente:

- **copy**: Copia una imagen entre ubicaciones, utilizando diferentes transportes, como Docker Registry, directorios locales, OCI, tarballs, OSTree y archivos OCI
- **delete**: Elimina una imagen de una ubicación de destino
- **generate-sigstore-key**: Genera un par de claves pública/privada de sigstore
- **help**: Imprime los comandos de ayuda
- **inspect**: Inspecciona los metadatos, las etiquetas y la configuración de una imagen en una ubicación de destino
- **list-tags**: Muestra las etiquetas disponibles para un repositorio de imágenes específico
- **login**: Se autentica en un registro remoto
- **logout**: Cierra la sesión en un registro remoto
- **manifest-digest**: Produce un resumen de manifiesto (*manifest digest*) para un archivo
- **standalone-sign**: Una herramienta de depuración para publicar y firmar una imagen utilizando archivos locales
- **standalone-verify**: Verifica la firma de una imagen utilizando archivos locales
- **sync**: Sincroniza una o más imágenes entre ubicaciones

Inspeccionemos ahora con mayor detalle algunos de los comandos de Skopeo más interesantes.

#### Copia de imágenes entre ubicaciones

Podman, al igual que Docker, se puede utilizar no solo para ejecutar contenedores sino también para descargar imágenes localmente y enviarlas a otras ubicaciones. Sin embargo, una de las principales advertencias es la necesidad de ejecutar dos comandos, uno para descargar y otro para enviar, mientras que el almacenamiento de imágenes local permanece lleno con las imágenes descargadas. Por lo tanto, los usuarios deben limpiar periódicamente el almacén local.

Skopeo ofrece una forma más inteligente y sencilla de lograr este objetivo con el comando `skopeo copy`. El comando implementa la siguiente sintaxis:

```text
skopeo copy [command options] SOURCE-IMAGE DESTINATION-IMAGE
```

En esta descripción genérica, `SOURCE-IMAGE` y `DESTINATION-IMAGE` son imágenes que pertenecen a ubicaciones locales o remotas y a las que se puede acceder utilizando uno de los siguientes transportes:

- `docker://docker-reference`: Este transporte está relacionado con imágenes almacenadas en registros que implementan la API HTTP V2 de Docker Registry. Esta configuración utiliza el archivo `/etc/containers/registries.conf` o `$HOME/.config/containers/registries.conf` para obtener configuraciones de registro adicionales. El campo `docker-reference` sigue el formato `name[:tag|@digest]`.
- `containers-storage:[[storage-specifier]]{image-id|docker-reference[@image-id]}`: Esta configuración se refiere a una imagen en el almacenamiento de contenedores local. El campo `storage-specifier` tiene el formato `[[driver@]root[+run-root][:options]]`.
- `dir:path`: Esta configuración se refiere a un directorio local existente que contiene manifiestos, capas (en formato tarball) y firmas.
- `docker-archive:path[:{docker-reference|@source-index}]`: Esta configuración se refiere a un archivo Docker obtenido con el comando `docker save` o `podman save`.
- `docker-daemon:docker-reference|algo:digest`: Esta configuración se refiere al almacenamiento de imágenes en el almacenamiento interno del demonio de Docker.
- `oci:path[:tag]`: Esta configuración se refiere a una imagen almacenada en una ruta local compatible con las especificaciones de diseño (*layout*) de OCI.
- `oci-archive:path[:tag]`: Esta configuración se refiere a una imagen compatible con la especificación de diseño OCI almacenada en formato tarball.

Inspeccionemos algunos ejemplos de uso del comando `skopeo copy` en escenarios del mundo real. El primer ejemplo muestra cómo copiar una imagen de un registro remoto a otro registro remoto:

```bash
$ skopeo copy \
  docker://docker.io/library/nginx:latest \
  docker://private-registry.example.com/lab/nginx:latest
```

El ejemplo anterior no se encarga de la autenticación del registro, que suele ser un requisito para enviar imágenes al repositorio remoto. En el siguiente ejemplo, mostramos una variante donde tanto el registro de origen como el de destino están decorados con opciones de autenticación:

```bash
$ skopeo copy \
  --src-creds USERNAME:PASSWORD \
  --dest-creds USERNAME:PASSWORD \
  docker://registry1.example.com/mirror/nginx:latest \
  docker://registry2.example.com/lab/nginx:latest
```

El enfoque anterior, a pesar de funcionar perfectamente, tiene la limitación de pasar cadenas de nombre de usuario y contraseña como texto sin formato. Para evitar esto, podemos usar el comando `skopeo login` para autenticarnos en nuestros registros antes de ejecutar `skopeo copy`.

El tercer ejemplo muestra una autenticación previa en el registro de destino, asumiendo que el registro de origen es de acceso público para descargas:

```bash
$ skopeo login private-registry.example.com
$ skopeo copy \
  docker://docker.io/library/nginx:latest \
  docker://private-registry.example.com/lab/nginx:latest
```

Cuando iniciamos sesión en los registros de origen/destino, el sistema persiste los tokens de autenticación proporcionados por el registro en archivos de autenticación dedicados que podemos reutilizar más adelante para un acceso posterior.

De forma predeterminada, Skopeo busca en la ruta `${XDG_RUNTIME_DIR}/containers/auth.json`, pero podemos proporcionar una ubicación personalizada para el archivo de autenticación. Por ejemplo, si usamos el tiempo de ejecución del contenedor Docker anteriormente, podríamos encontrarlo en la ruta `${HOME}/.docker/config.json`. Este archivo contiene un objeto JSON simple que contiene, para cada registro utilizado, el token obtenido tras la autenticación. El cliente (Podman, Skopeo o Buildah) utilizará este token para acceder directamente al registro. Básicamente, una vez que una herramienta inicia sesión, todas las herramientas pueden usar el token. `podman login` permite que `skopeo copy` y `buildah push` accedan al registro, por ejemplo.

El siguiente ejemplo muestra el uso del archivo de autenticación, provisto con una ruta personalizada:

```bash
$ skopeo copy \
  --authfile ${HOME}/.docker/config.json \
  docker://docker.io/library/nginx:latest \
  docker://private-registry.example.com/lab/nginx:latest
```

Otro problema común que se puede encontrar al trabajar con un registro privado es la falta de certificados firmados por una autoridad de certificación (CA) conocida o la falta de comunicación HTTPS (lo que significa que todo el tráfico no está cifrado). Si consideramos que estos escenarios totalmente no seguros son seguros para confiar en un entorno de laboratorio, podemos omitir la verificación TLS con las opciones `--dest-tls-verify` y `--src-tls-verify`, que aceptan un valor booleano simple.

El siguiente ejemplo muestra cómo omitir la verificación TLS en el registro de destino:

```bash
$ skopeo copy \
  --authfile ${HOME}/.docker/config.json \
  --dest-tls-verify false \
  docker://docker.io/library/nginx:latest \
  docker://private-registry.example.com/lab/nginx:latest
```

Hasta ahora, hemos visto cómo mover imágenes a través de registros públicos y privados, pero podemos usar Skopeo para mover imágenes hacia y desde almacenes locales fácilmente. Por ejemplo, podemos usar Skopeo como una herramienta de envío/descarga altamente especializada para imágenes dentro de nuestras canalizaciones de compilación.

El siguiente ejemplo muestra cómo enviar una imagen creada localmente a un registro público. La imagen ya existe localmente y luego se envía al registro remoto:

```bash
$ podman images
REPOSITORY                 TAG      IMAGE ID       CREATED        SIZE
<namespace>/python_httpd   latest   4067fa24786a   12 days ago    318 MB
$ skopeo copy \
  --authfile ${HOME}/.docker/config.json \
  containers-storage:quay.io/<namespace>/python_httpd \
  docker://quay.io/<namespace>/python_httpd:latest
```

Esta es una forma increíble de administrar la publicación de imágenes con un control total sobre el proceso de envío/descarga y muestra cómo las tres herramientas (Podman, Buildah y Skopeo) pueden realizar tareas especializadas en nuestro entorno DevOps, logrando cada una el propósito para el que fue diseñada de la mejor manera.

Veamos otro ejemplo, esta vez mostrando cómo descargar una imagen de un registro remoto a un almacén local compatible con OCI:

```bash
$ skopeo copy \
  --authfile ${HOME}/.docker/config.json \
  docker://docker.io/library/nginx:latest \
  oci:/tmp/nginx
```

La carpeta de salida cumple con las especificaciones de imagen OCI y tendrá la siguiente estructura (los hashes de blob se cortaron por razones de diseño): Esto básicamente expande el tarball de la imagen en una ubicación específica, lo que permite una fácil inspección:

```text
$ tree /tmp/nginx
/tmp/nginx/
├── blobs
│   └── sha256
│       ├── 21e0df283cd68384e5e8dff7e6be1774c86ea3110c1b1e932[...]
│       ├── 44be98c0fab60b6cef9887dbad59e69139cab789304964a19[...]
│       ├── 77700c52c9695053293be96f9cbcf42c91c5e097daa382933[...]
│       ├── 81d15e9a49818539edb3116c72fbad1df1241088116a7363a[...]
│       ├── 881ff011f1c9c14982afc6e95ae70c25e38809843bb7d42ab[...]
│       ├── d86da3a6c06fb46bc76d6dc7b591e87a73cb456c990d814fd[...]
│       ├── e5ae68f740265288a4888db98d2999a638fdcb6d725f42767[...]
│       └── ed835de16acd8f5821cf3f3ef77a66922510ee6349730d89a[...]
├── index.json
└── oci-layout
```

Los archivos dentro de la carpeta `blobs/sha256` incluyen el manifiesto de la imagen (en formato JSON) y las capas de la imagen en formato tarball comprimido.

Es interesante saber que Podman puede ejecutar sin problemas un contenedor basado en una carpeta local que cumpla con las especificaciones de imagen OCI. El siguiente ejemplo muestra cómo ejecutar un contenedor NGINX desde la imagen descargada previamente:

```bash
$ podman run -d oci:/tmp/nginx
Getting image source signatures
Copying blob e5ae68f74026 done
Copying blob 21e0df283cd6 done
Copying blob ed835de16acd done
Copying blob 881ff011f1c9 done
Copying blob 77700c52c969 done
Copying blob 44be98c0fab6 done
Copying config 81d15e9a49 done
Writing manifest to image destination
Storing signatures
90493fe89f024cfffda3f626acb5ba8735cadd827be6c26fa44971108e09b54f
```

Observa el prefijo `oci:` antes de la ruta de la imagen, necesario para especificar que la ruta proporcionada es compatible con OCI.

Además, es interesante mostrar que Podman copia y extrae los blobs dentro de su almacén local (en `$HOME/.local/share/containers/storage` para un contenedor rootless como el del ejemplo anterior).

Después de aprender a copiar imágenes con Skopeo, veamos cómo inspeccionar imágenes remotas sin necesidad de descargarlas localmente.

#### Inspección de imágenes remotas

A veces necesitamos verificar las configuraciones, etiquetas o metadatos de una imagen antes de descargarla y ejecutarla localmente. Para este propósito, Skopeo ofrece el útil comando `skopeo inspect` para inspeccionar imágenes a través de transportes compatibles.

El primer ejemplo muestra cómo inspeccionar el repositorio oficial de imágenes de NGINX:

```bash
$ skopeo inspect docker://docker.io/library/nginx
```

El comando `skopeo inspect` crea una salida con formato JSON con los siguientes campos:

- **Name**: El nombre del repositorio de imágenes.
- **Digest**: El resumen (*digest*) calculado con SHA256.
- **RepoTags**: La lista completa de etiquetas de imagen disponibles en el repositorio. Esta lista estará vacía al inspeccionar transportes locales, como `containers-storage:` u `oci:`, ya que se referirán a una sola imagen.
- **Created**: La fecha de creación del repositorio o imagen.
- **DockerVersion**: La versión de Docker utilizada para crear la imagen. Este valor está vacío para las imágenes creadas con Podman, Buildah u otras herramientas.
- **Labels**: Etiquetas adicionales aplicadas a la imagen en el momento de la compilación.
- **Architecture**: La arquitectura del sistema de destino para la que se compiló la imagen. Este valor es `amd64` para sistemas x86-64.
- **Os**: El sistema operativo de destino para el que se compiló la imagen.
- **Layers**: La lista de capas que componen la imagen, junto con su resumen SHA256.
- **Env**: Variables de entorno adicionales definidas en la imagen en el momento de la compilación.

Las mismas consideraciones ilustradas previamente sobre la autenticación y la verificación TLS se aplican al comando `skopeo inspect`: es posible inspeccionar imágenes en un registro privado previa autenticación y omitir la verificación TLS. El siguiente ejemplo muestra este caso de uso:

```bash
$ skopeo inspect \
  --authfile ${HOME}/.docker/config.json \
  --tls-verify false \
  registry.example.com/library/test-image
```

Inspeccionar imágenes locales es posible pasando el transporte correcto. El siguiente ejemplo muestra cómo inspeccionar una imagen OCI local:

```bash
$ skopeo inspect oci:/tmp/custom_image
```

La salida de este comando tendrá un campo `RepoTags` vacío.

Además, es posible usar la opción `--no-tags` para omitir intencionalmente las etiquetas del repositorio, como en el siguiente ejemplo:

```bash
$ skopeo inspect --no-tags docker://docker.io/library/nginx
```

Por otro lado, si necesitamos imprimir solo las etiquetas de repositorio disponibles, podemos usar el comando `skopeo list-tags`. El siguiente ejemplo imprime todas las etiquetas disponibles del repositorio oficial de NGINX:

```bash
$ skopeo list-tags docker://docker.io/library/nginx
```

El tercer caso de uso que vamos a analizar es la sincronización de imágenes entre registros y almacenes locales.

#### Sincronización de registros y directorios locales

Cuando se trabaja con entornos desconectados, un escenario bastante común es la necesidad de sincronizar repositorios desde un registro remoto de forma local.

Para cumplir con este propósito, Skopeo introdujo el comando `skopeo sync`, que ayuda a sincronizar contenido entre un origen y un destino, admitiendo diferentes tipos de transporte.

Podemos usar este comando para sincronizar un repositorio completo, con todas las etiquetas disponibles en su interior, entre un origen y un destino. Alternativamente, es posible sincronizar solo una etiqueta de imagen específica.

El primer ejemplo muestra cómo sincronizar el repositorio oficial de busybox desde un registro privado con el sistema de archivos local. Este comando descarga todas las etiquetas contenidas en el repositorio remoto al destino local (el directorio de destino ya debe existir):

```bash
$ mkdir /tmp/images
$ skopeo sync \
  --src docker --dest dir \
  registry.example.com/lab/busybox /tmp/images
```

Observa el uso de las opciones `--src` y `--dest` para definir el tipo de transporte. Los tipos de transporte admitidos son los siguientes:

- **Origen**: `docker`, `dir` y `yaml` (cubierto más adelante en esta sección)
- **Destino**: `docker` y `dir`

De forma predeterminada, Skopeo sincroniza el contenido del repositorio con el destino sin la ruta completa de origen de la imagen. Esto podría representar una limitación cuando necesitamos sincronizar repositorios con el mismo nombre de múltiples fuentes. Para resolver esta limitación, podemos agregar la opción `--scoped` y obtener la ruta de origen completa de la imagen copiada en el árbol de destino.

El segundo ejemplo muestra una sincronización con alcance (*scoped*) del repositorio de busybox:

```bash
$ skopeo sync \
  --src docker --dest dir --scoped \
  registry.example.com/lab/busybox /tmp/images
```

La ruta resultante en el directorio de destino contendrá el nombre del registro y el espacio de nombres relacionado, con una nueva carpeta con el nombre de la etiqueta de la imagen.

El siguiente ejemplo muestra la estructura de directorios del destino después de una sincronización exitosa:

```text
ls -A1 /tmp/images/docker.io/library/
busybox:1
busybox:1.21.0-ubuntu
busybox:1.21-ubuntu
busybox:1.23
busybox:1.23.2
busybox:1-glibc
busybox:1-musl
busybox:1-ubuntu
busybox:1-uclibc
[...omitted output...]
```

Si necesitamos sincronizar solo una etiqueta de imagen específica, es posible especificar el nombre de la etiqueta en el argumento de origen, como en este tercer ejemplo:

```bash
$ skopeo sync --src docker --dest dir docker.io/library/busybox:latest /tmp/images
```

Podemos sincronizar directamente dos registros usando Docker, tanto para el transporte de origen como para el de destino. Esto es especialmente útil en entornos desconectados donde a los sistemas solo se les permite acceder a un registro local. El registro local puede reflejar repositorios de otros registros públicos o privados, y la tarea se puede programar periódicamente para mantener actualizado el espejo.

> [!IMPORTANT]
> Docker Hub impone límites estrictos de tasa de descarga a usuarios no autenticados (anónimos). Si estás siguiendo estos ejercicios y no has iniciado sesión en una cuenta de Docker, es posible que alcances rápidamente estos límites, lo que provocará que fallen las descargas de imágenes. Para evitar errores de *Too Many Requests* y garantizar una experiencia más confiable, recomendamos usar las imágenes alojadas en Quay, que actualmente ofrece una vía más generosa para descargas públicas y no autenticadas.

El siguiente ejemplo muestra cómo sincronizar la imagen UBI9 y todas sus etiquetas desde el repositorio público de Red Hat a un registro espejo local:

```bash
$ skopeo sync \
  --src docker --dest docker \
  --dest-tls-verify=false \
  registry.access.redhat.com/ubi9 \
  mirror-registry.example.com
```

El comando anterior duplicará todas las etiquetas de imagen de UBI9 en el registro de destino.

Observa la opción `--dest-tls-verify=false` para deshabilitar las comprobaciones de certificados TLS en el destino.

El comando `skopeo sync` es excelente para duplicar repositorios e imágenes individuales entre ubicaciones, pero cuando se trata de duplicar registros completos o un gran conjunto de repositorios, tendríamos que ejecutar el comando muchas veces, pasando diferentes argumentos de origen.

Para evitar esta limitación, el transporte de origen se puede definir como un archivo YAML para incluir una lista exhaustiva de registros, repositorios e imágenes. También es posible utilizar expresiones regulares para capturar solo subconjuntos seleccionados de etiquetas de imagen.

El siguiente es un ejemplo de un archivo YAML personalizado que se pasará como argumento de origen a Skopeo (`Chapter09/example_sync.yaml`):

```yaml
docker.io:
  tls-verify: true
  images:
    alpine: []
    nginx:
      - "latest"
  images-by-tag-regex:
    httpd: ^2\.4\.[0-9]*-alpine$
quay.io:
  tls-verify: true
  images:
    fedora/fedora:
      - latest
registry.access.redhat.com:
  tls-verify: true
  images:
    ubi9:
      - "9.4"
      - "9.5"
```

En el ejemplo anterior, se definen diferentes imágenes y repositorios y, por lo tanto, el contenido del archivo merece una descripción detallada.

Todo el repositorio de `alpine` se descarga de `docker.io`, junto con la etiqueta de imagen `nginx:latest`. Además, se utiliza una expresión regular para definir un patrón de etiquetas para la imagen de `httpd`, con el fin de descargar solo la versión de imagen 2.4.z basada en Alpine.

El archivo también define una etiqueta específica (`latest`) para la imagen de `fedora` almacenada en [https://quay.io/](https://quay.io/) y las etiquetas `9.4` y `9.5` para la imagen de `ubi9` almacenada en el registro `registry.access.redhat.com`.

Una vez definido, el archivo se pasa como argumento a Skopeo, junto con el destino:

```bash
$ skopeo sync \
  --src yaml --dest dir \
  --scoped example_sync.yaml /tmp/images
```

Todos los contenidos enumerados en el archivo `example_sync.yaml` se copiarán en el directorio de destino, siguiendo las reglas de filtrado mencionadas anteriormente.

El siguiente ejemplo muestra un caso de uso de duplicación más grande, aplicado a las imágenes de lanzamiento de OpenShift. El siguiente archivo `openshift_sync.yaml` define una expresión regular para sincronizar todas las imágenes para la versión 4.17.z de OpenShift compilada para la arquitectura x86_64 (`Chapter09/openshift_sync.yaml`):

```yaml
quay.io:
  tls-verify: true
  images-by-tag-regex:
    openshift-release-dev/ocp-release: ^4\.17\..*-x86_64$
```

> [!NOTE]
> El flujo `z` en un número de versión (como `x.y.z`) que normalmente representa versiones de parches que contienen correcciones de errores y mejoras menores que no introducen nuevas características. Aumentar el valor `z` indica una actualización estable y compatible con versiones anteriores centrada en mejorar la calidad de la versión existente.

Podemos usar este archivo para duplicar una versión menor completa de OpenShift en un registro interno accesible desde entornos desconectados y usar este espejo para realizar con éxito una instalación aislada (*air-gapped*) de OpenShift Container Platform. El siguiente ejemplo de comando muestra este caso de uso:

```bash
$ skopeo sync \
  --src yaml --dest docker \
  --dest-tls-verify=false \
  --src-authfile pull_secret.json \
  openshift_sync.yaml mirror-registry.example.com:5000
```

Vale la pena señalar el uso de un archivo pull secret, pasado con la opción `--src-authfile`, para autenticarse en el registro público de Quay y descargar imágenes del repositorio `ocp-release`.

Hay una última característica de Skopeo que capta nuestro interés: la eliminación remota de imágenes, cubierta en la siguiente subsección.

#### Eliminación de imágenes

Un registro se puede imaginar como un almacén de objetos especializado que implementa un conjunto de API HTTP para manipular su contenido y publicar/descargar objetos en forma de capas de imágenes y metadatos.

El protocolo Docker Registry v2 es una especificación de API estándar que se adopta ampliamente entre muchos proyectos de registro [3]. Este conjunto de especificaciones de API cubre todas las funciones de registro que se espera que se expongan a un cliente externo a través de los métodos HTTP estándar GET, PUT, DELETE, POST y PATCH.

Esto significa que podríamos interactuar con un registro con cualquier tipo de cliente HTTP capaz de gestionar las solicitudes correctamente, por ejemplo, el comando `curl`.

Cualquier motor de contenedores utiliza, en un nivel inferior, bibliotecas cliente HTTP para ejecutar los distintos métodos contra el registro (por ejemplo, para la descarga de una imagen).

El protocolo Docker v2 también admite la eliminación remota de imágenes, y cualquier registro que implemente este protocolo admite la siguiente solicitud DELETE para imágenes:

```text
DELETE /v2/<name>/manifests/<reference>
```

El siguiente ejemplo representa un comando DELETE teórico emitido con el comando `curl` contra un registro local:

```bash
$ curl -v --silent \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -X DELETE http://127.0.0.1:5000/v2/<name>/manifests/sha256:<image_tag_digest>
```

El ejemplo anterior evita intencionalmente incluir la administración de tokens de autorización para facilitar la lectura.

Podman y Docker, diseñados para funcionar como motores de registro, no implementan una función de eliminación remota entre sus interfaces de comandos.

Afortunadamente, Skopeo viene al rescate con su comando integrado `skopeo delete` para administrar la eliminación remota de imágenes con una sintaxis simple y fácil de usar.

El siguiente ejemplo elimina una imagen en un hipotético registro interno `mirror-registry.example.com:5000`:

```bash
$ skopeo delete \
  docker://mirror-registry.example.com:5000/foo:bar
```

El comando elimina inmediatamente las referencias de etiquetas de imagen en el registro remoto.

> [!IMPORTANT]
> Al eliminar imágenes con Skopeo, es necesario habilitar la eliminación de imágenes en el registro remoto, como se explica en la siguiente sección, *Ejecución de un registro de contenedores local*.

En esta sección, hemos aprendido a usar Skopeo para copiar, eliminar, inspeccionar y sincronizar imágenes o incluso repositorios completos a través de diferentes transportes, incluidos registros locales privados, obteniendo el control sobre las operaciones diarias de manipulación de imágenes.

No cubrimos el proceso de firma de la imagen del contenedor aquí; lo exploraremos en profundidad con ejemplos prácticos en el [Capítulo 10](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/10), *Aseguramiento de Contenedores*.

En la siguiente sección, aprenderemos a ejecutar y configurar un registro de contenedores local para administrar directamente el almacenamiento de imágenes en nuestros entornos de desarrollo o laboratorio.

---

### Ejecución de un registro de contenedores local

La mayoría de las empresas y organizaciones adoptan registros de nivel empresarial para confiar en soluciones seguras y resilientes para el almacenamiento de sus imágenes de contenedores. La mayoría de los registros empresariales también ofrecen funciones avanzadas como control de acceso basado en roles (RBAC), un escáner de vulnerabilidades de imágenes, duplicación (*mirroring*), replicación geográfica y alta disponibilidad, convirtiéndose en la opción predeterminada para entornos de producción y de misión crítica.

Sin embargo, a veces es muy útil ejecutar un registro local simple, por ejemplo, en entornos de desarrollo o laboratorios de capacitación. Los registros locales también pueden ser útiles en entornos desconectados para duplicar los principales registros públicos o privados.

Esta sección tiene como objetivo ilustrar cómo ejecutar un registro local simple y cómo aplicar configuraciones básicas.

#### Ejecución de un registro en contenedor

Como cualquier aplicación, un administrador puede instalar un registro local en el host. Alternativamente, un enfoque comúnmente preferido es ejecutar el registro dentro de un contenedor.

La solución de registro en contenedor más utilizada se basa en la imagen oficial Docker Registry 2.0, que ofrece todas las funcionalidades necesarias para un registro básico y es muy fácil de usar.

Al ejecutar un registro local, ya sea en un contenedor o no, debemos definir un directorio de destino para alojar todas las capas de imágenes y los metadatos. El siguiente ejemplo muestra la primera ejecución de un registro en contenedor, con el volumen `/var/lib/registry` creado y montado para almacenar datos de imágenes:

```bash
# podman volume create registry_data
# podman run -d \
  --name local_registry \
  -p 5000:5000 \
  -v registry_data:/var/lib/registry:Z \
  --restart=always registry:2
```

El registro estará accesible en la dirección del host en el puerto 5000/tcp, que también es el puerto predeterminado para este servicio. Si ejecutamos el registro en nuestra estación de trabajo local, será accesible en `localhost:5000`, y estará expuesto a la conexión externa utilizando la dirección IP asignada o su nombre de dominio completo (FQDN) si la estación de trabajo/portátil es resuelta por un servicio DNS local.

Por ejemplo, si un host tiene la dirección IP `10.10.2.30` y el FQDN `registry.example.com` resuelto correctamente mediante consultas DNS, el servicio de registro será accesible en `10.10.2.30:5000` o en `registry.example.com:5000`.

> [!IMPORTANT]
> Si el host ejecuta un servicio de firewall local o está detrás de un firewall corporativo, no olvides abrir los puertos correctos para exponer el registro externamente.

Podemos intentar compilar y enviar una imagen de prueba al nuevo registro. El siguiente Containerfile compila un servidor httpd básico basado en UBI (`Chapter09/local_registry/minimal_httpd/Containerfile`):

```dockerfile
FROM registry.access.redhat.com/ubi8:latest
RUN dnf install -y httpd && dnf clean all -y
COPY index.html /var/www/html
RUN dnf install -y git && dnf clean all -y
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Podemos compilar la nueva imagen con Buildah:

```bash
$ buildah build -t minimal_httpd .
```

Para enviar la imagen al registro local, podemos usar Podman o sus herramientas complementarias, Buildah o Skopeo. Skopeo es muy útil para estos casos de uso, ya que ni siquiera necesitamos definir el alcance del nombre de la imagen con el nombre del registro.

El siguiente comando muestra cómo enviar la nueva imagen al registro:

```bash
$ skopeo copy --dest-tls-verify=false \
  containers-storage:localhost/minimal_httpd \
  docker://localhost:5000/minimal_httpd
```

Observa el uso de `--dest-tls-verify=false`: esto es necesario ya que el registro local no tiene TLS ni un certificado de confianza; proporciona un transporte HTTP de forma predeterminada.

A pesar de ser simple de implementar, la configuración de registro predeterminada tiene algunas limitaciones que deben abordarse. Para ilustrar una de esas limitaciones, intentemos eliminar la imagen que acabamos de cargar:

```bash
$ skopeo delete \
  --tls-verify=false \
  docker://localhost:5000/minimal_httpd
FATA[0000] Failed to delete /v2/minimal_httpd/manifests/sha256:f8c0c374cf124e728e20045f327de30ce1f3c552b307945de9b911cbee103522: {"errors":[{"code":"UNSUPPORTED","message":"The operation is unsupported."}]} (405 Method Not Allowed)
```

Como podemos ver en la salida anterior, el registro no nos permitió eliminar la imagen, devolviendo un mensaje de error HTTP 405. Para alterar este comportamiento, necesitamos editar la configuración del registro.

#### Personalización de la configuración del registro

El archivo de configuración del registro (`/etc/docker/registry/config.yml`) se puede modificar para alterar su comportamiento. El contenido predeterminado de este archivo es el siguiente:

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Pronto nos damos cuenta de que se trata de una configuración extremadamente básica sin autenticación, sin eliminación permitida de imágenes y sin cifrado TLS. Nuestra versión personalizada intentará abordar esas limitaciones.

> [!NOTE]
> La documentación completa sobre la configuración del registro tiene una amplia gama de opciones que no mencionamos aquí porque está fuera del alcance de este libro. Se pueden encontrar más opciones de configuración en este enlace: [https://docs.docker.com/registry/configuration/](https://docs.docker.com/registry/configuration/).

El siguiente archivo contiene una versión modificada del registro `config.yml` (`Chapter09/local_registry/customizations/config.yml`):

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
auth:
  htpasswd:
    realm: basic-realm
    path: /var/lib/htpasswd
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
  tls:
    certificate: /etc/pki/certs/tls.crt
    key: /etc/pki/certs/tls.key
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Las secciones destacadas en el ejemplo anterior enfatizan las siguientes características agregadas:

- **Eliminación de imágenes**: De forma predeterminada, esta configuración está deshabilitada. Necesitamos habilitarla explícitamente.
- **Autenticación básica mediante un archivo htpasswd**: Este enfoque es adecuado para entornos de desarrollo y laboratorio, mientras que una autenticación basada en tokens que dependa de un emisor externo se adaptaría mejor a los casos de uso de producción.
- **Transporte HTTPS mediante certificados autofirmados**: La sección `http` configura el servidor HTTP del registro, incluida su dirección de escucha, encabezados personalizados y configuraciones TLS.

Antes de ejecutar el registro nuevamente con nuestra configuración personalizada, debemos generar un archivo `htpasswd` que contenga al menos un inicio de sesión válido y los certificados autofirmados para el cifrado TLS. Comencemos con el archivo `htpasswd`: podemos generarlo usando la utilidad `htpasswd`, como en el siguiente ejemplo:

```bash
htpasswd -cBb ./htpasswd admin p0dman4Dev0ps#
```

La opción `-cBb` habilita el modo por lotes (útil para proporcionar la contraseña de forma no interactiva), crea el archivo si no existe y habilita la función de hash bcrypt [2]. En este ejemplo, creamos el usuario `admin` con la contraseña `p0dman4Dev0ps#`.

Finalmente, necesitamos crear un certificado de servidor autofirmado con su clave privada relacionada, para usarlo en conexiones HTTPS. A modo de ejemplo, se creará un certificado asociado con el nombre común (*Common Name* o CN) `localhost`.

> [!IMPORTANT]
> Vincular certificados al CN `localhost` es una práctica frecuente en entornos de desarrollo. Sin embargo, si el registro está destinado a exponerse externamente, los campos CN y SubjectAltName deben asignarse al FQDN del host y a los nombres alternativos.

El siguiente ejemplo muestra cómo crear un certificado autofirmado con la utilidad `openssl`:

```bash
$ mkdir certs
$ openssl req -newkey rsa:4096 -x509 -sha256 -nodes \
  -days 365 \
  -out certs/tls.crt \
  -keyout certs/tls.key \
  -subj '/CN=localhost' \
  -addext "subjectAltName=DNS:localhost"
```

El comando emitirá una generación de certificados no interactiva, sin ninguna información adicional sobre el sujeto del certificado. La clave privada `tls.key` se genera utilizando un algoritmo RSA de 4096 bits. El certificado, llamado `tls.crt`, está configurado para caducar después de 1 año. Tanto la clave como el certificado se escriben dentro del directorio `certs`.

Para inspeccionar el contenido del certificado generado, podemos ejecutar el siguiente comando:

```bash
$ openssl x509 -in certs/tls.crt -text -noout
```

El comando producirá un volcado legible por humanos de los datos y la validez del certificado.

> [!TIP]
> Para el propósito de este ejemplo, el certificado autofirmado es aceptable, pero debe evitarse en escenarios de producción. Soluciones como Let's Encrypt brindan un servicio de CA gratuito para todos y se pueden usar para proteger de manera confiable el registro o cualquier otro servicio HTTPS. Para obtener más detalles, visita [https://letsencrypt.org/](https://letsencrypt.org/).

Ahora tenemos todos los requisitos para ejecutar nuestro registro personalizado. Antes de crear el nuevo contenedor, asegúrate de que la instancia anterior se haya detenido y eliminado:

```bash
# podman stop local_registry && podman rm local_registry
```

El siguiente comando muestra cómo ejecutar el nuevo registro personalizado utilizando montajes de enlace (*bind mounts*) para pasar la carpeta de certificados, el archivo `htpasswd`, el almacén del registro y, obviamente, el archivo de configuración personalizado:

```bash
# podman volume create registry_data
# podman run -d --name local_registry \
  -p 5000:5000 \
  -v $PWD/htpasswd:/var/lib/htpasswd:z \
  -v $PWD/config.yml:/etc/docker/registry/config.yml:z \
  -v registry_data:/var/lib/registry:Z \
  -v $PWD/certs:/etc/pki/certs:z \
  --restart=always \
  registry:2
```

Ahora podemos probar el inicio de sesión en el registro remoto utilizando las credenciales definidas previamente:

```bash
$ skopeo login -u admin -p p0dman4Dev0ps# --tls-verify=false localhost:5000
Login Succeeded!
```

Observa la opción `--tls-verify=false` para omitir la validación del certificado TLS. Dado que es un certificado autofirmado, debemos eludir las comprobaciones que producirían el mensaje de error `x509: certificate signed by unknown authority`.

Podemos intentar nuevamente eliminar la imagen enviada antes:

```bash
$ skopeo delete \
  --tls-verify=false \
  docker://localhost:5000/minimal_httpd
```

Esta vez, el comando se ejecutará correctamente ya que la función de eliminación se habilitó en el archivo de configuración.

Se puede utilizar un registro local para duplicar imágenes de un registro público externo. En la siguiente subsección, veremos un ejemplo de duplicación de registros utilizando nuestro registro local y un conjunto seleccionado de repositorios e imágenes.

#### Uso de un registro local para sincronizar repositorios

Duplicar imágenes y repositorios en un registro local puede ser muy útil en entornos desconectados. Esto también puede ser muy útil para mantener una copia asíncrona de imágenes seleccionadas y poder seguir descargándolas durante interrupciones de servicios públicos.

El siguiente ejemplo muestra una duplicación simple utilizando el comando `skopeo sync` con una lista de imágenes provista por un archivo YAML y nuestro registro local como destino:

```bash
$ skopeo sync \
  --src yaml --dest docker \
  --dest-tls-verify=false \
  kube_sync.yaml localhost:5000
```

El archivo YAML contiene una lista de las imágenes que componen un plano de control de Kubernetes para una versión específica. Nuevamente, aprovechamos las expresiones regulares para personalizar las imágenes a descargar (`Chapter09/kube_sync.yaml`):

```yaml
k8s.gcr.io:
  tls-verify: true
  images-by-tag-regex:
    kube-apiserver: ^v1\.32\..*
    kube-controller-manager: ^v1\.32\..*
    kube-proxy: ^v1\.32\..*
    kube-scheduler: ^v1\.32\..*
    coredns/coredns: ^v1\.9\..*
    etcd: 3\.5\.[0-9]*-[0-9]*
```

Al sincronizar un registro remoto y local, se pueden duplicar muchas capas en el proceso. Por este motivo, es importante supervisar el almacenamiento utilizado por el registro (`/var/lib/registry` en nuestro ejemplo) para evitar llenar el sistema de archivos.

Cuando el sistema de archivos se llena, eliminar imágenes más antiguas y no utilizadas con Skopeo no es suficiente, y es necesaria una acción adicional de recolección de elementos no utilizados (*garbage collection*) para liberar espacio. La siguiente subsección ilustra este proceso.

#### Administración de la recolección de elementos no utilizados (*Garbage Collection*)

Cuando se emite un comando de eliminación en un registro de contenedores, solo se eliminan los manifiestos de imagen que hacen referencia a un conjunto de blobs (que podrían ser capas u otros manifiestos), mientras se mantienen los blobs en el sistema de archivos.

Si un blob ya no está referenciado por ningún manifiesto, puede ser elegible para la recolección de elementos no utilizados por parte del registro. El proceso de recolección se administra con un comando dedicado, `registry garbage-collect`, emitido dentro del contenedor del registro. Este no es un proceso automático y debe ejecutarse manualmente o programarse.

En el siguiente ejemplo, ejecutaremos una recolección básica. El indicador `--dry-run` solo imprime los blobs elegibles a los que ya no hace referencia un manifiesto y, por lo tanto, se pueden eliminar de forma segura:

```bash
# podman exec -it local_registry \
  registry garbage-collect --dry-run \
  /etc/docker/registry/config.yml
```

Para eliminar los blobs, simplemente retira la opción `--dry-run`:

```bash
# podman exec -it local_registry \
  registry garbage-collect /etc/docker/registry/config.yml
```

La recolección de elementos no utilizados ayuda a mantener el registro libre de blobs no utilizados y ahorra espacio de almacenamiento. Por otro lado, debemos tener en cuenta que un blob no referenciado aún podría reutilizarse en el futuro para otra imagen. Si se elimina, podría ser necesario cargarlo nuevamente eventualmente.

---

### Resumen

En este capítulo, exploramos cómo interactuar con registros de contenedores, que son los servicios de almacenamiento fundamentales para nuestras imágenes. Comenzamos con una descripción de alto nivel de qué es un registro de contenedores y cómo funciona e interactúa con nuestros motores y herramientas de contenedores. Luego pasamos a una descripción más detallada de las diferencias entre registros públicos basados en la nube y registros privados, generalmente ejecutados de forma local (*on-premises*). Fue especialmente útil para comprender los beneficios y las limitaciones de ambos y ayudarnos a comprender el mejor enfoque para nuestras necesidades.

Para administrar imágenes de contenedores en registros, presentamos la herramienta Skopeo, que forma parte de la familia de herramientas complementarias de Podman, e ilustramos cómo se puede utilizar para copiar, sincronizar, eliminar o simplemente inspeccionar imágenes a través de registros, brindando a los usuarios un mayor grado de control sobre sus imágenes.

Finalmente, aprendimos cómo ejecutar un registro en contenedor local utilizando la imagen comunitaria oficial de Docker Registry v2. Después de mostrar un uso básico, profundizamos en detalles de configuración más avanzados mostrando cómo habilitar la autenticación, la eliminación de imágenes y el cifrado HTTPS. El registro local demostró ser útil para sincronizar imágenes locales así como registros remotos. El proceso de recolección de elementos no utilizados del registro se ilustró para mantener las cosas en orden dentro del almacén del registro.

Con el conocimiento adquirido en este capítulo, podrás administrar imágenes a través de registros e incluso instancias de registros locales con un mayor grado de conocimiento de lo que sucede en segundo plano. Los registros de contenedores son una parte crucial de una estrategia exitosa de adopción de contenedores y deben entenderse muy bien: con los conceptos de este capítulo en mente, también deberías poder comprender y diseñar las soluciones más adecuadas y obtener un control profundo sobre las herramientas para manipular imágenes.

Con este capítulo, también hemos completado la exploración de todas las tareas básicas relacionadas con la gestión de contenedores. Ahora podemos pasar a temas más avanzados, como la resolución de problemas y la supervisión de contenedores, que se tratan en el próximo capítulo.

---

### Lecturas adicionales

- [1] Especificación de Distribución OCI: [https://github.com/opencontainers/distribution-spec/blob/main/spec.md](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
- [2] Descripción de Bcrypt: [https://en.wikipedia.org/wiki/Bcrypt](https://en.wikipedia.org/wiki/Bcrypt)
- [3] Especificaciones de la API de Docker Registry v2: [https://docs.docker.com/registry/spec/api/](https://docs.docker.com/registry/spec/api/)
