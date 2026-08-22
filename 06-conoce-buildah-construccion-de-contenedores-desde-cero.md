# Parte 2: Construyendo tu Contenedor desde Cero con Buildah

## Capítulo 6: Conoce Buildah – Construcción de Contenedores desde Cero

El gran atractivo de los contenedores es que nos permiten empaquetar aplicaciones dentro de imágenes inmutables que se pueden desplegar en sistemas y ejecutar sin problemas. En este capítulo, aprenderemos cómo crear imágenes utilizando diferentes técnicas y herramientas. Esto incluye aprender cómo funciona la compilación de imágenes bajo el capó y cómo crear imágenes desde cero.

En este capítulo, vamos a cubrir los siguientes temas principales:

- Compilación básica de imágenes con Podman
- Conoce Buildah, el compañero de Podman
- Preparación de nuestro entorno
- Elección de nuestra estrategia de compilación
- Construcción de imágenes desde cero
- Construcción de imágenes a partir de un Dockerfile

---

### Requisitos técnicos

Antes de continuar con este capítulo, se requiere una máquina con una instalación de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Una buena comprensión de los temas tratados en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, es útil para asimilar fácilmente conceptos sobre imágenes de la Open Container Initiative (OCI).

---

### Compilación básica de imágenes con Podman

Una imagen OCI de un contenedor es un conjunto de capas inmutables apiladas juntas con una lógica de copia en escritura (*copy-on-write*). Cuando se compila una imagen, todas las capas se crean en un orden preciso y luego se envían al registro de contenedores, que almacena nuestras capas como archivos basados en tar junto con metadatos adicionales de la imagen.

Como aprendimos en la sección de imágenes OCI del [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/2), *Comparación entre Podman y Docker*, estos manifiestos son necesarios para reensamblar correctamente las capas de la imagen (el manifiesto de la imagen y el índice de la imagen) y pasar las configuraciones de tiempo de ejecución al motor de contenedores (la configuración de la imagen).

Antes de continuar con los ejemplos básicos de compilación de imágenes con Podman, debemos comprender cómo funcionan generalmente las compilaciones de imágenes para asimilar los conceptos clave simples pero muy inteligentes que subyacen.

#### Comprensión de las compilaciones bajo el capó

Las imágenes de contenedores se pueden compilar de diferentes maneras, pero el enfoque más común, probablemente una de las claves del gran éxito de los contenedores, se basa en Dockerfiles.

Un Dockerfile, como su nombre indica, es el archivo de configuración principal para las compilaciones de Docker y es una lista simple de acciones que se ejecutarán en el proceso de compilación.

Con el tiempo, los Dockerfiles se convirtieron en un estándar en las compilaciones de imágenes OCI y hoy en día se adoptan en muchos casos de uso.

> [!IMPORTANT]
> Para estandarizar y eliminar la asociación con la marca, también se creó la denominación Containerfiles; tienen exactamente la misma sintaxis que los Dockerfiles y son compatibles de forma nativa con Podman. En este libro, utilizaremos los dos términos Dockerfile y Containerfile indistintamente.

Aprenderemos en detalle sobre la sintaxis de Dockerfile en la siguiente subsección. Por ahora, centrémonos en un concepto: un Dockerfile es un conjunto de instrucciones de compilación que la herramienta de compilación ejecuta secuencialmente. Veamos este ejemplo:

```dockerfile
FROM docker.io/library/fedora 
RUN dnf install -y httpd && dnf clean all -y 
COPY index.html /var/www/html 
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Este ejemplo básico de un Dockerfile contiene solo cuatro instrucciones:

- La instrucción `FROM`, que define la imagen base inicial utilizada para compilar la imagen del contenedor.
- La instrucción `RUN`, que ejecuta algunas acciones durante la compilación (en este ejemplo, instalando paquetes con el administrador de paquetes `dnf`).
- La instrucción `COPY`, que copia archivos o directorios desde el directorio de trabajo de compilación a la imagen.
- La instrucción `CMD`, que define el comando que se ejecutará cuando se inicie el contenedor.

Cuando se ejecutan las acciones `RUN` y `COPY` del ejemplo, las nuevas capas que contienen los cambios se almacenan en caché en capas intermedias, representadas por imágenes temporales. Esta es una característica nativa de Docker que tiene la ventaja de reutilizar capas almacenadas en caché en compilaciones posteriores cuando no se solicitan cambios en una capa específica. Todos los contenedores intermedios producirán capas de solo lectura fusionadas por el controlador de gráficos overlay (*overlay graph driver*).

Los usuarios no necesitan administrar manualmente las capas en caché; el motor implementa automáticamente las acciones necesarias creando los contenedores temporales, ejecutando las acciones definidas por las instrucciones de Dockerfile y luego confirmando (*committing*). Al repetir la misma lógica para todas las instrucciones necesarias, Podman crea una nueva imagen con capas adicionales encima de las de la imagen base.

Es posible aplastar (*squash*) las capas de la imagen en una sola para evitar un impacto negativo en el rendimiento de overlay. Podman ofrece las mismas funciones y te permite elegir entre almacenar en caché capas intermedias o no.

No todas las instrucciones de Dockerfile cambian el sistema de archivos, y solo aquellas que lo hacen crearán una nueva capa de imagen; todas las demás instrucciones, como la instrucción `CMD` en el ejemplo anterior, producen una capa vacía con metadatos únicamente y sin cambios en el sistema de archivos overlay.

En general, las únicas instrucciones que crean nuevas capas al cambiar efectivamente el sistema de archivos son las instrucciones `RUN`, `COPY` y `ADD`. Todas las demás instrucciones en un Dockerfile o Containerfile simplemente crean imágenes intermedias temporales y no afectan el sistema de archivos de la imagen final.

Esta es también una buena razón para mantener limitado el número de instrucciones `RUN`, `COPY` y `ADD` en un Dockerfile, ya que tener imágenes sobrecargadas con demasiadas capas no es un buen patrón e impacta el rendimiento del controlador de gráficos.

Podemos inspeccionar el historial de una imagen y las acciones que se han aplicado a cada capa. El siguiente ejemplo muestra un extracto de la salida del comando `podman inspect`, siendo la imagen de destino una potencial creada a partir del Dockerfile de ejemplo anterior:

```bash
$ podman inspect myhttpd
[...omitted output]
    "History": [
      {
        "created": "2024-10-31T11:44:19Z",
        "created_by": "LABEL maintainer=Clement Verna <cverna@fedoraproject.org>",
        "comment": "buildkit.dockerfile.v0",
        "empty_layer": true
      },
      {
        "created": "2024-10-31T11:44:19Z",
        "created_by": "ENV DISTTAG=f41container FGC=f41 FBR=f41",
        "comment": "buildkit.dockerfile.v0",
        "empty_layer": true
      },
      {
        "created": "2024-10-31T11:44:19Z",
        "created_by": "ADD fedora-20241031.tar / # buildkit",
        "comment": "buildkit.dockerfile.v0"
      },
      {
        "created": "2024-10-31T11:44:19Z",
        "created_by": "CMD [\"/bin/bash\"]",
        "comment": "buildkit.dockerfile.v0",
        "empty_layer": true
      },
      {
        "created": "2024-11-01T10:26:32.41866158Z",
        "created_by": "/bin/sh -c dnf install -y httpd && dnf clean all -y ",
        "comment": "FROM docker.io/library/fedora:latest"
      },
      {
        "created": "2024-11-01T10:26:33.243300733Z",
        "created_by": "/bin/sh -c #(nop) COPY file:846c7ee0fa97be3c73c8d2fb0dd482f31c2ec12d7e74ad2323e6c5bde9ef68ad in /var/www/html ",
        "comment": "FROM 97ad0fd6a1a3"
      },
      {
        "created": "2024-11-01T10:26:33.317359389Z",
        "created_by": "/bin/sh -c #(nop) CMD [\"/usr/sbin/httpd\", \"-DFOREGROUND\"]",
        "comment": "FROM f737afe7b188",
        "empty_layer": true
      }
    ]
[...omitted output]
```

Al observar los últimos tres elementos del historial de la imagen, podemos notar las instrucciones exactas definidas en el Dockerfile, incluida la última instrucción `CMD`, que no crea ninguna capa nueva sino metadatos que persistirán en la configuración de la imagen.

Con esta comprensión más profunda de la lógica de compilación de imágenes en mente, exploremos ahora las instrucciones de Dockerfile más comunes antes de continuar con los ejemplos de compilación de Podman.

#### Exploración de las instrucciones de Dockerfile y Containerfile

Como se indicó anteriormente, los Dockerfiles y Containerfiles comparten la misma sintaxis. Las instrucciones en esos archivos deben verse como (y verdaderamente son) comandos pasados al motor de contenedores o herramienta de compilación. Esta subsección proporciona una descripción general de las instrucciones utilizadas con mayor frecuencia.

Todas las instrucciones de Dockerfile/Containerfile siguen el mismo patrón:

```dockerfile
# Comment
INSTRUCTION arguments
```

La siguiente lista proporciona una descripción no exhaustiva de las instrucciones más comunes:

- **FROM**: Esta es la primera instrucción de una etapa de compilación y define la imagen base utilizada como punto de partida de la compilación. Sigue la sintaxis `FROM <image>[:<tag>]` para identificar la imagen correcta a utilizar.
- **RUN**: Esta instrucción le indica al motor que ejecute los comandos pasados como argumentos dentro de un contenedor temporal. Este contenedor temporal utiliza el sistema de archivos de la imagen que estás compilando. Sigue la sintaxis `RUN <command>`. El binario o script invocado debe existir en la imagen base o en una capa anterior. Como se indicó anteriormente, la instrucción `RUN` crea una nueva capa de imagen; por lo tanto, es una práctica frecuente concatenar comandos en la misma instrucción `RUN` para evitar sobrecargar la imagen con demasiadas capas.
  
  Este ejemplo compacta tres comandos dentro de la misma instrucción `RUN`:

  ```dockerfile
  RUN dnf upgrade -y && \
      dnf install httpd -y && \
      dnf clean all -y
  ```

- **COPY**: Esta instrucción copia archivos y carpetas desde el directorio de trabajo de compilación al entorno aislado (*sandbox*) de compilación. Los recursos copiados se conservan en la imagen final. Sigue la sintaxis `COPY <src>… <dest>`, y tiene una opción muy útil que nos permite definir el usuario y grupo de destino en lugar de cambiar la propiedad manualmente más adelante: `--chown=<user>:<group>`.
- **ADD**: Esta instrucción copia archivos, carpetas y URLs remotas al destino de compilación. Sigue la sintaxis `ADD <src>… <dest>`. Esta instrucción también admite la extracción automática de archivos tar desde un origen directamente a la ruta de destino.
- **ENTRYPOINT**: El comando ejecutado en el contenedor. Recibe argumentos desde la línea de comandos (en la forma de `podman run <image> <arguments>`) o desde la instrucción `CMD`. Un `ENTRYPOINT` de imagen no se puede anular mediante argumentos de línea de comandos. Las formas admitidas son las siguientes:
  - `ENTRYPOINT ["command", "param1", "paramN"]` (también conocida como la forma *exec*)
  - `ENTRYPOINT command param1 paramN` (la forma *shell*)
  
  Si no se establece, el valor predeterminado para `ENTRYPOINT` es `bash -c`. Cuando se establece en el valor predeterminado, los comandos se pasan como argumento al proceso bash. Por ejemplo, si se pasa un comando `ps aux` como argumento en tiempo de ejecución o en una instrucción `CMD`, el contenedor ejecutará `bash -c "ps aux"`.
  Una práctica frecuente es reemplazar el comando `ENTRYPOINT` predeterminado con un script personalizado que se comporta de la misma manera y ofrece un control más granular de la ejecución en tiempo de ejecución.
- **CMD**: El argumento (o argumentos) predeterminado pasado a la instrucción `ENTRYPOINT`. Puede ser un comando completo o un conjunto de argumentos simples que se pasarán a un script o binario personalizado establecido como `ENTRYPOINT`. Las formas admitidas son las siguientes:
  - `CMD ["command", "param1", "paramN"]` (la forma *exec*)
  - `CMD ["param1", "paramN"]` (la forma de parámetros, utilizada para pasar argumentos a un `ENTRYPOINT` personalizado)
  - `CMD command param1 paramN` (la forma *shell*)
- **LABEL**: Esta instrucción se utiliza para aplicar etiquetas personalizadas a la imagen. Las etiquetas se utilizan como metadatos en el momento de la compilación o en tiempo de ejecución. Sigue la sintaxis `LABEL <key1>=<value1> … <keyN>=<valueN>`.
- **EXPOSE**: Esto establece metadatos sobre los puertos de escucha expuestos por los procesos que se ejecutan en el contenedor. Admite el formato `EXPOSE <port>/<protocol>`. Estos puertos no se reenvían de forma predeterminada cuando se ejecuta un contenedor basado en la imagen y requieren una acción explícita del usuario.
- **ENV**: Esto configura variables de entorno que estarán disponibles para los siguientes comandos de compilación y en tiempo de ejecución cuando se ejecute el contenedor. Esta instrucción admite el formato `ENV <key1>=<value1>… <keyN>=<valueN>`. Las variables de entorno también se pueden establecer dentro de una instrucción `RUN` con un alcance limitado a la instrucción misma.
- **VOLUME**: Esto establece un volumen que se creará en tiempo de ejecución durante la ejecución del contenedor. El volumen será mapeado automáticamente por Podman dentro del directorio de almacenamiento de volúmenes predeterminado. Los formatos admitidos son los siguientes:
  - `VOLUME ["/path/to/dir"]`
  - `VOLUME /path/to/dir`
  
  Consulta también la sección *Adjuntar almacenamiento del host a un contenedor* en el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*, para obtener más detalles sobre los volúmenes.
- **USER**: Esta instrucción define el nombre de usuario y el grupo de usuarios para las siguientes instrucciones `RUN`, `CMD` y `ENTRYPOINT` cuando ejecutas la imagen. El valor de GID no es obligatorio. Los formatos admitidos son los siguientes:
  - `USER <username>:[<groupname>]`
  - `USER <UID>:[<GID>]`
- **WORKDIR**: Esto establece el directorio de trabajo durante el proceso de compilación. Este valor se conserva durante la ejecución del contenedor. Admite el formato `WORKDIR /path/to/workdir`.
- **ONBUILD**: Esta instrucción define un comando de activación (*trigger*) que se ejecutará una vez que se haya completado la compilación de una imagen. De esta manera, la imagen se puede utilizar como base para una nueva compilación llamándola con la instrucción `FROM`. Su propósito es permitir la ejecución de algún comando final en una imagen de contenedor secundaria. Los formatos admitidos son los siguientes:
  - `ONBUILD ADD . /opt/app`
  - `ONBUILD RUN /opt/bin/custom-build /opt/app/src`

Ahora que hemos aprendido las instrucciones más comunes, profundicemos en nuestros primeros ejemplos de compilación con Podman.

#### Ejecución de compilaciones con Podman

Buenas noticias: Podman proporciona los mismos comandos y sintaxis de compilación que Docker. Si estás migrando desde Docker, no habrá curva de aprendizaje para comenzar a compilar tus imágenes con él. Bajo el capó, existe una ventaja notable al elegir Podman como herramienta de compilación: Podman puede compilar contenedores en modo rootless, utilizando un modelo fork/exec.

Este es un paso adelante en comparación con las compilaciones de Docker, donde la comunicación con el demonio que escucha en el socket Unix es necesaria para ejecutar la compilación.

Comencemos ejecutando una compilación simple basada en el Dockerfile de httpd ilustrado en la primera subsección, *Comprensión de las compilaciones bajo el capó*. Usaremos el siguiente comando `podman build`:

```bash
$ podman build -t myhttpd .
STEP 1/4: FROM docker.io/library/fedora
STEP 2/4: RUN dnf install -y httpd && dnf clean all -y
[...omitted output]
--> 50a981094eb
STEP 3/4: COPY index.html /var/www/html
--> 73f8702c5e0
STEP 4/4: CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
COMMIT myhttpd
--> e773bfee6f2
Successfully tagged localhost/myhttpd:latest
e773bfee6f289012b37285a9e559bc44962de3aeed001455231b5a8f2721b8f9
```

En el ejemplo anterior, la salida del comando `dnf install` se omitió por razones de claridad y espacio.

El comando ejecuta las instrucciones secuencialmente y conserva las capas intermedias hasta que la imagen final se confirma y se etiqueta. Estas capas permanecen disponibles como caché hasta que se limpian manualmente con `podman image prune`; de lo contrario, no las tendríamos disponibles si se reconstruyera la imagen.

Los pasos de compilación están numerados (1/4 a 4/4), y algunos de ellos (`RUN` y `COPY` aquí) producen capas no vacías, formando parte de la imagen `lowerDirs`.

La primera instrucción `FROM` define la imagen base, que se descarga automáticamente si no está presente en el host.

La segunda instrucción es `RUN`, que ejecuta el comando `dnf` para instalar el paquete httpd y limpiar el sistema al finalizar. Bajo el capó, esta línea se ejecuta como `"bash -c 'dnf install -y httpd && dnf clean all -y'"`.

La tercera instrucción `COPY` simplemente copia el archivo `index.html` en la raíz de documentos predeterminada de httpd.

Finalmente, el cuarto paso define la instrucción `CMD` predeterminada del contenedor. Dado que no se establecieron instrucciones `ENTRYPOINT`, esto se traducirá en el siguiente comando:

```bash
"bash -c '/usr/sbin/httpd -DFOREGROUND'"
```

El siguiente ejemplo es un Dockerfile/Containerfile personalizado donde se compila un servidor web a medida:

```dockerfile
FROM docker.io/library/fedora
# Install required packages
RUN set -euo pipefail; \
    dnf upgrade -y; \
    dnf install httpd -y; \
    dnf clean all -y; \
    rm -rf /var/cache/dnf/*
# Custom webserver configs for rootless execution
RUN set -euo pipefail; \
    sed -i 's|Listen 80|Listen 8080|' \
    /etc/httpd/conf/httpd.conf; \
    sed -i 's|ErrorLog "logs/error_log"|ErrorLog /dev/stderr|' \
    /etc/httpd/conf/httpd.conf; \
    sed -i 's|CustomLog "logs/access_log" combined|CustomLog /dev/stdout combined|' \
    /etc/httpd/conf/httpd.conf; \
    chown 1001 /var/run/httpd
# Copy web content
COPY index.html /var/www/html
# Define content volume
VOLUME /var/www/html
# Copy container entrypoint.sh script
COPY entrypoint.sh /entrypoint.sh
# Declare exposed ports
EXPOSE 8080
# Declare default user
USER 1001
ENTRYPOINT ["/entrypoint.sh"]
CMD ["httpd"]
```

Este ejemplo fue diseñado para el propósito de este libro para ilustrar algunos elementos peculiares:

- Los paquetes instalados con un administrador de paquetes deben mantenerse al mínimo. Después de instalar el paquete httpd, necesario para ejecutar el servidor web, se limpia la caché para reducir el tamaño de la imagen.
- Se pueden agrupar múltiples comandos en una sola instrucción `RUN`. Sin embargo, no queremos continuar con la compilación si un solo comando falla. Para proporcionar una ejecución de shell a prueba de fallos, se antepuso el comando `set -euo pipefail`. Además, para mejorar la legibilidad, los comandos individuales se dividieron en más líneas utilizando el carácter `\`, que puede funcionar como un salto de línea o carácter de escape.
- Para evitar ejecutar los procesos aislados como usuario root, se implementaron una serie de soluciones alternativas para tener el proceso httpd ejecutándose como el usuario genérico `1001`. Esas soluciones incluyeron la actualización de los permisos de archivos y la propiedad de grupo en directorios específicos a los que se espera que accedan usuarios no root. Esta es una mejor práctica de seguridad que reduce la superficie de ataque del contenedor. Por supuesto, esto no es necesario con contenedores rootless.
- Un patrón común en los contenedores es la redirección de los registros de la aplicación a `stdout` y `stderr` del contenedor. Los flujos de registro comunes de httpd se han modificado para este propósito utilizando expresiones regulares contra el archivo `/etc/httpd/conf/httpd.conf`.
- Los puertos del servidor web se declaran como expuestos con la instrucción `EXPOSE`.
- La instrucción `CMD` es un comando httpd simple sin ningún otro argumento. Esto se hizo para ilustrar cómo `ENTRYPOINT` puede interactuar con los argumentos de `CMD`.
- La instrucción `ENTRYPOINT` del contenedor se modifica con un script personalizado que aporta más flexibilidad a la forma en que se gestiona la instrucción `CMD`. El archivo `entrypoint.sh` comprueba si el contenedor se ejecuta como root y verifica el primer argumento de `CMD`: si el argumento es `httpd`, ejecuta el comando `httpd -DFOREGROUND`; de lo contrario, te permite ejecutar cualquier otro comando (una shell, por ejemplo). El siguiente código es el contenido del script `entrypoint.sh`:

```bash
#!/bin/sh
set -euo pipefail
if [ $UID != 0 ]; then
    echo "Running as user $UID"
fi
if [ "$1" == "httpd" ]; then
    echo "Starting custom httpd server"
    exec $1 -DFOREGROUND
else
    echo "Starting container with custom arguments"
    exec "$@"
fi
```

Ahora compilemos la imagen con el comando `podman build`:

```bash
$ podman build -t myhttpd .
```

La imagen recién compilada estará disponible en la caché local del host:

```bash
$ podman images | grep myhttpd
localhost/myhttpd   latest   6dc90348520c   2 minutes ago   248 MB
```

Después de compilar, podemos etiquetar la imagen con el nombre del registro de destino. El siguiente ejemplo etiqueta la imagen aplicando la etiqueta `v1.0` y la etiqueta `latest`:

```bash
$ podman tag localhost/myhttpd quay.io/<username>/myhttpd:v1.0
```

Después de etiquetarla, la imagen estará lista para enviarse al registro remoto. Cubriremos la interacción con los registros con mayor detalle en el [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*.

La imagen de ejemplo estará compuesta por cinco capas, incluida la capa de la imagen base de Fedora. Podemos verificar el número de capas ejecutando el comando `podman inspect` contra la nueva imagen:

```bash
$ podman inspect myhttpd --format '{{ .RootFS.Layers }}'
[sha256:b6d0e02fe431db7d64d996f3dbf903153152a8f8b857cb4829ab3c4a3e484a72 sha256:f41274a78d9917b0412d99c8b698b0094aa0de74ec8995c88e5dbf1131494912 sha256:e57dde895085c50ea57db021bffce776ee33253b4b8cb0fe909bbbac45af0e8c sha256:9989ee85603f534e7648c74c75aaca5981186b787d26e0cae0bc7ee9eb54d40d sha256:ca402716d23bd39f52d040a39d3aee242bf235f626258958b889b40cdec88b43]
```

También puedes echar un vistazo a las capas de la imagen con el subcomando dedicado `podman image tree`:

```bash
$ podman image tree myhttpd
  main Image ID: 4067fa24786a
  Tags: [localhost/myhttpd:latest]
  Size: 317.6MB
  Image Layers
  ├── ID: 38f7d1e80c77 Size: 186.2MB Top Layer of: [docker.io/library/fedora:latest]
  ├── ID: 3d6a999a163f Size: 131.3MB
  ├── ID: 6fc772abb506 Size: 15.87kB
  ├── ID: 778ad0afa90a Size: 3.584kB
  └── ID: da726f9c0cf9 Size: 2.048kB Top Layer of: [localhost/myhttpd:latest]
```

Es posible aplastar (*squash*) las capas de compilación actuales en una sola capa utilizando la opción `--layers=false`. La imagen resultante tendrá solo dos capas: la capa base de Fedora y la capa aplastada. El siguiente ejemplo reconstruye la imagen sin almacenar en caché las capas intermedias:

```bash
$ podman build -t myhttpd --layers=false .
```

Inspeccionemos la imagen de salida nuevamente:

```bash
$ podman inspect myhttpd --format '{{ .RootFS.Layers }}'
[sha256:b6d0e02fe431db7d64d996f3dbf903153152a8f8b857cb4829ab3c4a3e484a72 sha256:6c279ab14837b30af9360bf337c7f9b967676a61831eee91012fa67083f5dcf1]
```

Esta vez, la imagen final tiene solo las dos capas esperadas.

Reducir la cantidad de capas puede ser útil para mantener la imagen mínima en términos de superposiciones (*overlays*). La desventaja de este enfoque es que tendremos que reconstruir toda la imagen para cada cambio de configuración sin aprovechar las capas en caché.

En términos de aislamiento, Podman puede compilar imágenes de forma segura en modo rootless. De hecho, esto se considera un valor ya que no debería haber necesidad de ejecutar compilaciones con un usuario privilegiado como root. Si las compilaciones rootful son necesarias, son totalmente funcionales y compatibles. El siguiente ejemplo ejecuta una compilación como usuario root (observa el símbolo de almohadilla típico de la sesión de terminal de root en Linux):

```bash
# podman build -t myhttpd .
```

La imagen resultante estará disponible solo en la caché de imágenes del sistema y sus capas se almacenarán bajo `/var/lib/containers/storage/`.

La naturaleza flexible de las compilaciones de Podman está fuertemente relacionada con su herramienta complementaria, Buildah, una herramienta especializada para compilar imágenes OCI que proporciona una mayor flexibilidad en las compilaciones. En la siguiente sección, describiremos las características de Buildah y cómo gestiona las compilaciones de imágenes.

---

### Presentación de Buildah, el compañero de Podman

Podman hace un excelente trabajo en compilaciones simples con Dockerfiles/Containerfiles y ayuda a los equipos a preservar sus canalizaciones de compilación implementadas previamente sin necesidad de nuevas inversiones.

Sin embargo, cuando se trata de tareas de compilación más especializadas o cuando los usuarios necesitan más control sobre el flujo de trabajo de compilación, con la opción de incluir lógica de scripting, el enfoque de Dockerfile/Containerfile muestra sus limitaciones. Las comunidades lucharon por encontrar enfoques de compilación alternativos que pudieran superar la lógica rígida basada en flujos de trabajo de los Dockerfiles/Containerfiles.

La misma comunidad que desarrolla Podman dio vida al proyecto Buildah (pronunciado *build-ah*), una herramienta para administrar compilaciones OCI con soporte para múltiples estrategias de compilación. Las imágenes creadas con Buildah son totalmente portátiles y compatibles con Docker, y todos los motores cumplen con las especificaciones de imagen y tiempo de ejecución de OCI.

Buildah es un proyecto de código abierto publicado bajo la licencia Apache 2.0. Las fuentes están disponibles en GitHub en la siguiente URL: [https://github.com/containers/buildah](https://github.com/containers/buildah).

Buildah es complementario a Podman, que toma prestada su lógica de compilación mediante la inclusión de sus bibliotecas para implementar funcionalidades básicas de compilación contra Dockerfiles y Containerfiles. El binario final de Podman, que se compila en Go como un único archivo vinculado estáticamente, incorpora paquetes de Buildah para administrar los pasos de compilación.

Buildah utiliza el proyecto `containers/image` ([https://github.com/containers/image](https://github.com/containers/image)) para administrar el ciclo de vida de una imagen y su interacción con los registros, y el proyecto `containers/storage` ([https://github.com/containers/storage](https://github.com/containers/storage)) para administrar imágenes y capas del sistema de archivos de contenedores. Estas bibliotecas son las mismas que usa Podman, y ambas se han trasladado ahora al proyecto `containers/containers-libs` ([https://github.com/containers/container-libs](https://github.com/containers/container-libs)).

La estrategia de compilación avanzada de Buildah se basa en el soporte paralelo para compilaciones tradicionales basadas en Dockerfile/Containerfile y para compilaciones impulsadas por comandos nativos de Buildah que replican las instrucciones de Dockerfile.

Al replicar las instrucciones de Dockerfile en comandos estándar, Buildah se convierte en una herramienta integrable en scripts que se puede interpolar con lógica personalizada y construcciones nativas de shell como condicionales, bucles o variables de entorno. Por ejemplo, la instrucción `RUN` en un Dockerfile se puede reemplazar con un comando `buildah run`.

Si los equipos necesitan preservar la lógica de compilación implementada en Dockerfiles anteriores, Buildah ofrece el comando `buildah build` (o su alias, `buildah bud`), que compila la imagen leyendo desde el Dockerfile/Containerfile proporcionado, como en un comando `podman build` típico.

Buildah puede ejecutarse sin problemas en modo rootless para compilar imágenes; esta es una característica valiosa y muy demandada desde el punto de vista de la seguridad. No se necesitan sockets Unix para ejecutar una compilación. Al principio de este capítulo, explicamos cómo las compilaciones siempre se basan en contenedores; Buildah no está exento de este comportamiento y todas sus compilaciones se ejecutan dentro de contenedores de trabajo (*working containers*), comenzando sobre la imagen base.

La siguiente lista proporciona una descripción no exhaustiva de los comandos utilizados con mayor frecuencia en Buildah:

- **buildah from**: Inicializa un nuevo contenedor de trabajo sobre una imagen base. Acepta la sintaxis `buildah from [options] <image>`. Un ejemplo de este comando es `$ buildah from fedora`.
- **buildah run**: Esto es equivalente a la instrucción `RUN` de un Dockerfile; ejecuta un comando dentro de un contenedor de trabajo. Este comando acepta la sintaxis `buildah run [options] [--] <container> <command>`. La opción `--` (doble guion) es necesaria para separar las opciones potenciales del comando de contenedor efectivo. Un ejemplo de este comando es `buildah run <containerID> -- dnf install -y nginx`.
- **buildah config**: Este comando configura metadatos de la imagen. Acepta el formato `buildah config [options] <container>`. Las opciones disponibles para este comando están asociadas con las diversas instrucciones de Dockerfile que no modifican las capas del sistema de archivos sino que establecen algunos metadatos del contenedor; por ejemplo, la configuración del entrypoint del contenedor. Un ejemplo de este comando es `buildah config --entrypoint /entrypoint.sh <containerID>`.
- **buildah add**: Esto es equivalente a la instrucción `ADD` del Dockerfile; agrega archivos, directorios e incluso URLs al contenedor. Admite la sintaxis `buildah add [options] <container> <src> [[src …] <dst>` y te permite copiar varios archivos en un solo comando. Un ejemplo de este comando es `buildah add <containerID> index.php /var/www/html`.
- **buildah copy**: Esto es lo mismo que la instrucción `COPY` de Dockerfile; agrega archivos, URLs y directorios al contenedor. Admite la sintaxis `buildah copy [options] <container> <src> [[src …] <dst>`. Un ejemplo de este comando es `buildah copy <containerID> entrypoint.sh /`.
- **buildah commit**: Esto confirma (*commits*) una imagen final a partir de un contenedor de trabajo. Este comando suele ser el último en ejecutarse. Admite la sintaxis `buildah commit [options] <container> <image_name>`. La imagen de contenedor creada a partir de este comando se puede etiquetar y enviar a un registro más adelante. Un ejemplo de este comando es `buildah commit <containerID> <myhttpd>`.
- **buildah build**: El comando equivalente del clásico comando `podman build`. Este comando toma Dockerfiles o Containerfiles como argumentos, junto con la ruta del directorio de compilación. Acepta la sintaxis `buildah build [options] [context]` y el alias de comando `buildah bud`. Un ejemplo de este comando es `buildah build -t <imageName> .`.
- **buildah containers**: Esto enumera los contenedores de trabajo activos involucrados en compilaciones de Buildah, junto con la imagen base utilizada como punto de partida. Los comandos equivalentes son `buildah ls` y `buildah ps`. La sintaxis admitida es `buildah containers [options]`. Un ejemplo de este comando es `buildah containers`.
- **buildah rm**: Se utiliza para eliminar contenedores de trabajo. El comando `buildah delete` es equivalente. La sintaxis admitida es `buildah rm <container>`. Este comando tiene solo una opción, la opción `-all`, `-a`, para eliminar todos los contenedores de trabajo. Un ejemplo de este comando es `buildah rm <containerID>`.
- **buildah mount**: Este comando se puede utilizar para montar el sistema de archivos raíz de un contenedor de trabajo. La sintaxis aceptada es `buildah mount [containerID … ]`. Cuando no se pasa ningún argumento, el comando solo muestra los contenedores montados actualmente. Un ejemplo práctico de este comando utilizado para montar el sistema de archivos de un contenedor es `buildah mount <containerID>`. Esto se puede utilizar para evitar `buildah add` y `buildah copy` y, en su lugar, cambiar directamente el sistema de archivos del contenedor tú mismo.
- **buildah images**: Esto enumera todas las imágenes disponibles en la caché local del host. La sintaxis aceptada es `buildah images [options] [image]`. Hay disponibles formatos de salida personalizados como JSON. Un ejemplo de este comando es `buildah images --json`.
- **buildah tag**: Esto aplica un nombre personalizado y etiquetas a una imagen en el almacén local. La sintaxis sigue el formato `buildah tag <name> <new-name>`. Un ejemplo de este comando es `buildah tag myapp quay.io/packt/myapp:latest`.
- **buildah push**: Esto envía una imagen local a un registro remoto privado o público, o directorios locales en formato Docker u OCI. La sintaxis del comando es `buildah push [options] <image> [destination]`. Los ejemplos de este comando incluyen `buildah push quay.io/packt/myapp:latest`, `buildah push <imageID> docker://<URL>/repository:tag` y `buildah push <imageID> oci:</path/to/dir>:image:tag`.
- **buildah pull**: Esto descarga una imagen desde un registro, un archivo comprimido OCI o un directorio. La sintaxis incluye `buildah pull [options] <image>`. Los ejemplos de este comando incluyen `buildah pull <imageName>`, `buildah pull docker://<URL>/repository:tag` y `buildah pull dir:</path/to/dir>`.

Todos los comandos descritos anteriormente tienen su página de manual correspondiente, con el patrón `man buildah-<command>`. Por ejemplo, para leer los detalles de la documentación sobre el comando `buildah run`, simplemente escribe `man buildah-run` en el terminal.

El siguiente ejemplo muestra las capacidades básicas de Buildah. Una imagen base de Fedora se personaliza para ejecutar un proceso httpd:

```bash
$ container=$(buildah from fedora)
$ buildah run $container -- dnf install -y httpd; dnf clean all
$ buildah config --cmd "httpd -DFOREGROUND" $container
$ buildah config --port 80 $container
$ buildah commit $container myhttpd
$ buildah tag myhttpd registry.example.com/myhttpd:v0.0.1
```

Los comandos anteriores producirán una imagen portátil que cumple con OCI con las mismas características que una imagen compilada a partir de un Dockerfile, todo en unas pocas líneas que se pueden incluir en un script simple.

> [!TIP]
> Si bien el ejemplo anterior utiliza `buildah run` para ejecutar comandos dentro del contenedor, la verdadera arma secreta de Buildah es el comando `buildah mount`.
> Una de las razones principales por las que los desarrolladores eligen Buildah en lugar de Dockerfiles estándar es la capacidad de interactuar con el sistema de archivos del contenedor directamente desde el host. Cuando ejecutas `buildah mount $container`, el sistema de archivos raíz del contenedor se asigna a una ruta en tu máquina host. Esto desbloquea varios flujos de trabajo potentes.

Nos centraremos ahora en el primer comando:

```bash
$ container=$(buildah from fedora)
```

El comando `buildah from` descarga una imagen de Fedora de uno de los registros permitidos e inicia un contenedor de trabajo a partir de ella, devolviendo el nombre del contenedor. En lugar de simplemente imprimirlo en la salida estándar, capturaremos el nombre con la sintaxis de expansión de shell. A partir de ahora, podemos pasar la variable `$container`, que contiene el nombre del contenedor generado, a los comandos posteriores. Por lo tanto, los comandos de compilación se ejecutarán dentro de este contenedor de trabajo. Este es un patrón bastante común y es especialmente útil para automatizar comandos de Buildah en scripts.

> [!IMPORTANT]
> Hay una sutil diferencia entre el concepto de contenedores en Buildah y Podman. Ambos adoptan la misma tecnología para crear contenedores, pero los contenedores de Buildah son entidades de corta duración que se crean para ser modificadas y confirmadas (*committed*), mientras que los contenedores de Podman están destinados a ejecutar cargas de trabajo de larga duración.
> Además, ten en cuenta que comandos como `podman ps` no muestran los contenedores de Buildah de forma predeterminada y requieren la opción `--external`.

La naturaleza flexible e integrable de este enfoque es notable: los comandos de Buildah se pueden incluir en cualquier lugar y los usuarios pueden elegir entre un proceso de compilación totalmente automatizado y uno más interactivo.

Por ejemplo, Buildah se puede integrar fácilmente con Ansible, el motor de automatización de código abierto, para proporcionar compilaciones automatizadas mediante complementos de conexión nativos que permiten la comunicación con contenedores de trabajo.

Puedes optar por incluir Buildah dentro de una canalización de CI (como Jenkins, Tekton o GitLab CI/CD) para obtener un control total de las tareas de compilación e integración. Tekton, la herramienta de CI/CD nativa de la nube ([https://tekton.dev/](https://tekton.dev/)), ofrece una colección de tareas de plantilla, el bloque de construcción de las canalizaciones de Tekton, en el repositorio de Tekton Hub ([https://hub.tekton.dev/](https://hub.tekton.dev/)); las tareas de Buildah también están disponibles en Tekton Hub ([https://hub.tekton.dev/tekton/task/buildah](https://hub.tekton.dev/tekton/task/buildah)). Además, Buildah se puede ejecutar dentro de un contenedor sin problemas, lo que le permite ejecutarse en muchas otras canalizaciones de CI, incluso si normalmente no admiten compilaciones de contenedores.

Buildah también se incluye en proyectos más grandes de la comunidad nativa de la nube, como el proyecto Shipwright ([https://github.com/shipwright-io/build](https://github.com/shipwright-io/build)).

Shipwright es un marco de compilación extensible para Kubernetes que brinda la flexibilidad de personalizar compilaciones de imágenes mediante definiciones de recursos personalizados y diferentes herramientas de compilación. Buildah es una de las soluciones disponibles que puedes elegir al diseñar tus procesos de compilación.

Veremos ejemplos más detallados y enriquecidos en las siguientes subsecciones. Ahora que hemos visto una descripción general de las capacidades y casos de uso de Buildah, profundicemos en los pasos de instalación y preparación del entorno.

---

### Preparación de nuestro entorno

Buildah está disponible en diferentes distribuciones y se puede instalar utilizando los administradores de paquetes respectivos. Esta sección proporciona una lista no exhaustiva de ejemplos de instalación en las principales distribuciones. En aras de la claridad, es importante reiterar que los entornos de laboratorio del libro se basaron todos en Fedora 40:

- **Fedora**: Para instalar Buildah en Fedora y en cualquier otra distribución de Linux que utilice el administrador de paquetes DNF, ejecuta el siguiente comando `dnf`:

  ```bash
  $ sudo dnf -y install buildah
  ```

- **Debian**: Para instalar Buildah en Debian Bullseye o posterior, ejecuta los siguientes comandos `apt-get`:

  ```bash
  $ sudo apt-get update 
  $ sudo apt-get -y install buildah
  ```

- **CentOS Stream 9**: Para instalar Buildah en CentOS Stream 9, ejecuta el siguiente comando `yum`:

  ```bash
  $ sudo dnf install -y buildah
  ```

- **RHEL8**: Para instalar Buildah en RHEL8, ejecuta los siguientes comandos de módulo `yum`:

  ```bash
  $ sudo yum module enable -y container-tools:1.0 
  $ sudo yum module install -y buildah
  ```

- **RHEL9**: Para instalar Buildah en RHEL9, instala el metapaquete `container-tools`:

  ```bash
  $ sudo dnf -y install container-tools
  ```

- **Arch Linux**: Para instalar Buildah en Arch Linux, ejecuta el siguiente comando `pacman`:

  ```bash
  $ sudo pacman -S buildah
  ```

- **Ubuntu**: Para instalar Buildah en Ubuntu 20.10 o posterior, ejecuta los siguientes comandos `apt-get`:

  ```bash
  $ sudo apt-get -y update 
  $ sudo apt-get -y install buildah
  ```

- **Gentoo**: Para instalar Buildah en Gentoo, ejecuta el siguiente comando `emerge`:

  ```bash
  $ sudo emerge app-emulation/libpod
  ```

- **Compilación desde el código fuente**: Buildah también se puede compilar desde el código fuente. Para el propósito de este libro, mantendremos el enfoque en métodos de implementación simples, pero si tienes curiosidad, encontrarás útil la siguiente guía para probar tus propias compilaciones: [https://github.com/containers/buildah/blob/main/install.md#building-from-scratch](https://github.com/containers/buildah/blob/main/install.md#building-from-scratch).

Finalmente, Buildah se puede implementar como un contenedor y las compilaciones se pueden ejecutar dentro de él con un enfoque anidado. Este proceso se cubrirá con mayor detalle en el [Capítulo 7](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/7), *Integración con Procesos de Compilación de Aplicaciones Existentes*.

Después de instalar Buildah en nuestro host, podemos pasar a verificar nuestra instalación.

#### Verificación de la instalación

Después de instalar Buildah, ahora podemos ejecutar algunos comandos de prueba básicos para verificar la instalación.

Para ver todas las imágenes disponibles en el almacén local del host, utiliza los siguientes comandos:

```bash
$ buildah images 
# buildah images
```

La lista de imágenes será la misma que la impresa por el comando `podman images`, ya que comparten el mismo almacén local.

Ten en cuenta también que los dos comandos se ejecutan como un usuario no privilegiado y como root, apuntando respectivamente al almacén local rootless del usuario y al almacén local de todo el sistema.

Podemos ejecutar una compilación de prueba simple para verificar la instalación. Esta es una buena oportunidad para probar un script de compilación básico cuyo único propósito es verificar si Buildah puede ejecutar completamente una compilación completa.

Para el propósito de este libro (y por diversión), hemos creado el siguiente script de prueba simple que crea una imagen mínima de Python 3:

```bash
#!/bin/bash 
BASE_IMAGE=alpine 
TARGET_IMAGE=python3-minimal 

if [ $UID != 0 ]; then 
    echo "### Running build test as unprivileged user" 
else 
    echo "### Running build test as root" 
fi 

echo "### Testing container creation" 
container=$(buildah from $BASE_IMAGE) 
if [ $? -ne 0 ]; then 
    echo "Error initializing working container" 
fi 

echo "### Testing run command" 
buildah run $container apk add --update python3 py3-pip 
if [ $? -ne 0 ]; then 
    echo "Error on run build action" 
fi 

echo "### Testing image commit" 
buildah commit $container $TARGET_IMAGE 
if [ $? -ne 0 ]; then 
    echo "Error committing final image" 
fi 

echo "### Removing working container" 
buildah rm $container 
if [ $? -ne 0 ]; then 
    echo "Error removing working container" 
fi 

echo "### Build test completed successfully!" 
exit 0
```

El mismo script de prueba puede ser ejecutado por un usuario no privilegiado y por root.

Podemos verificar la imagen recién creada ejecutando un contenedor simple que ejecuta una shell de Python:

```bash
$ podman run -it python3-minimal /usr/bin/python3 
Python 3.12.7 (main, Oct 7 2024, 11:30:19) [GCC 13.2.1 20240309] on linux 
Type "help", "copyright", "credits" or "license" for more information. 
>>>
```

Después de probar con éxito nuestra nueva instalación de Buildah, inspeccionemos los principales archivos de configuración utilizados por Buildah.

#### Archivos de configuración de Buildah

Los principales archivos de configuración de Buildah son los mismos que utiliza Podman. Se pueden aprovechar para personalizar el comportamiento de los contenedores de trabajo ejecutados en las compilaciones.

En Fedora, estos archivos de configuración son instalados por el paquete `containers-common`, y ya los cubrimos en la sección *Preparar tu entorno* en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*.

Los principales archivos de configuración utilizados por Buildah son los siguientes:

- `/usr/share/containers/mounts.conf`: Este archivo de configuración define los archivos y directorios que se montan automáticamente dentro de un contenedor de trabajo de Buildah.
- `/etc/containers/registries.conf`: Este archivo de configuración tiene la función de administrar los registros a los que se permite acceder para búsquedas, descargas (*pulls*) y cargas (*pushes*) de imágenes.
- `/usr/share/containers/policy.json`: Este archivo de configuración JSON define el comportamiento de verificación de firmas de imágenes.
- `/usr/share/containers/seccomp.json`: Este archivo de configuración JSON define las llamadas al sistema (*syscalls*) permitidas y prohibidas para un proceso en contenedor.
- `/usr/share/containers/containers.conf`: Este archivo de configuración contiene muchas configuraciones específicas de Podman, pero también lo utiliza Buildah.

En esta sección, hemos aprendido cómo preparar el entorno del host para ejecutar Buildah. En la siguiente sección, identificaremos las posibles estrategias de compilación que se pueden implementar con Buildah.

---

### Elección de nuestra estrategia de compilación

Básicamente, existen tres tipos de estrategias de compilación que podemos usar con Buildah:

1. Compilar una imagen de contenedor a partir de una imagen base existente.
2. Compilar una imagen de contenedor desde cero (*from scratch*).
3. Compilar una imagen de contenedor a partir de un Dockerfile.

Ya proporcionamos un ejemplo de la estrategia de compilación a partir de una imagen base existente en la sección *Presentación de Buildah, el compañero de Podman*. Dado que esta estrategia es bastante similar desde el punto de vista del flujo de trabajo a la compilación desde cero, centraremos nuestros ejemplos prácticos en esta última, que brinda una gran flexibilidad para crear imágenes seguras y con una huella (*footprint*) reducida.

Antes de revisar los diversos detalles técnicos en la siguiente sección, comencemos explorando todas estas estrategias a un alto nivel.

Aunque podemos encontrar muchas imágenes de contenedores precompiladas disponibles en los registros de contenedores públicos más populares, a veces es posible que no podamos encontrar una configuración, instalación o paquete particular de herramientas y servicios para nuestros contenedores; es por eso que la creación de imágenes de contenedores se convierte en un paso realmente importante que debemos practicar. Además de eso, la creación de imágenes también permite agrupar tus propios archivos de configuración para personalizar una imagen para tu uso exacto.

Asimismo, las restricciones de seguridad a menudo nos obligan a implementar imágenes con superficies de ataque reducidas y, por lo tanto, los equipos de DevOps deben saber cómo personalizar cada paso del proceso de compilación para lograr este resultado. En contextos empresariales, por motivos de optimización y seguridad, las imágenes se crean casi siempre desde cero o utilizando imágenes base mínimas listas para ser personalizadas.

Con esta conciencia en mente, comencemos con la primera estrategia de compilación.

#### Compilar una imagen de contenedor a partir de una imagen base existente

Imaginemos encontrar una imagen de contenedor precompilada y bien hecha para nuestro servidor de aplicaciones favorito que nuestra empresa utiliza ampliamente. Todas las configuraciones para esta imagen de contenedor están bien y podemos adjuntar almacenamiento a los puntos de montaje correctos para persistir los datos, etc., pero tarde o temprano, ¡podemos darnos cuenta de que faltan en la imagen del contenedor algunas herramientas particulares que usamos para solucionar problemas, o que faltan algunas bibliotecas que deberían incluirse!

En otro escenario, podríamos estar contentos con la imagen precompilada pero aún necesitar agregar contenido personalizado a ella, por ejemplo, la aplicación del cliente.

¿Cuál sería la solución en esos casos?

En este primer caso de uso, podemos extender la imagen del contenedor existente, agregando elementos y editando los archivos existentes para adaptarlos a nuestros propósitos. En los ejemplos básicos anteriores, las imágenes de Fedora y Alpine se personalizaron para servir a diferentes propósitos. Esas imágenes eran sistemas de archivos de SO genéricos sin un propósito específico, pero el mismo concepto se puede aplicar a una imagen más compleja.

En el segundo caso de uso, podemos personalizar una imagen, por ejemplo, la biblioteca predeterminada Httpd. Podemos instalar módulos PHP y luego agregar los archivos PHP de nuestra aplicación, produciendo una nueva imagen con nuestro contenido personalizado ya incorporado.

Veremos en las próximas secciones que podemos extender una imagen de contenedor existente.

Pasemos a la segunda estrategia.

#### Compilar una imagen de contenedor desde cero (*from scratch*)

La estrategia anterior sería suficiente para muchas situaciones comunes, donde podemos encontrar una imagen precompilada con la que comenzar a trabajar, pero a veces puede ser que el caso de uso particular, la aplicación o el servicio que queremos contenerizar no sea tan común o ampliamente utilizado.

Imagina tener una aplicación heredada (*legacy*) personalizada que requiere algunas bibliotecas y herramientas antiguas que ya no están incluidas en la última distribución de Linux, o que pueden haber sido reemplazadas por otras más recientes. En este escenario, es posible que debas comenzar desde una imagen de contenedor vacía y agregar todo lo necesario para tu aplicación heredada pieza por pieza.

Finalmente, `FROM scratch` es también una forma de distribuir imágenes de contenedores supermínimas que contienen solo los archivos necesarios para ejecutar tu contenedor. Incluso puedes crear imágenes de un solo archivo con solo un binario estático dentro de ellas.

Hemos aprendido en este capítulo que, en realidad, siempre comenzaremos desde una especie de imagen de contenedor inicial, por lo que esta estrategia y la anterior son prácticamente las mismas.

Pasemos a la tercera y última estrategia.

#### Compilar una imagen de contenedor a partir de un Dockerfile

En el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*, hablamos sobre la historia de la tecnología de contenedores y cómo Docker ganó impulso en ese contexto. Podman nació como un proyecto de evolución alternativo de los grandes conceptos que Docker ayudó a desarrollar hasta ahora. Una de las grandes innovaciones que Docker creó en la historia de su propio proyecto es, sin duda, el Dockerfile.

Al analizar esta estrategia a un alto nivel, podemos afirmar que incluso al usar un Dockerfile, llegaremos a una de las estrategias de compilación anteriores. La realidad no está lejos de la última suposición que hicimos, porque Buildah, bajo el capó, analizará el Dockerfile y compilará el contenedor que presentamos brevemente para las estrategias de compilación anteriores.

Entonces, en resumen, ¿hay alguna diferencia o ventaja que debamos considerar al elegir nuestra estrategia de compilación predeterminada? Obviamente, no hay una respuesta definitiva a esta pregunta. En primer lugar, siempre debemos buscar en las comunidades de contenedores alguna imagen precompilada que pueda ayudar a nuestro proceso de compilación; por otro lado, siempre podemos recurrir al proceso de compilación desde cero. Por último, pero no menos importante, podemos considerar Dockerfile para distribuir y compartir fácilmente nuestros pasos de compilación con nuestro grupo de desarrollo o con toda la comunidad de contenedores.

Esto pone fin a nuestra rápida introducción de alto nivel; ¡ahora podemos pasar a los ejemplos prácticos!

---

### Construcción de imágenes desde cero

Antes de entrar en los detalles de esta sección y aprender cómo compilar una imagen de contenedor desde cero, hagamos algunas pruebas para verificar que el Buildah instalado esté funcionando correctamente.

En primer lugar, comprobemos si nuestra caché de imágenes de Buildah está vacía:

```bash
# buildah images 
REPOSITORY TAG IMAGE ID CREATED SIZE 
# buildah containers -a 
CONTAINER ID BUILDER IMAGE ID IMAGE NAME CONTAINER NAME
```

> [!IMPORTANT]
> Podman y Buildah comparten el mismo almacenamiento de contenedores; por esta razón, si ejecutaste previamente cualquier otro ejemplo mostrado en este capítulo o libro, ¡descubrirás que la caché de almacenamiento de tu contenedor no está tan vacía!

Como aprendimos en la sección anterior, podemos aprovechar el hecho de que Buildah mostrará el nombre del contenedor de trabajo recién creado para almacenarlo fácilmente en una variable de entorno y usarlo una vez que sea necesario. Creemos un contenedor completamente nuevo desde cero:

```bash
# buildah from scratch 
# buildah images 
REPOSITORY TAG IMAGE ID CREATED SIZE 
# buildah containers 
CONTAINER ID BUILDER IMAGE ID IMAGE NAME CONTAINER NAME 
af69b9547db9 * scratch working-container
```

Como puedes ver, utilizamos las palabras clave especiales `from scratch` que le indican a Buildah que cree un contenedor vacío sin datos dentro. Si ejecutamos el comando `buildah images`, notaremos que esta imagen especial no aparece en la lista.

Comprobemos si el contenedor de trabajo creado desde cero está realmente vacío:

```bash
# buildah run working-container bash 
2021-10-26T20:15:49.000397390Z: executable file `bash` not found in $PATH: No such file or directory 
error running container: error from crun creating container for [bash]: : exit status 1 
error while running runtime: exit status 1
```

No se encontró ningún ejecutable en nuestro contenedor vacío, ¡qué sorpresa! La razón es que el contenedor de trabajo se ha creado en un sistema de archivos vacío.

Veamos cómo podemos llenar fácilmente este contenedor vacío. En el siguiente ejemplo, interactuaremos directamente con el almacenamiento subyacente, utilizando el administrador de paquetes de nuestro sistema host para instalar los binarios y las bibliotecas necesarias para ejecutar una shell de bash en nuestra imagen de contenedor.

En primer lugar, instruyamos a Buildah para que monte el almacenamiento del contenedor y compruebe dónde reside:

```bash
# buildah mount working-container 
/var/lib/containers/storage/overlay/b5034cc80252b6f4af2155f9e0a2a7e65b77dadec7217bd2442084b1f4449c1a/merged
```

> [!TIP]
> Si inicias la compilación en modo rootless, Buildah ejecutará el montaje en un namespace diferente y, por esta razón, es posible que el volumen montado no sea accesible desde el host cuando se utiliza un controlador diferente de `vfs`. Como alternativa, puedes ejecutar los comandos en una shell `podman unshare`.

¡Genial! Ahora que lo hemos encontrado, podemos aprovechar el administrador de paquetes del host para instalar todos los paquetes necesarios en esta carpeta raíz, que será la ruta raíz de nuestra imagen de contenedor:

```bash
# scratchmount=$(buildah mount working-container) 
# dnf install --installroot $scratchmount --releasever 40 bash coreutils --setopt install_weak_deps=false -y
```

> [!IMPORTANT]
> Si estás ejecutando el comando anterior en una versión de Fedora diferente de la versión 40 (por ejemplo, la versión 39), debes importar las claves públicas GPG de Fedora 40 o usar la opción adicional `--nogpgcheck`.

En primer lugar, guardaremos la ruta de directorio muy larga en una variable de entorno y luego ejecutaremos el administrador de paquetes `dnf`, pasando la ruta de directorio recién obtenida como el directorio raíz de instalación, estableciendo la versión de lanzamiento de nuestro sistema operativo Fedora, especificando los paquetes que queremos instalar (`bash` y `coreutils`) y, finalmente, deshabilitando las dependencias débiles, aceptando todos los cambios en el sistema.

El comando debería terminar con una declaración de `Complete!`; una vez hecho esto, intentemos nuevamente con el mismo comando que vimos fallar anteriormente en esta sección:

```bash
# buildah run working-container bash 
bash-5.2# cat /etc/fedora-release 
Fedora release 40 (Forty)
```

¡Funcionó! Acabamos de instalar una shell de Bash en nuestro contenedor vacío. Veamos ahora cómo terminar la creación de nuestra imagen con algunos otros pasos de configuración. En primer lugar, a nuestra imagen de contenedor final, debemos agregar un comando para que se ejecute una vez que esté en funcionamiento. Por esta razón, crearemos un archivo de script Bash con algunos comandos básicos:

```bash
# cat command.sh 
#!/bin/bash 
cat /etc/fedora-release 
/usr/bin/date
```

Hemos creado un archivo de script Bash que imprime la versión de Fedora del contenedor y la fecha del sistema. El archivo debe tener permisos de ejecución antes de copiarse:

```bash
# chmod +x command.sh
```

Ahora que hemos llenado nuestro almacenamiento de contenedor subyacente con todos los paquetes base necesarios, podemos desmontar el almacenamiento de `working-container` y usar el comando `buildah copy` para inyectar archivos desde el host al contenedor:

```bash
# buildah unmount working-container 
af69b9547db93a7dc09b96a39bf5f7bc614a7ebd29435205d358e09ac99857bc 
# buildah copy working-container ./command.sh /usr/bin 
659a229354bdef3f9104208d5812c51a77b2377afa5ac819e3c3a1a2887eb9f7
```

El comando `buildah copy` nos brinda la capacidad de trabajar con el almacenamiento subyacente sin preocuparnos por montarlo o manipularlo bajo el capó.

Ahora estamos listos para completar nuestra imagen de contenedor agregándole algunos metadatos:

```bash
# buildah config --cmd /usr/bin/command.sh working-container 
# buildah config --created-by "Podman for DevOps example" working-container 
# buildah config --label name=fedora-date working-container
```

Comenzamos con la opción `cmd` y luego agregamos algunos metadatos descriptivos. ¡Finalmente podemos confirmar (*commit*) `working-container` en una imagen!

```bash
# buildah commit working-container fedora-date 
Getting image source signatures 
Copying blob 939ac17066d4 done 
Copying config e24a2fafde done 
Writing manifest to image destination 
Storing signatures 
e24a2fafdeb5658992dcea9903f0640631ac444271ed716d7f749eea7a651487
```

Limpiemos el entorno y verifiquemos las imágenes de contenedor disponibles en el host:

```bash
# buildah rm working-container 
af69b9547db93a7dc09b96a39bf5f7bc614a7ebd29435205d358e09ac99857bc
```

Ahora podemos inspeccionar los detalles de la imagen de contenedor recién creada:

```bash
# podman images 
REPOSITORY TAG IMAGE ID CREATED SIZE 
localhost/fedora-date latest e24a2fafdeb5 About a minute ago 366 MB 
# podman inspect localhost/fedora-date:latest 
[...omitted output] 
        "Labels": { 
            "io.buildah.version": "1.37.3", 
            "name": "fedora-date" 
        }, 
        "Annotations": { 
            "org.opencontainers.image.base.digest": "", 
            "org.opencontainers.image.base.name": "" 
        }, 
        "ManifestType": "application/vnd.oci.image.manifest.v1+json", 
        "User": "", 
        "History": [ 
            { 
                "created": "2024-11-01T13:51:50.170096562Z", 
                "created_by": "Podman for DevOps example" 
            } 
        ], 
        "NamesHistory": [ 
            "localhost/fedora-date:latest" 
        ] 
    } 
]
```

Como podemos ver en la salida anterior, la imagen del contenedor tiene muchos metadatos que pueden decirnos muchos detalles. Algunos de ellos los configuramos a través de los comandos anteriores, como las etiquetas `created_by`, `name` y `Cmd`; las otras etiquetas son completadas automáticamente por Buildah.

¡Finalmente, ejecutemos nuestra nueva imagen de contenedor con Podman!

```bash
# podman run -ti localhost/fedora-date:latest 
Fedora release 40 (Forty) 
Tue Nov 01 21:18:29 UTC 2024
```

Esto pone fin a nuestro viaje en la creación de una imagen de contenedor desde cero. Como vimos, este no es un método típico para crear una imagen de contenedor; en muchos escenarios y para varios casos de uso, puede ser suficiente comenzar con una imagen base de SO, como `fedora` o `alpine`, y luego agregar los paquetes requeridos, utilizando los administradores de paquetes respectivos disponibles en esas imágenes.

> [!TIP]
> Algunas distribuciones de Linux también proporcionan imágenes de contenedores base en una variante mínima (por ejemplo, `fedora-minimal`) que reduce la cantidad de paquetes instalados, así como el tamaño de la imagen del contenedor de destino. Para obtener más información, consulta [https://www.docker.com/](https://www.docker.com/) y [https://quay.io/](https://quay.io/).

Inspeccionemos ahora cómo compilar imágenes a partir de Dockerfiles con Buildah.

---

### Construcción de imágenes a partir de un Dockerfile

Como describimos anteriormente en este capítulo, el Dockerfile puede ser una opción fácil para crear y compartir los pasos de compilación para crear una imagen de contenedor y, por esta razón, es muy fácil encontrar muchos Dockerfiles de origen en la red.

El primer paso de esta actividad es crear un Dockerfile simple con el que trabajar. Creemos un Dockerfile para crear un servidor web en contenedor:

```bash
# mkdir webserver 
# cd webserver/ 
[webserver]# vi Dockerfile 
[webserver]# cat Dockerfile 
# Start from latest fedora container base image 
FROM fedora:latest 

# Update the container base image 
RUN echo "Updating all fedora packages"; dnf -y update; dnf -y clean all 

# Install the httpd package 
RUN echo "Installing httpd"; dnf -y install httpd 

# Expose the http port 80 
EXPOSE 80 

# Set the default command to run once the container will be started 
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Observando la salida anterior, primero creamos un nuevo directorio y, dentro, creamos un archivo de texto llamado `Dockerfile`. Después de eso, insertamos las diversas palabras clave y pasos comúnmente utilizados en la definición de un Dockerfile completamente nuevo; cada paso y palabra clave tiene un comentario de descripción dedicado en la parte superior, por lo que el archivo debería ser fácil de leer.

Solo para recapitular, estos son los pasos contenidos en nuestro nuevo Dockerfile:

1. Comenzar desde la última imagen base de contenedor de Fedora.
2. Actualizar todos los paquetes para la imagen base del contenedor.
3. Instalar el paquete `httpd`.
4. Exponer el puerto HTTP 80.
5. Establecer el comando predeterminado para ejecutarse una vez que se inicie el contenedor.

Como se vio anteriormente en este capítulo, Buildah proporciona un comando dedicado `buildah build` para iniciar una compilación desde un Dockerfile.

Veamos cómo funciona:

```bash
[webserver]# buildah build -f Dockerfile -t myhttpdservice . 
STEP 1/6: FROM fedora:latest 
Resolved "fedora" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf) 
Trying to pull registry.fedoraproject.org/fedora:latest... 
Getting image source signatures 
Copying blob 944c4b241113 done 
Copying config 191682d672 done 
Writing manifest to image destination 
Storing signatures 
STEP 2/6: MAINTAINER podman-book # this should be an email 
STEP 3/6: RUN echo "Updating all fedora packages"; dnf -y update; dnf -y clean all 
Updating all fedora packages 
Fedora 40 - x86_64 16 MB/s | 74 MB 00:04 
... 
STEP 4/6: RUN echo "Installing httpd"; dnf -y install httpd 
Installing httpd 
Fedora 40 - x86_64 20 MB/s | 74 MB 00:03 
... 
STEP 5/6: EXPOSE 80 
STEP 6/6: CMD ["/usr/sbin/httpd", "-DFOREGROUND"] 
COMMIT myhttpdservice 
Getting image source signatures 
Copying blob 7500ce202ad6 skipped: already exists 
Copying blob 51b52d291273 done 
Copying config 14a2226710 done 
Writing manifest to image destination 
Storing signatures 
--> 14a2226710e 
Successfully tagged localhost/myhttpdservice:latest 
14a2226710e7e18d2e4b6478e09a9f55e60e0666dd8243322402ecf6fd1eaa0d
```

Como podemos ver en la salida anterior, pasamos las siguientes opciones al comando `buildah build`:

- `-f`: Para definir el nombre del Dockerfile. El nombre de archivo predeterminado es `Dockerfile`, por lo que en nuestro caso, podemos omitir esta opción porque nombramos el archivo con el valor predeterminado.
- `-t`: Para definir el nombre y la etiqueta de la imagen que estamos compilando. En nuestro caso, solo estamos definiendo el nombre. La imagen se etiquetará como `latest` de forma predeterminada.
- Finalmente, como última opción, debemos especificar el directorio donde Buildah necesita trabajar y buscar el Dockerfile. En nuestro caso, estamos pasando el directorio `.`, que representa el directorio de trabajo actual.

Por supuesto, estas no son las únicas opciones que nos brinda Buildah para configurar la compilación; veremos algunas de ellas más adelante en esta sección.

Volviendo al comando que acabamos de ejecutar, como podemos ver en la salida, todos los pasos definidos en el Dockerfile se han ejecutado en el orden exacto y se han impreso con un número fraccionario determinado para mostrar los pasos intermedios frente al número total. En total, se ejecutaron seis pasos.

Podemos verificar el resultado de nuestro comando enumerando las imágenes con el comando `buildah images`:

```bash
[webserver]# buildah images 
REPOSITORY TAG IMAGE ID CREATED SIZE 
localhost/myhttpdservice latest 14a2226710e7 2 minutes ago 497 MB
```

Como podemos ver, nuestra imagen de contenedor se acaba de crear con la etiqueta `latest`; intentemos ejecutarla:

```bash
# podman run -d localhost/myhttpdservice:latest 
133584ab526faaf7af958da590e14dd533256b60c10f08acba6c1209ca05a885 
# podman logs 133584ab526faaf7af958da590e14dd533256b60c10f08acba6c1209ca05a885 
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.88.0.4. Set the 'ServerName' directive globally to suppress this message 
# curl 10.88.0.4 
<!doctype html> 
<html> 
<head> 
<meta charset='utf-8'> 
<meta name='viewport' content='width=device-width, initial-scale=1'> 
<title>Test Page for the HTTP Server on Fedora</title> 
<style type="text/css"> 
...
```

Mirando la salida, acabamos de ejecutar nuestro contenedor en modo desconectado (*detached*); después de eso, inspeccionamos los registros para averiguar la dirección IP que debemos pasar como argumento para el comando de prueba `curl`.

Acabamos de ejecutar el contenedor como usuario root en nuestra estación de trabajo, y el contenedor acaba de recibir una dirección IP interna en la interfaz de red de contenedores de Podman. Podemos verificar que la dirección IP sea parte de esa red ejecutando los siguientes comandos:

```bash
# ip a show dev cni-podman0 
14: cni-podman0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000 
    link/ether c6:bc:ba:7c:d3:0c brd ff:ff:ff:ff:ff:ff 
    inet 10.88.0.1/16 brd 10.88.255.255 scope global cni-podman0 
       valid_lft forever preferred_lft forever 
    inet6 fe80::c4bc:baff:fe7c:d30c/64 scope link 
       valid_lft forever preferred_lft forever
```

Como podemos ver, la dirección IP del contenedor se tomó de la red informada en la salida `10.88.0.1/16` anterior.

Como anticipamos, el comando `buildah build` tiene muchas otras opciones que pueden ser útiles al desarrollar y crear imágenes de contenedores completamente nuevas. Exploremos una de ellas que vale la pena mencionar: `--layers`.

Ya aprendimos cómo usar esta opción con Podman anteriormente en este capítulo. A partir de la versión 1.2 de Buildah, el equipo de desarrollo agregó esta excelente opción que nos brinda la capacidad de habilitar o deshabilitar el mecanismo de almacenamiento en caché de capas. La configuración predeterminada establece la opción `--layers` en `false`, lo que significa que Buildah no mantendrá capas intermedias, lo que resulta en una compilación que aplasta (*squashes*) todos los cambios en una sola capa.

También es posible establecer la gestión de las capas con una variable de entorno; por ejemplo, para habilitar el almacenamiento en caché de capas, ejecuta `export BUILDAH_LAYERS=true`.

Si bien conservar las capas intermedias consume espacio de almacenamiento inicial, el impacto real en tu sistema es más matizado. Mantener estas capas permite compartir capas entre múltiples imágenes diferentes, lo que en realidad puede ahorrar un espacio y ancho de banda significativos; si 10 imágenes comparten las mismas capas base, esas capas se almacenan y descargan solo una vez.

Sin embargo, hay un par de compensaciones (*trade-offs*) distintas a considerar:

- **La desventaja de muchas capas**: Cada capa adicional agrega complejidad al sistema de archivos de unión (*union filesystem*). Esto puede conducir a tiempos de inicio de contenedor ligeramente más lentos y una mayor sobrecarga cuando el kernel fusiona las capas en una sola vista.
- **La desventaja de aplastar (*squashing*)**: Si bien aplastar capas en una sola puede mejorar la velocidad de inicio y eliminar permanentemente los archivos eliminados en capas superiores (archivos que de otro modo permanecerían ocultos pero presentes en una imagen multicapa), destruye la deduplicación. Una imagen aplastada no puede compartir sus capas con otras, lo que significa que el almacenamiento total consumido en tu host puede aumentar si mantienes varias imágenes similares.

En última instancia, mantener capas intermedias es un equilibrio: cambias un poco de complejidad del sistema de archivos por ganancias masivas en velocidad de compilación y eficiencia de almacenamiento.

---

### Resumen

En este capítulo, exploramos un tema fundamental de la gestión de contenedores: su creación. Este paso es obligatorio si queremos personalizar, mantener actualizada y administrar nuestra infraestructura de contenedores correctamente. Aprendimos que Podman a menudo se asocia con otra herramienta llamada Buildah que puede ayudarnos en el proceso de creación de imágenes de contenedores. Esta herramienta tiene muchas opciones, como Podman, y comparte muchas de ellas con él (¡almacenamiento incluido!). Finalmente, revisamos las diferentes estrategias que Buildah nos ofrece para compilar nuevas imágenes de contenedores, y una de ellas es heredada del ecosistema de Docker: el Dockerfile.

Este capítulo es solo una introducción al tema de la compilación de imágenes de contenedores; ¡descubriremos técnicas más avanzadas en el próximo capítulo!

---

### Lecturas adicionales

- Tutoriales del proyecto Buildah: [https://github.com/containers/buildah/tree/main/docs/tutorials](https://github.com/containers/buildah/tree/main/docs/tutorials)
- Cómo usar Podman dentro de un contenedor: [https://www.redhat.com/sysadmin/podman-inside-container](https://www.redhat.com/sysadmin/podman-inside-container)
- Cómo crear imágenes de contenedores diminutas: [https://www.redhat.com/sysadmin/tiny-containers](https://www.redhat.com/sysadmin/tiny-containers)
