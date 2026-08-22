# Parte 2: Construyendo tu Contenedor desde Cero con Buildah

## Capítulo 8: Elección de la Imagen Base del Contenedor

La forma más rápida y sencilla de aprender y adquirir experiencia con los contenedores es comenzar a trabajar con imágenes de contenedores precompiladas, como vimos en los capítulos anteriores. Después de profundizar en la gestión de contenedores, descubrimos que, a veces, el servicio disponible, su configuración o incluso la versión de la aplicación no es la que requiere nuestro proyecto. Luego, presentamos Buildah y su función para crear imágenes de contenedores personalizadas. En este capítulo, abordaremos otro tema importante que a menudo se cuestiona en proyectos comunitarios y empresariales: la elección de una imagen base de contenedor.

Elegir la imagen base de contenedor adecuada es una tarea importante en el viaje hacia los contenedores: una imagen base de contenedor es la capa subyacente del sistema operativo en la que se basará el servicio, la aplicación o el código de nuestro sistema. Debido a esto, debemos elegir una que se adapte a nuestras mejores prácticas con respecto a la seguridad y las actualizaciones.

En este capítulo, vamos a cubrir los siguientes temas principales:

- El formato de imagen de la Open Container Initiative (OCI)
- ¿De dónde vienen las imágenes de los contenedores?
- Fuentes confiables de imágenes de contenedores
- Presentación de la Universal Base Image (UBI)

---

### Requisitos técnicos

Para completar este capítulo, necesitarás una máquina con una instalación de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos de este libro se han ejecutado en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Tener una buena comprensión de los temas que cubrimos en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, te ayudará a asimilar fácilmente conceptos sobre imágenes de contenedores.

---

### El formato de imagen de la Open Container Initiative

Como describimos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*, en 2013, Docker se introdujo en el panorama de los contenedores y se volvió muy popular rápidamente.

A un alto nivel, el equipo de Docker introdujo el concepto de imágenes de contenedores y registros de contenedores, lo que supuso un cambio radical en las reglas del juego. Otro paso importante fue poder extraer los proyectos de containerd de Docker y donarlos a la Cloud Native Computing Foundation (CNCF). Esto motivó a la comunidad de código abierto a comenzar a trabajar seriamente en motores de contenedores que pudieran incorporarse a una capa de orquestación, como Kubernetes.

De manera similar, en 2015, Docker, con la ayuda de muchas otras empresas (Red Hat, AWS, Google, Microsoft, IBM y otras), inició la Open Container Initiative (OCI) bajo el paraguas de la Linux Foundation.

Estos colaboradores desarrollaron la Especificación de Tiempo de Ejecución (*runtime-spec*) y la Especificación de Imagen (*image-spec*) para describir cómo deberían crearse en el futuro la API y la arquitectura para los nuevos motores de contenedores.

Después de unos meses de trabajo, el equipo de OCI lanzó su primera implementación de un entorno de ejecución de contenedores que se adhería a sus especificaciones; el proyecto se denominó `runc`.

Vale la pena examinar la especificación de imagen de contenedor en detalle y revisar parte de la teoría detrás de la práctica que presentamos en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*.

La especificación define una imagen de contenedor OCI que consta de lo siguiente:

- **Manifiesto (*Manifest*)**: Contiene los metadatos de los contenidos y dependencias de la imagen. Esto también incluye la capacidad de identificar uno o más archivos de almacenamiento del sistema de archivos que se descomprimirán para obtener el sistema de archivos ejecutable final.
- **Índice de imagen (*Image index*, opcional)**: Representa una lista de manifiestos y descriptores que pueden proporcionar diferentes implementaciones de la imagen, según la plataforma de destino.
- **Conjunto de capas del sistema de archivos**: El conjunto real de capas que se deben fusionar para compilar el sistema de archivos del contenedor final.
- **Configuración**: Contiene toda la información requerida por el motor de tiempo de ejecución del contenedor para ejecutar efectivamente la aplicación, como argumentos y variables de entorno.

No profundizaremos en cada elemento de la Especificación de Imagen de OCI ya que está fuera de alcance, pero el Manifiesto de Imagen merece una mirada más cercana.

#### Manifiesto de Imagen OCI (*OCI Image Manifest*)

El Manifiesto de Imagen define un conjunto de capas y la configuración para una sola imagen de contenedor que se compila para una arquitectura y un sistema operativo específicos.

Exploremos los detalles del Manifiesto de Imagen OCI observando el siguiente ejemplo:

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

Aquí estamos utilizando las siguientes palabras clave:

- **schemaVersion**: Una propiedad que debe establecerse en un valor de 2. Esto garantiza la compatibilidad con versiones anteriores de Docker.
- **config**: Una propiedad que hace referencia a la configuración de una imagen a través de un resumen (*digest*):
  - **mediaType**: Esta propiedad define el formato de configuración real (solo uno actualmente).
- **layers**: Esta propiedad proporciona una matriz de objetos descriptores:
  - **mediaType**: En este caso, este descriptor debe ser uno de los tipos de medios permitidos para los descriptores de capa.
- **annotations**: Esta propiedad define metadatos adicionales para el Manifiesto de Imagen.

En resumen, el objetivo principal de la especificación es crear herramientas interoperables para compilar, transportar y preparar una imagen de contenedor para su ejecución.

La Especificación del Manifiesto de Imagen tiene tres objetivos principales:

1. Permitir el cálculo de hash para la configuración de la imagen, generando así un ID único.
2. Permitir imágenes multiarquitectura debido a su manifiesto de alto nivel (índice de imagen) que hace referencia a versiones específicas de la plataforma del Manifiesto de Imagen.
3. Poder traducir fácilmente la imagen del contenedor a la Especificación de Tiempo de Ejecución de OCI.

Ahora, aprendamos de dónde provienen estas imágenes de contenedores.

---

### ¿De dónde vienen las imágenes de los contenedores?

En los capítulos anteriores, utilizamos imágenes precompiladas para ejecutar, compilar o administrar un contenedor, pero ¿de dónde provienen estas imágenes de contenedores?

¿Cómo podemos profundizar en sus comandos de origen o en el Dockerfile/Containerfile utilizado para construirlas?

Bueno, como hemos mencionado anteriormente, Docker introdujo el concepto de imagen de contenedor y registro de contenedores para almacenar estas imágenes, incluso públicamente. El registro de contenedores más famoso es Docker Hub, pero después de la introducción de Docker, también se lanzaron otros registros de contenedores en la nube.

Podemos elegir entre los siguientes registros de contenedores en la nube:

- **Docker Hub**: Esta es la solución de registro alojada por Docker Inc. Este registro también aloja repositorios oficiales e imágenes verificadas por seguridad para algunos proyectos populares de código abierto.
- **Quay.io**: Esta es la solución de registro alojada que nació bajo la empresa CoreOS, aunque ahora forma parte de Red Hat. Ofrece repositorios públicos y privados, escaneo automatizado para fines de seguridad, compilaciones de imágenes e integración con repositorios públicos populares de Git.
- **Registros de distribuciones de Linux**: Las distribuciones populares de Linux suelen ser de base comunitaria, como Fedora Linux, o empresariales, como Red Hat Enterprise Linux (RHEL). Por lo general, ofrecen registros de contenedores públicos, aunque a menudo solo están disponibles para proyectos o paquetes que ya se han proporcionado como paquetes del sistema. Estos registros no están disponibles para los usuarios finales y son alimentados por los mantenedores de las distribuciones de Linux.
- **Registros de nubes públicas**: Amazon, Google, Microsoft y otros proveedores de nube pública ofrecen registros de contenedores públicos y privados para sus clientes.

Exploraremos estos registros con más detalle en el [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*.

Docker Hub, así como Quay.io, son registros públicos de contenedores donde podemos encontrar imágenes de contenedores que han sido creadas por cualquier persona. Estos registros están llenos de imágenes personalizadas útiles que podemos usar como puntos de partida para probar imágenes de contenedores de forma rápida y sencilla.

Simplemente descargar y ejecutar una imagen de contenedor no siempre es lo mejor que se puede hacer: podríamos toparnos con software muy antiguo y desactualizado que podría ser vulnerable a alguna vulnerabilidad pública conocida; o, peor aún, podríamos descargar y ejecutar código malicioso que podría comprometer toda nuestra infraestructura.

Por esta razón, Docker Hub y Quay.io suelen ofrecer funciones para destacar de dónde provienen dichas imágenes. Inspeccionémoslas.

#### Servicio de registro de contenedores Docker Hub

Como presentamos anteriormente, Docker Hub es el registro de contenedores más famoso disponible. Aloja múltiples imágenes de contenedores para productos comunitarios o empresariales.

Al mirar la página de detalles de una imagen de contenedor, podemos descubrir fácilmente toda la información requerida sobre ese proyecto y sus imágenes de contenedor. La siguiente captura de pantalla muestra la página de Docker Hub de Alpine Linux:

*Figura 8.1 – Imagen de contenedor de Alpine Linux en Docker Hub*

Como puedes ver, en la parte superior de la página, podemos encontrar información útil, las etiquetas más recientes, las arquitecturas admitidas y enlaces útiles a la documentación del proyecto y al sistema de informes de problemas.

En la página de Docker Hub, podemos encontrar la etiqueta **Official Image**, justo después del nombre de la imagen, cuando esa imagen es parte del programa de Imágenes Oficiales de Docker. Las imágenes de este programa, que se reportan como oficiales, son curadas directamente por el equipo de Docker en colaboración con los mantenedores de los proyectos ascendentes (*upstream*).

> [!NOTE]
> Si deseas consultar esta página con mayor profundidad, dirígete en tu navegador web a [https://hub.docker.com/_/alpine](https://hub.docker.com/_/alpine).

Otra característica importante que ofrece Docker Hub (no solo para imágenes oficiales) es la capacidad de mirar dentro del Dockerfile que se utilizó para crear una determinada imagen.

Si hacemos clic en una de las etiquetas disponibles en la página de la imagen del contenedor, podemos ver fácilmente el Dockerfile de esa etiqueta de imagen de contenedor.

Al hacer clic en la etiqueta llamada `edge` en esa página, se nos redirigirá a la página de GitHub del proyecto docker-alpine, que se define como el siguiente Dockerfile: [https://github.com/alpinelinux/docker-alpine/blob/edge/x86_64/Dockerfile](https://github.com/alpinelinux/docker-alpine/blob/edge/x86_64/Dockerfile).

Siempre debemos buscar y preferir imágenes oficiales. Si una imagen oficial no está disponible o no se ajusta a nuestras necesidades, entonces debemos inspeccionar el Dockerfile que publicó el creador del contenido, así como la imagen del contenedor.

Docker Hub también incluye un servicio adicional llamado Scout. Es una herramienta que ayuda a los desarrolladores a encontrar y corregir problemas de seguridad en sus imágenes de Docker. Escanea imágenes en busca de vulnerabilidades, brinda consejos de corrección y se integra con canalizaciones de CI/CD para garantizar que la seguridad se aborde durante todo el proceso de desarrollo.

> [!IMPORTANT]
> Al momento de escribir este libro, Docker Hub tiene varias limitaciones para usuarios autenticados no de pago y también para usuarios no autenticados. Una limitación común es la cantidad de imágenes descargadas (*pulled*) por hora por usuario. Puedes obtener más información en la siguiente URL: [https://docs.docker.com/docker-hub/download-rate-limit/](https://docs.docker.com/docker-hub/download-rate-limit/).

#### Servicio de registro de contenedores Quay.io

Quay es un servicio de registro de contenedores que fue adquirido por CoreOS en 2014 y ahora forma parte del ecosistema de Red Hat. Quay.io es el servicio gestionado, ofrecido como SaaS, del proyecto de código abierto Quay.

El registro en línea permite a sus usuarios ser más cautelosos una vez que han elegido una imagen de contenedor al proporcionar software de escaneo de seguridad.

Quay, y también Quay.io, adopta el proyecto Clair, un escáner de vulnerabilidades de contenedores líder que muestra informes en la página web de etiquetas del repositorio, como se muestra en la siguiente captura de pantalla:

*Figura 8.2 – Página de escaneo de seguridad de vulnerabilidades de Quay*

En esta página, podemos hacer clic en **SECURITY SCAN** para inspeccionar los detalles de ese escaneo de seguridad. Si deseas obtener más información sobre esta función, visita [https://quay.io/repository/openshift-release-dev/ocp-release?tab=tags](https://quay.io/repository/openshift-release-dev/ocp-release?tab=tags).

Como hemos visto, el uso de un registro público que ofrece a cada usuario la función de escaneo de seguridad podría ayudar a garantizar que elijamos la variante correcta y más segura de la imagen de contenedor que estamos buscando.

Al momento de escribir este libro, Quay.io no tiene límites en la cantidad de imágenes de contenedores descargadas; el nivel gratuito ofrece cinco repositorios privados y repositorios públicos ilimitados.

#### Registro de Contenedores de Red Hat (*Red Hat Container Registry*)

Red Hat Container Registry es el registro de contenedores predeterminado para los usuarios de RHEL y Red Hat OpenShift Container Platform (OCP). La interfaz web de este registro es accesible públicamente para cualquier usuario, esté autenticado o no, aunque casi todas las imágenes que se proporcionan están reservadas para usuarios de pago (usuarios de RHEL u OCP).

Estamos hablando de este registro porque combina todas las funciones de las que hablamos anteriormente. Este registro ofrece lo siguiente a sus usuarios:

- Imágenes de contenedores oficiales de Red Hat.
- Fuentes de Containerfile/Dockerfile para inspeccionar el contenido de la imagen.
- Informes de seguridad (índice) sobre cada imagen de contenedor distribuida.

La siguiente captura de pantalla muestra cómo se ve esta información en la página del Catálogo del Ecosistema de Red Hat (*Red Hat Ecosystem Catalog*):

*Figura 8.3 – Página de descripción de la imagen de contenedor de Mariadb en Red Hat Ecosystem Catalog*

Como podemos ver, la página muestra la descripción de la imagen del contenedor que hemos seleccionado (base de datos MariaDB), la versión, las arquitecturas disponibles y varias etiquetas que se pueden seleccionar en el menú desplegable correspondiente. Algunas pestañas también mencionan las palabras clave que nos interesan: **Security** y **Dockerfile**.

Al hacer clic en la pestaña **Security**, podemos ver el estado del escaneo de vulnerabilidades que se ejecutó para esa etiqueta de imagen, como se muestra en la siguiente captura de pantalla:

*Figura 8.4 – Página de seguridad de la imagen de contenedor de Mariadb en Red Hat Ecosystem Catalog*

Como podemos ver, al momento de escribir este libro, para esta última etiqueta de imagen, ya se ha identificado una vulnerabilidad de seguridad que afecta a tres paquetes. A la derecha, podemos encontrar el ID del aviso de Red Hat (*Red Hat Advisory ID*), que está vinculado a las Vulnerabilidades y Exposiciones Comunes (CVE) públicas.

Al hacer clic en la pestaña **Dockerfile**, podemos ver el Containerfile de origen que se utilizó para compilar esa imagen de contenedor:

*Figura 8.5 – Página de Dockerfile de la imagen de contenedor de Mariadb en Red Hat Ecosystem Catalog*

Podemos consultar el Containerfile de origen que se utilizó para compilar la imagen de contenedor que vamos a descargar y ejecutar. Esta es una gran característica a la que podemos acceder haciendo clic en la misma página de descripción de la imagen del contenedor que estamos buscando.

Si deseas ver más de cerca la imagen del contenedor anterior, puedes visitar esta URL: [https://catalog.redhat.com/software/containers/rhel9/mariadb-105/61a6084dbfd4a5234d596220](https://catalog.redhat.com/software/containers/rhel9/mariadb-105/61a6084dbfd4a5234d596220).

Otra imagen de contenedor interesante disponible en el registro empresarial de Red Hat es la UBI:

*Figura 8.6 – Página de Dockerfile de la imagen de contenedor UBI 9 en Red Hat Ecosystem Catalog*

UBI son las siglas de **Universal Base Image**. Es una iniciativa lanzada por Red Hat que permite a todos los usuarios (clientes de Red Hat o no) utilizar imágenes de contenedores de Red Hat. Esto permite que el ecosistema de Red Hat se expanda aprovechando todos los servicios mencionados anteriormente que ofrece el Catálogo del Ecosistema de Red Hat, así como aprovechando los paquetes actualizados que provienen directamente de Red Hat. Según la versión de lanzamiento de RHEL, encontrarás una versión de lanzamiento de UBI correspondiente.

Puedes echar un vistazo a la imagen del contenedor visitando esta URL: [https://catalog.redhat.com/software/containers/ubi9/ubi/615bcf606feffc5384e8452e](https://catalog.redhat.com/software/containers/ubi9/ubi/615bcf606feffc5384e8452e).

Hablaremos más sobre UBI y sus imágenes de contenedores más adelante en este capítulo.

---

### Fuentes confiables de imágenes de contenedores

En la sección anterior, definimos el papel central del registro de imágenes como fuente de información confiable para imágenes válidas y utilizables. En esta sección, queremos enfatizar la importancia de adoptar imágenes confiables que provengan de fuentes confiables.

Una imagen OCI se utiliza para empaquetar binarios y tiempos de ejecución en un sistema de archivos estructurado con el propósito de entregar un servicio específico. Cuando descargamos esa imagen y la ejecutamos en nuestros sistemas sin ningún tipo de control, confiamos implícitamente en que el autor no ha manipulado su contenido utilizando componentes maliciosos. Pero hoy en día, la confianza es algo que no se puede conceder tan fácilmente.

Como veremos en el [Capítulo 10](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/10), *Aseguramiento de Contenedores*, existen muchos casos de uso de ataques y comportamientos maliciosos que se pueden realizar desde un contenedor: escalada de privilegios, filtración de datos y mineros de criptomonedas son solo algunos ejemplos. Estos comportamientos pueden amplificarse cuando los contenedores que se ejecutan dentro de clústeres de Kubernetes (muchos miles de clústeres) pueden generar Pods maliciosos en toda la infraestructura fácilmente.

Para ayudar a los equipos de seguridad a mitigar esto, MITRE Corporation publica periódicamente matrices MITRE ATT&CK para identificar todas las posibles estrategias de ataque y sus técnicas relacionadas, con casos de uso de la vida real, y sus mejores prácticas de detección y mitigación. Una de estas matrices está dedicada a los contenedores, donde se implementan muchas técnicas basadas en imágenes inseguras, donde se pueden llevar a cabo con éxito comportamientos maliciosos.

> [!IMPORTANT]
> Debes preferir las imágenes que provengan de un registro que admita escaneos de vulnerabilidades. Si los resultados del escaneo están disponibles, verifícalos cuidadosamente y evita usar imágenes que detecten vulnerabilidades críticas.

Con esto en mente, ¿cuál es el primer paso para crear una infraestructura nativa de la nube segura? La respuesta es elegir imágenes que solo provengan de fuentes confiables, y lo primero que se debe hacer es configurar registros confiables y patrones para bloquear los no permitidos. Cubriremos esto en la siguiente subsección.

#### Administración de registros confiables

Como se muestra en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, en la sección *Preparación de tu entorno*, Podman puede administrar registros confiables con archivos de configuración.

El archivo `/etc/containers/registries.conf` (anulado por el archivo `$HOME/.config/containers/registries.conf` relacionado con el usuario, si está presente) administra una lista de registros confiables a los que Podman puede contactar de manera segura para buscar y descargar imágenes.

Veamos un ejemplo de este archivo:

```toml
unqualified-search-registries = ["docker.io", "quay.io"]

[[registry]]
location = "registry.example.com:5000"
insecure = false
```

Este archivo nos ayuda a definir los registros confiables que Podman puede utilizar, por lo que merece un análisis detallado.

Podman acepta tanto imágenes no calificadas (*unqualified*) como totalmente calificadas (*fully qualified*). La diferencia es bastante simple y se puede ilustrar de la siguiente manera:

- Una **imagen totalmente calificada** incluye un FQDN del servidor de registro, el espacio de nombres, el nombre de la imagen y la etiqueta. Por ejemplo, `docker.io/library/nginx:latest` es una imagen totalmente calificada. Tiene un nombre completo que no se puede confundir con ninguna otra imagen de Nginx.
- Una **imagen no calificada** solo incluye el nombre de la imagen. Por ejemplo, la imagen `nginx` puede tener múltiples instancias en los registros buscados. La mayoría de las imágenes que resultan del comando básico `podman search nginx` no serán oficiales y deben analizarse en detalle para garantizar que sean confiables. La salida se puede filtrar mediante el indicador `OFFICIAL` y por la cantidad de estrellas (cuantas más, mejor).

La primera configuración global del archivo de configuración de registros es la matriz `unqualified-search-registries`, que define la lista de búsqueda de registros para imágenes no calificadas. Cuando el usuario ejecuta el comando `podman search <image_name>`, Podman buscará en los registros definidos en esta lista.

Al eliminar un registro de la lista, Podman dejará de buscar en ese registro. Sin embargo, Podman aún podrá descargar una imagen totalmente calificada de un registro externo.

Para administrar registros individuales y crear patrones de coincidencia para imágenes específicas, podemos usar las tablas TOML (*Tom's Obvious, Minimal Language*) `[[registry]]`. Las configuraciones principales de estas tablas son las siguientes:

- **prefix**: Se utiliza para definir los nombres de las imágenes y puede admitir varios formatos. En general, podemos definir imágenes siguiendo el patrón `host[:port]/namespace[/_namespace_…]/repo(:_tag|@digest)`, aunque se pueden aplicar patrones más simples, como `host[:port]`, `host[:port]/namespace` e incluso `[*.]host`. Siguiendo este enfoque, los usuarios pueden definir un prefijo genérico para un registro o un prefijo más detallado para que coincida con una imagen o etiqueta específica. Dada una imagen totalmente calificada, si dos tablas `[[registry]]` tienen un prefijo con una coincidencia parcial, se utilizará el patrón de coincidencia más largo.
- **insecure**: Es un booleano (`true` o `false`) que permite conexiones HTTP no cifradas o conexiones TLS basadas en certificados no confiables.
- **blocked**: Es un booleano (`true` o `false`) que se utiliza para definir registros bloqueados. Si se establece en `true`, los registros o imágenes que coinciden con el prefijo se bloquean.
- **location**: Este campo define la ubicación del registro. De forma predeterminada, es igual a `prefix`, pero puede tener un valor diferente. En ese caso, un patrón que coincida con un espacio de nombres de prefijo personalizado se resolverá en el valor de ubicación.

Junto con la tabla principal `[[registry]]`, podemos definir una matriz de tablas TOML `[[registry.mirror]]` para proporcionar rutas alternativas al registro principal o al espacio de nombres del registro.

Cuando se proporcionan varios espejos (*mirrors*), Podman buscará a través de ellos primero y luego recurrirá a la ubicación que se define en la tabla principal `[[registry]]`.

El siguiente ejemplo amplía el anterior al definir una entrada de registro con espacio de nombres y su espejo:

```toml
unqualified-search-registries = ["docker.io", "quay.io"]

[[registry]]
location = "registry.example.com:5000/foo"
insecure = false

[[registry.mirror]]
location = "mirror1.example.com:5000/bar"

[[registry.mirror]]
location = "mirror2.example.com:5000/bar"
```

De acuerdo con este ejemplo, si un usuario intenta descargar la imagen etiquetada como `registry.example.com:5000/foo/app:latest`, Podman probará `mirror1.example.com:5000/bar/app:latest`, luego `mirror2.example.com:5000/bar/app:latest`, y recurrirá a `registry.example.com:5000/foo/app:latest` en caso de que ocurra una falla.

El uso de un prefijo proporciona aún más flexibilidad. En el siguiente ejemplo, todas las imágenes que coincidan con `example.com/foo` se redirigirán a ubicaciones espejo y recurrirán a la ubicación principal al final:

```toml
unqualified-search-registries = ["docker.io", "quay.io"]

[[registry]]
prefix = "example.com/foo"
location = "registry.example.com:5000/foo"
insecure = false

[[registry.mirror]]
location = "mirror1.example.com:5000/bar"

[[registry.mirror]]
location = "mirror2.example.com:5000/bar"
```

En este ejemplo, cuando descargamos la imagen `example.com/foo/app:latest`, Podman intentará `mirror1.example.com:5000/bar/app:latest`, seguido de `mirror2.example.com:5000/bar/app:latest` y `registry.example.com:5000/foo/app:latest`.

Es posible utilizar la duplicación (*mirroring*) de una forma más avanzada, como reemplazar los registros públicos por espejos privados en entornos desconectados. El siguiente ejemplo reasigna los registros `docker.io` y `quay.io` a un espejo privado con diferentes espacios de nombres:

```toml
[[registry]]
prefix="quay.io"
location="mirror-internal.example.com/quay"

[[registry]]
prefix="docker.io"
location="mirror-internal.example.com/docker"
```

> [!IMPORTANT]
> Los registros espejo deben mantenerse actualizados con los repositorios duplicados. Por esta razón, los administradores o los equipos de SRE deben implementar una política de sincronización de imágenes para mantener los repositorios actualizados.

Finalmente, vamos a aprender a bloquear una fuente que no se considera confiable. Este comportamiento podría afectar a una sola imagen, un espacio de nombres o un registro completo.

El siguiente ejemplo le indica a Podman que no busque ni descargue imágenes de un registro bloqueado:

```toml
[[registry]]
location = "registry.rogue.io"
blocked = true
```

Es posible refinar la política de bloqueo pasando un espacio de nombres específico sin bloquear todo el registro. En el siguiente ejemplo, se bloquea cada búsqueda o descarga de imágenes que coincida con el patrón de espacio de nombres `quay.io/foo` definido en el campo `prefix`:

```toml
[[registry]]
prefix = "quay.io/foo/"
location = "quay.io"
blocked = true
```

De acuerdo con este patrón, si el usuario intenta descargar una imagen llamada `quay.io/foo/nginx:latest` o `quay.io/foo/httpd:v2.4`, el prefijo coincide y la descarga se bloquea. No se produce ninguna acción de bloqueo cuando se descarga la imagen `quay.io/bar/fedora:latest`.

Los usuarios también pueden definir una regla de bloqueo muy específica para una sola imagen o incluso una sola etiqueta utilizando el mismo enfoque que se describió para los espacios de nombres. El siguiente ejemplo bloquea una etiqueta de imagen específica:

```toml
[[registry]]
prefix = "internal-registry.example.com/dev/app:v0.1"
location = "internal-registry.example.com"
blocked = true
```

Es posible combinar muchas reglas de bloqueo y agregar tablas espejo encima de ellas.

> [!IMPORTANT]
> En una infraestructura compleja con muchas máquinas ejecutando Podman (por ejemplo, estaciones de trabajo de desarrolladores), una idea inteligente sería mantener actualizado el archivo de configuración del registro mediante herramientas de administración de configuración y aplicar de manera declarativa los filtros del registro.

Los nombres de imágenes totalmente calificados (FQN) pueden llegar a ser bastante largos si sumamos el FQDN del registro, los espacios de nombres, el repositorio y las etiquetas.

Si bien el uso de FQN como `quay.io/podman/stable:latest` es el estándar de oro para la seguridad y la automatización, a menudo resultan engorrosos de escribir manualmente. Para cerrar la brecha entre la conveniencia y la seguridad, Podman utiliza un sistema de alias de nombres cortos (*short-name aliasing*).

Normalmente, cuando proporcionas un nombre no calificado (por ejemplo, `podman pull fedora`), Podman debe iterar a través de una lista de registros definidos en la tabla `[registries.search]` de tu configuración. Esto puede ser lento y, si no se configura correctamente, potencialmente inseguro.

Los alias cambian este comportamiento al acortar la búsqueda. Si un nombre corto coincide con una entrada en una tabla de alias, Podman lo resuelve inmediatamente en el nombre específico y totalmente calificado proporcionado en la asignación, omitiendo por completo la lista de registros de búsqueda.

En Fedora y RHEL, no tienes que crear esta lista desde cero. El sistema viene preconfigurado con una extensa biblioteca de asignaciones confiables ubicada en la siguiente ruta:

```text
/etc/containers/registries.conf.d/000-shortnames.conf
```

Este archivo es mantenido por la comunidad y los proveedores de sistemas operativos para garantizar que los nombres comunes apunten a sus fuentes ascendentes más lógicas y seguras. Por ejemplo, una entrada en este archivo podría verse de la siguiente manera:

```toml
[aliases]
"fedora" = "registry.fedoraproject.org/fedora"
"ubi8" = "registry.access.redhat.com/ubi8"
"alpine" = "docker.io/library/alpine"
```

Cuando un alias coincide con un nombre corto, se utiliza inmediatamente sin buscar en los registros definidos en la lista `unqualified-search-registries`.

> [!IMPORTANT]
> Podemos crear archivos personalizados dentro de la carpeta `/etc/containers/registries.conf.d/` para definir alias sin sobrecargar el archivo de configuración principal.

Si bien los alias simplifican la administración, tienen límites específicos:

- **Sin etiquetas ni resúmenes**: Los alias resuelven la ruta del repositorio, no una versión específica. Por ejemplo, si asignas el alias `my-app` a `quay.io/org/my-app`, ejecutar `podman pull my-app:v2` se resolverá correctamente en `quay.io/org/my-app:v2`. El alias maneja el prefijo, mientras que la etiqueta permanece dinámica.
- **Precedencia**: Los alias locales definidos por el usuario (a menudo ubicados en `$HOME/.config/containers/registries.conf.d/`) normalmente anularán los alias globales del sistema que se encuentran en `/etc/containers/`.

Con eso, hemos aprendido cómo administrar fuentes confiables y bloquear imágenes, registros o espacios de nombres no deseados. Esta es una mejor práctica de seguridad, pero no nos exime de la responsabilidad de elegir una imagen válida que se adapte a nuestras necesidades, sea confiable y tenga la menor superficie de ataque posible. Esto también se aplica cuando creamos una nueva aplicación, donde las imágenes base deben ser ligeras y seguras. Las imágenes UBI de Red Hat pueden ser una solución útil para este problema.

---

### Presentación de la Universal Base Image

Cuando se trabaja en entornos empresariales, muchos usuarios y empresas adoptan RHEL como el sistema operativo de elección para ejecutar cargas de trabajo de manera confiable y segura. Las imágenes de contenedores basadas en RHEL también están disponibles y aprovechan el mismo control de versiones de paquetes que la versión del sistema operativo. Todas las actualizaciones de seguridad que se publican para RHEL se aplican inmediatamente a las imágenes OCI, lo que las convierte en imágenes seguras y confiables con las que crear aplicaciones de nivel de producción.

Desafortunadamente, las imágenes de RHEL no están disponibles públicamente sin una suscripción a Red Hat. Los usuarios que han activado una suscripción válida pueden usarlas libremente en sus sistemas RHEL y crear imágenes personalizadas sobre ellas, pero no son redistribuibles libremente sin romper el acuerdo empresarial de Red Hat.

Entonces, ¿por qué preocuparse? Hay muchas imágenes de uso común que pueden reemplazarlas. Esto es cierto, pero cuando se trata de confiabilidad y seguridad, muchas empresas eligen ceñirse a una solución de nivel empresarial, y los contenedores no son una excepción.

Por estas razones, y para abordar las limitaciones de redistribución de las imágenes de RHEL, Red Hat creó UBI. Las imágenes UBI son libremente redistribuibles; se pueden utilizar para compilar aplicaciones en contenedores, middleware y utilidades; y Red Hat las mantiene y actualiza constantemente.

Las imágenes UBI se basan en las versiones de RHEL actualmente compatibles. Al momento de escribir este libro, están disponibles las imágenes UBI 8, UBI 9 y UBI 10, basadas en RHEL 8, RHEL 9 y RHEL 10, respectivamente. En general, las imágenes UBI pueden considerarse un subconjunto del sistema operativo RHEL.

Todas las imágenes UBI están disponibles en el registro público de Red Hat ([registry.access.redhat.com](https://registry.access.redhat.com/)) y en Docker Hub ([docker.io](https://docker.io/)).

Actualmente existen cuatro variantes diferentes de imágenes UBI, cada una especializada para un caso de uso particular:

- **Standard**: Esta es la imagen estándar de UBI. Tiene la mayor cantidad de funciones y disponibilidad de paquetes.
- **Minimal**: Esta es una versión reducida de la imagen estándar con una gestión de paquetes minimalista.
- **Micro**: Esta es una versión de UBI con la menor huella posible, sin un administrador de paquetes.
- **Init**: Esta es una imagen de UBI que incluye el sistema init systemd para que puedas administrar la ejecución de múltiples servicios en un solo contenedor.

Todas ellas son de uso y redistribución gratuitos dentro de imágenes personalizadas. A diferencia de las imágenes estándar de RHEL, UBI se puede redistribuir libremente, lo que permite a los desarrolladores crear aplicaciones sobre una base confiable y compartirlas en cualquier lugar, incluidos los registros públicos, sin necesidad de una suscripción a Red Hat. Además de eso, UBI cierra la brecha entre el desarrollo y la producción; puedes compilar en una computadora portátil con Fedora o en una canalización de CI/CD en la nube e implementar en un entorno de producción de RHEL u OpenShift sin problemas de compatibilidad.

Describamos cada una en detalle, comenzando con la imagen UBI Standard.

#### La imagen UBI Standard

La imagen UBI Standard es la versión de imagen UBI más completa y la más cercana a las imágenes estándar de RHEL. Incluye el administrador de paquetes `yum`, que está disponible en RHEL, y se puede personalizar instalando los paquetes que están disponibles en sus repositorios de software dedicados, es decir, `ubi-8-baseos` y `ubi-8-appstream`.

El siguiente ejemplo muestra un Dockerfile/Containerfile que utiliza una imagen UBI 8 estándar para compilar un servidor httpd mínimo:

```dockerfile
FROM registry.access.redhat.com/ubi8
# Update image and install httpd
RUN yum update -y && yum install -y httpd && yum clean all
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

La imagen UBI Standard fue diseñada para aplicaciones y paquetes genéricos que están disponibles en RHEL y ya incluye una lista seleccionada de herramientas básicas del sistema (incluidas `curl`, `tar`, `vi`, `sed` y `gzip`) y bibliotecas OpenSSL, manteniendo un tamaño reducido (alrededor de 230 MiB): menos paquetes significan imágenes más ligeras y una superficie de ataque menor.

En el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), *Conoce Buildah – Construcción de Contenedores desde Cero*, aprendimos cómo compilar una nueva imagen de contenedor a partir de un Containerfile o un Dockerfile, por lo que puedes aprovechar Buildah para probar el ejemplo anterior.

Si la imagen UBI Standard todavía se considera demasiado grande, UBI Minimal puede ser una buena opción.

#### La imagen UBI Minimal

La imagen UBI Minimal es una versión reducida de UBI Standard y fue diseñada para aplicaciones autónomas y sus tiempos de ejecución (Python, Ruby, Node.js, etc.). Por esta razón, es de menor tamaño, tiene una pequeña selección de paquetes y no incluye el administrador de paquetes `yum`; este ha sido reemplazado por una herramienta mínima llamada `microdnf`. La imagen UBI Minimal es más pequeña que UBI Standard y tiene aproximadamente la mitad de su tamaño.

El siguiente ejemplo muestra un Dockerfile/Containerfile que utiliza una imagen UBI 8 minimal para compilar un servidor web Python de prueba de concepto:

```dockerfile
# Based on the UBI8 Minimal image
FROM registry.access.redhat.com/ubi8-minimal
# Upgrade and install Python 3
RUN microdnf upgrade && microdnf install python3
# Copy source code
COPY entrypoint.sh http_server.py /
# Expose the default httpd port 80
EXPOSE 8080
# Configure the container entrypoint
ENTRYPOINT ["/entrypoint.sh"]
# Run the httpd
CMD ["/usr/bin/python3", "-u", "/http_server.py"]
```

Al observar el código fuente del servidor web Python que ejecuta el contenedor, podemos ver que el controlador del servidor web imprime una cadena `Hello World!` cuando se recibe una solicitud HTTP GET. El servidor también gestiona la terminación de señales utilizando el módulo `signal` de Python, lo que permite que el contenedor se detenga de forma controlada:

```python
#!/usr/bin/python3
import http.server
import socketserver
import logging
import sys
import signal
from http import HTTPStatus

port = 8080
message = b'Hello World!\n'

logging.basicConfig(
    stream = sys.stdout,
    level = logging.INFO
)

def signal_handler(signum, frame):
    sys.exit(0)

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(HTTPStatus.OK)
        self.end_headers()
        self.wfile.write(message)

if __name__ == "__main__":
    signal.signal(signal.SIGTERM, signal_handler)
    signal.signal(signal.SIGINT, signal_handler)
    try:
        httpd = socketserver.TCPServer(('', port), Handler)
        logging.info("Serving on port %s", port)
        httpd.serve_forever()
    except SystemExit:
        httpd.shutdown()
        httpd.server_close()
```

Finalmente, el ejecutable de Python es llamado por un script de entrypoint mínimo:

```bash
#!/bin/bash
set -e
exec $@
```

El script inicia el comando que pasa la matriz en la instrucción `CMD`. Además, observa la opción `-u` que se pasa al ejecutable de Python en la matriz de comandos. Esto habilita la salida sin búfer (*unbuffered output*) y hace que el contenedor imprima los registros de acceso en tiempo real.

Intentemos compilar y ejecutar el contenedor para ver qué sucede:

```bash
$ buildah build -t python_httpd .
$ podman run -p 8080:8080 python_httpd
INFO:root:Serving on port 8080
```

Con eso, nuestro servidor http mínimo de Python está listo para operar y servir respuestas `Hello World!`. Puedes encontrar todos estos ejemplos en el repositorio de GitHub del libro.

UBI Minimal funciona mejor para este tipo de casos de uso. Sin embargo, puede ser necesaria una imagen aún más pequeña. Este es el caso de uso perfecto para la imagen UBI Micro.

#### La imagen UBI Micro

La imagen UBI Micro es la última incorporación a la familia UBI. Su idea básica es proporcionar una imagen *distroless* y un administrador de paquetes reducido, sin todos los paquetes innecesarios, para proporcionar una imagen muy pequeña que también pueda ofrecer una superficie de ataque mínima. Reducir la superficie de ataque es necesario para lograr imágenes seguras y mínimas que sean más complejas de explotar. Además de eso, el tamaño reducido también minimiza la cantidad de datos que se deben descargar al usar la imagen por primera vez o después de una actualización.

La imagen UBI 8 Micro es excelente en compilaciones multietapa, donde la primera etapa crea los artefactos terminados y la segunda etapa los copia dentro de la imagen final. El siguiente ejemplo muestra un Dockerfile/Containerfile multietapa básico donde se compila una aplicación mínima de Golang dentro de un contenedor UBI Standard mientras que el artefacto final se copia dentro de una imagen UBI Micro:

```dockerfile
# Builder image
FROM registry.access.redhat.com/ubi8-minimal AS builder
# Install Golang packages
RUN microdnf upgrade && \
    microdnf install golang && \
    microdnf clean all
# Copy files for build
COPY go.mod /go/src/hello-world/
COPY main.go /go/src/hello-world/
# Set the working directory
WORKDIR /go/src/hello-world
# Download dependencies
RUN go get -d -v ./...
# Install the package
RUN go build -v ./...
# Runtime image
FROM registry.access.redhat.com/ubi8/ubi-micro:latest
COPY --from=builder /go/src/hello-world/hello-world /
EXPOSE 8080
CMD ["/hello-world"]
```

La salida de la compilación da como resultado una imagen que tiene un tamaño aproximado de 45 MB.

La imagen UBI Micro no tiene un administrador de paquetes integrado, pero aún es posible instalar paquetes adicionales utilizando los comandos nativos de Buildah. Esto funciona eficazmente en un sistema RHEL, donde están instalados todos los certificados GPG de Red Hat.

El siguiente ejemplo muestra un script de compilación que se puede ejecutar en RHEL 8. Su propósito es instalar paquetes de Python adicionales utilizando el administrador de paquetes `yum` del host, sobre una imagen UBI Micro:

```bash
#!/bin/bash
set -euo pipefail

if [ $UID -ne 0 ]; then
    echo "This script must be run as root"
    exit 1
fi

container=$(buildah from registry.access.redhat.com/ubi8/ubi-micro)
mount=$(buildah mount $container)

yum install -y \
    --installroot $mount \
    --setopt install_weak_deps=false \
    --nodocs \
    --noplugins \
    --releasever 8 \
    python3

yum clean all --installroot $mount
buildah umount $container
buildah commit $container micro_python3
```

Ten en cuenta que el comando `yum install` se ejecuta pasando la opción `--installroot $mount`, que le dice al instalador que use el punto de montaje del contenedor de trabajo como la raíz temporal para instalar los paquetes.

Las imágenes UBI Minimal y UBI Micro son excelentes para implementar arquitecturas de microservicios donde necesitamos orquestar múltiples contenedores juntos, cada uno ejecutando un microservicio específico.

Ahora, veamos la imagen UBI Init, que nos permite coordinar la ejecución de múltiples servicios dentro de un contenedor.

#### La imagen UBI Init

Un patrón común en el desarrollo de contenedores es crear imágenes altamente especializadas con un solo componente ejecutándose dentro de ellas.

Para implementar aplicaciones multicapa, como aquellas con una interfaz, middleware y un backend, la mejor práctica es crear y orquestar múltiples contenedores, cada uno ejecutando un componente específico. El objetivo es tener contenedores mínimos y muy especializados, cada uno ejecutando su propio servicio/proceso mientras sigue la filosofía *Keep It Simple, Stupid* (KISS), que se ha llevado a cabo en los sistemas UNIX desde sus inicios.

A pesar de ser excelente para la mayoría de los casos de uso, este enfoque no siempre se adapta a algunos escenarios especiales donde muchos procesos deben orquestarse juntos. Un ejemplo es cuando necesitamos compartir todos los namespaces de contenedores entre procesos, o cuando simplemente queremos una sola imagen integral (*uber image*).

Las imágenes de contenedores normalmente se crean sin un sistema init, y el proceso que se ejecuta dentro del contenedor (invocado por la instrucción `CMD`) generalmente obtiene el PID 1.

Por esta razón, Red Hat introdujo la imagen UBI Init, que ejecuta un proceso de inicio systemd mínimo dentro del contenedor, lo que permite que se ejecuten múltiples unidades de systemd gobernadas por el proceso systemd con un PID de 1.

La imagen UBI Init es ligeramente más pequeña que la imagen Standard pero tiene más paquetes disponibles que la imagen Minimal.

El `CMD` predeterminado se establece en `/sbin/init`, que corresponde al proceso systemd. systemd ignora las señales `SIGTERM` y `SIGKILL`, que utiliza Podman para detener los contenedores en ejecución. Por esta razón, la imagen está configurada para enviar señales `SIGRTMIN+3` para la terminación pasando la instrucción `STOPSIGNAL SIGRTMIN+3` dentro del Dockerfile de la imagen.

El siguiente ejemplo muestra un Dockerfile/Containerfile que instala el paquete httpd y configura una unidad systemd para ejecutar el servicio httpd:

```dockerfile
FROM registry.access.redhat.com/ubi8/ubi-init
RUN yum -y install httpd && \
    yum clean all && \
    systemctl enable httpd
RUN echo "Successful Web Server Test" > /var/www/html/index.html
RUN mkdir /etc/systemd/system/httpd.service.d/ && \
    echo -e '[Service]\nRestart=always' > /etc/systemd/system/httpd.service.d/httpd.conf
EXPOSE 80
CMD [ "/sbin/init" ]
```

Observa la instrucción `RUN`, donde creamos la carpeta `/etc/systemd/system/httpd.service.d/` y el archivo de unidad de systemd. Este ejemplo mínimo podría reemplazarse con una copia de archivos de unidad editados previamente, lo cual es particularmente útil cuando se deben crear múltiples servicios.

Podemos compilar y ejecutar la imagen e inspeccionar el comportamiento del sistema init dentro del contenedor usando el comando `ps`:

```bash
$ buildah build -t init_httpd .
$ podman run -d --name httpd_init -p 8080:80 init_httpd
$ podman exec -ti httpd_init /bin/bash
[root@b4fb727f1907 /]# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.1  0.0  89844  9404 ?        Ss   10:30   0:00 /sbin/init
root          10  0.0  0.0  95552 10636 ?        Ss   10:30   0:00 /usr/lib/systemd/systemd-journald
root          20  0.1  0.0 258068 10700 ?        Ss   10:30   0:00 /usr/sbin/httpd -DFOREGROUND
dbus          21  0.0  0.0  54056  4856 ?        Ss   10:30   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only
apache        23  0.0  0.0 260652  7884 ?        S    10:30   0:00 /usr/sbin/httpd -DFOREGROUND
apache        24  0.0  0.0 2760308 9512 ?       Sl   10:30   0:00 /usr/sbin/httpd -DFOREGROUND
apache        25  0.0  0.0 2563636 9748 ?       Sl   10:30   0:00 /usr/sbin/httpd -DFOREGROUND
apache        26  0.0  0.0 2563636 9516 ?       Sl   10:30   0:00 /usr/sbin/httpd -DFOREGROUND
root         238  0.0  0.0  19240  3564 pts/0    Ss   10:30   0:00 /bin/bash
root         247  0.0  0.0  51864  3728 pts/0    R+   10:30   0:00 ps aux
```

Ten en cuenta que el proceso `/sbin/init` se ejecuta con un PID de 1 y genera los procesos de httpd. El contenedor también ejecutó `dbus-daemon`, que es utilizado por systemd para exponer su API, junto con `systemd-journald` para manejar los registros.

Siguiendo este enfoque, podemos agregar múltiples servicios que se supone que funcionan juntos en el mismo contenedor y hacer que systemd los orqueste.

Hasta ahora, hemos visto las cuatro imágenes UBI disponibles actualmente y hemos demostrado cómo se pueden utilizar para crear aplicaciones personalizadas. Muchas imágenes públicas de Red Hat se basan en UBI. Echemos un vistazo.

#### Otras imágenes basadas en UBI

Red Hat utiliza imágenes UBI para producir muchas imágenes especializadas precompiladas, especialmente para tiempos de ejecución. Por lo general, se espera que no tengan limitaciones de redistribución.

Esto permite crear imágenes de tiempo de ejecución para lenguajes, entornos de ejecución y marcos como Python, Quarkus, Golang, Perl, PHP, .NET, Node.js, Ruby y OpenJDK.

UBI también se utiliza como imagen base para el marco Source-to-Image (S2I), que se utiliza para compilar aplicaciones de forma nativa en OpenShift sin el uso de Dockerfiles. Con S2I, es posible ensamblar imágenes a partir de scripts personalizados definidos por el usuario y, obviamente, el código fuente de la aplicación.

Por último, pero no menos importante, las versiones de contenedor compatibles de Red Hat de Buildah, Podman y Skopeo se empaquetan utilizando imágenes UBI 8.

Más allá de la oferta de Red Hat, otros proveedores también utilizan imágenes UBI para publicar sus imágenes: Intel, IBM, Isovalent, Cisco, Aqua Security y muchos otros adoptan UBI como base para sus imágenes oficiales en Red Hat Marketplace.

---

### Resumen

En este capítulo, aprendimos sobre la Especificación de Imagen OCI y el papel de los registros de contenedores. Después de eso, aprendimos cómo adoptar registros de imágenes seguros y cómo filtrar esos registros utilizando políticas personalizadas que nos permiten bloquear registros, espacios de nombres o imágenes específicos. Finalmente, presentamos UBI como una solución para crear imágenes ligeras, confiables y redistribuibles basadas en paquetes RHEL.

Con el conocimiento que has adquirido en este capítulo, deberías poder comprender la Especificación de Imagen OCI con más detalle y administrar los registros de imágenes de forma segura.

En el próximo capítulo, exploraremos la diferencia entre registros públicos y privados y cómo crear un registro privado localmente. Finalmente, aprenderemos cómo administrar imágenes de contenedores con la herramienta especializada Skopeo.

---

### Lecturas adicionales

Para obtener más información sobre los temas que se trataron en este capítulo, echa un vistazo a los siguientes recursos:

- Matriz MITRE ATT&CK®: [https://attack.mitre.org/matrices/enterprise/containers/](https://attack.mitre.org/matrices/enterprise/containers/)
- Cosas que debes saber sobre la matriz de amenazas de Kubernetes: [https://cloud.redhat.com/blog/2021-kubernetes-threat-matrix-updates-things-you-should-know](https://cloud.redhat.com/blog/2021-kubernetes-threat-matrix-updates-things-you-should-know)
- Cómo administrar registros de contenedores de Linux: [https://www.redhat.com/sysadmin/manage-container-registries](https://www.redhat.com/sysadmin/manage-container-registries)
- (Re)Presentación de la Red Hat Universal Base Image: [https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image)
- Introducción a UBI Micro de Red Hat: [https://www.redhat.com/en/blog/introduction-ubi-micro](https://www.redhat.com/en/blog/introduction-ubi-micro)
