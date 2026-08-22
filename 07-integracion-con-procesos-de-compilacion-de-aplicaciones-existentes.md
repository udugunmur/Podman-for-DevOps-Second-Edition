# Parte 2: Construyendo tu Contenedor desde Cero con Buildah

## Capítulo 7: Integración con Procesos de Compilación de Aplicaciones Existentes

Después de aprender cómo crear imágenes de contenedores personalizadas usando Podman y Buildah, ahora podemos centrarnos en casos de uso especiales que pueden hacer que nuestros flujos de trabajo de compilación sean más eficientes y portátiles. Por ejemplo, las imágenes pequeñas son un requisito muy común en un entorno empresarial, por razones de rendimiento y seguridad. Exploraremos cómo lograr este objetivo dividiendo el proceso de compilación en diferentes etapas.

Este capítulo también intentará descubrir escenarios donde no se espera que Buildah se ejecute directamente en la máquina de un desarrollador, sino que esté impulsado por un orquestador de contenedores o integrado dentro de aplicaciones personalizadas que deban llamar a sus bibliotecas o interfaz de línea de comandos (CLI).

En este capítulo, vamos a cubrir los siguientes temas principales:

- Compilaciones de contenedores multietapa (*multistage*)
- Ejecución de Buildah dentro de un contenedor
- Integración de Buildah con compiladores personalizados

---

### Requisitos técnicos

Antes de continuar con este capítulo, se requiere una máquina con una instalación funcional de Podman y Buildah. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40 o versiones posteriores, pero se pueden reproducir en el sistema operativo elegido por el lector.

Una buena comprensión de los temas tratados en el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), *Conoce Buildah – Construcción de Contenedores desde Cero*, será útil para asimilar fácilmente conceptos sobre compilaciones, tanto con comandos nativos de Buildah como desde Dockerfiles.

---

### Compilaciones de contenedores multietapa (*multistage*)

Hasta ahora, hemos aprendido a crear compilaciones con Podman y Buildah usando Dockerfiles o comandos nativos de Buildah que liberan posibles técnicas de compilación avanzadas. Todavía hay un punto importante que no hemos discutido: el tamaño de las imágenes.

Al crear una nueva imagen, siempre debemos cuidar su tamaño final, que es el resultado del número total de capas y el número de archivos modificados dentro de ellas.

Las imágenes mínimas con un tamaño reducido tienen la gran ventaja de poder descargarse más rápido desde los registros. No obstante, una imagen grande consumirá mucho espacio valioso en el disco del almacén local del host.

Ya mostramos ejemplos de algunas mejores prácticas para mantener las imágenes compactas en tamaño, como compilar desde cero (*from scratch*), limpiar las memorias caché del administrador de paquetes y reducir la cantidad de instrucciones `RUN`, `COPY` y `ADD` al mínimo necesario. Sin embargo, ¿qué sucede cuando necesitamos compilar una aplicación a partir de su código fuente y crear una imagen final con los artefactos resultantes?

Digamos que necesitamos compilar una aplicación de Go en un contenedor: debemos comenzar desde una imagen base que incluya los entornos de ejecución de Go, copiar el código fuente y compilar para producir el binario final con una serie de pasos intermedios, sobre todo descargando todos los paquetes de Go necesarios dentro de la memoria caché de la imagen. Al final de la compilación, debemos limpiar todo el código fuente y las dependencias descargadas y colocar el binario final (que está vinculado estáticamente en Go) en un directorio de trabajo. Todo funcionará, pero la imagen final seguirá incluyendo los entornos de ejecución de Go incluidos en la imagen base, que ya no son necesarios al final del proceso de compilación.

Cuando se introdujo Docker y los Dockerfiles ganaron impulso, este problema fue eludido de diferentes maneras por los equipos de DevOps que luchaban por mantener las imágenes mínimas. Por ejemplo, las compilaciones binarias eran una forma de inyectar el artefacto final compilado externamente dentro de la imagen construida. Este enfoque resuelve el problema del tamaño de la imagen, pero elimina la ventaja de un entorno estandarizado para las compilaciones proporcionado por las imágenes de tiempo de ejecución/compilador.

Un mejor enfoque es compartir volúmenes entre contenedores y hacer que la imagen del contenedor final tome los artefactos compilados de una primera imagen de compilación.

Para proporcionar un enfoque estandarizado, Docker, y luego las especificaciones de OCI, introdujeron el concepto de compilaciones multietapa (*multistage builds*). Las compilaciones multietapa, como sugiere el nombre, permiten a los usuarios crear compilaciones con múltiples etapas utilizando diferentes instrucciones `FROM` y hacer que las imágenes posteriores tomen contenidos de las anteriores.

En las siguientes subsecciones, exploraremos cómo lograr este resultado con Dockerfiles/Containerfiles y con los comandos nativos de Buildah.

#### Compilaciones multietapa con Dockerfiles

El primer enfoque para las compilaciones multietapa es crear múltiples etapas en un solo Dockerfile/Containerfile, con cada bloque comenzando con una instrucción `FROM`.

Las etapas de compilación pueden copiar archivos y carpetas de las anteriores utilizando la opción `--from` para especificar la etapa de origen.

Los siguientes ejemplos muestran cómo crear una compilación multietapa mínima para la aplicación Go, con la primera etapa actuando como un contexto de compilación puro y la segunda etapa copiando el artefacto final dentro de una imagen mínima.

Este ejemplo se define en el siguiente archivo: `Chapter07/http_hello_world/Dockerfile`:

```dockerfile
# Builder image
FROM docker.io/library/golang
# Copy files for build
COPY go.mod /go/src/hello-world/
COPY main.go /go/src/hello-world/
# Set the working directory
WORKDIR /go/src/hello-world
# Download dependencies
RUN go get -d -v ./...
# Install the package
RUN go build -v
# Runtime image
FROM registry.access.redhat.com/ubi9/ubi-micro:latest
COPY --from=0 /go/src/hello-world/hello-world /
EXPOSE 8080
CMD ["/hello-world"]
```

La primera etapa copia el archivo fuente `main.go` y el archivo `go.mod` para administrar las dependencias del módulo Go. Después de descargar los paquetes de dependencias (`go get -d -v ./...`), se compila la aplicación final (`go build -v ./...`).

La segunda etapa toma el artefacto final (`/go/src/hello-world/hello-world`) y lo copia en la raíz de la nueva imagen. Para especificar que el archivo de origen debe copiarse desde la primera etapa, se utiliza la sintaxis `--from=0`.

En la primera etapa, utilizamos la imagen oficial `docker.io/library/golang`, que incluye la última versión del lenguaje de programación Go. En la segunda etapa, utilizamos la imagen `ubi-micro`, una imagen mínima de Red Hat con una huella reducida, optimizada para microservicios y binarios enlazados estáticamente. La Universal Base Image (UBI) se cubrirá con mayor detalle en el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/8), *Elección de la Imagen Base del Contenedor*.

La aplicación Go que se detalla a continuación es un servidor web básico que escucha en el puerto 8080/tcp e imprime una página HTML diseñada con el mensaje "Hello World!" cuando recibe una solicitud `GET /`:

> [!NOTE]
> Para el propósito de este libro, no es necesario saber escribir o entender el lenguaje de programación Go. Sin embargo, una comprensión básica de la sintaxis y la lógica del lenguaje resultará muy útil, ya que la mayor parte del software relacionado con contenedores (como Podman, Docker, Buildah, Skopeo, Kubernetes y OpenShift) está escrito en Go.

`Chapter07/http_hello_world/main.go`:

```go
package main

import (
	"log"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	log.Printf("%s %s %s\n", r.RemoteAddr, r.Method, r.URL)
	w.Header().Set("Content-Type", "text/html")
	w.Write([]byte("<html>\n<body>\n"))
	w.Write([]byte("<p>Hello World!</p>\n"))
	w.Write([]byte("</body>\n</html>\n"))
}

func main() {
	http.HandleFunc("/", handler)
	log.Println("Starting http server")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

La aplicación se puede compilar utilizando Podman o Buildah. En este ejemplo, elegimos compilar la aplicación con Buildah:

```bash
$ cd http_hello_world
$ buildah build -t hello-world .
```

Finalmente, podemos verificar el tamaño de la imagen resultante:

```bash
$ buildah images --format '{{.Name}} {{.Size}}' \
  localhost/hello-world
localhost/hello-world 36.1 MB
```

¡La imagen final tiene un tamaño de solo 36 MB!

Podemos mejorar nuestro Dockerfile agregando nombres personalizados a las imágenes base usando la palabra clave `AS`. El siguiente ejemplo es una reelaboración del Dockerfile anterior siguiendo este enfoque:

`Chapter07/http_hello_world/Dockerfile-builder`:

```dockerfile
# Builder image
FROM docker.io/library/golang AS builder
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
FROM registry.access.redhat.com/ubi9/ubi-micro:latest AS srv
COPY --from=builder /go/src/hello-world/hello-world /
EXPOSE 8080
CMD ["/hello-world"]
```

En el ejemplo anterior, el nombre de la imagen de compilación se establece como `builder`, mientras que la imagen final se denomina `srv`. Curiosamente, la instrucción `COPY` ahora puede especificar el compilador utilizando el nombre personalizado con la opción `--from=builder`.

Podemos compilar nuevamente la imagen del contenedor con el siguiente comando:

```bash
$ cd http_hello_world
$ buildah build -f ./Dockerfile-builder -t hello-world-v2 .
```

Las compilaciones de Dockerfile/Containerfile son el enfoque más común, pero aún carecen de cierta flexibilidad a la hora de implementar un flujo de trabajo de compilación personalizado. Para esos casos de uso especiales, los comandos nativos de Buildah vienen a nuestro rescate.

#### Compilaciones multietapa con comandos nativos de Buildah

Como se mencionó anteriormente, la función de compilación multietapa es un gran enfoque para producir imágenes con una huella pequeña y una superficie de ataque reducida. Para brindar una mayor flexibilidad durante el proceso de compilación, los comandos nativos de Buildah vienen a nuestro rescate. Como mencionamos anteriormente en el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), *Conoce Buildah – Construcción de Contenedores desde Cero*, Buildah ofrece una serie de comandos que replican el comportamiento de las instrucciones de Dockerfile, ofreciendo así un mayor control sobre el proceso de compilación cuando esos comandos se incluyen en scripts o automatizaciones.

El mismo concepto se aplica cuando se trabaja con compilaciones multietapa, donde también podemos aplicar pasos adicionales entre las etapas. Por ejemplo, podemos montar el sistema de archivos superpuesto (*overlay*) del contenedor de compilación y extraer el artefacto compilado para liberar paquetes alternativos, todo antes de compilar la imagen final de tiempo de ejecución.

El siguiente ejemplo compila la misma aplicación Go hello-world traduciendo las instrucciones anteriores de Dockerfile en comandos nativos de Buildah, con todo dentro de un script de shell simple:

`Chapter07/http_hello_world/buildah-commands.sh`:

```bash
#!/bin/bash
# Define builder and runtime images
BUILDER=docker.io/library/golang
RUNTIME=registry.access.redhat.com/ubi9/ubi-micro:latest

# Create builder container
container1=$(buildah from $BUILDER)

# Copy files from host
if [ -f go.mod ]; then
    buildah copy $container1 'go.mod' '/go/src/hello-world/'
else
    exit 1
fi

if [ -f main.go ]; then
    buildah copy $container1 'main.go' '/go/src/hello-world/'
else
    exit 1
fi

# Configure and start build
buildah config --workingdir /go/src/hello-world $container1
buildah run $container1 go get -d -v ./...
buildah run $container1 go build -v ./...

# Create runtime container
container2=$(buildah from $RUNTIME)

# Copy files from the builder container
buildah copy --chown=1001:1001 \
    --from=$container1 $container2 \
    '/go/src/hello-world/hello-world' '/'

# Configure exposed ports
buildah config --port 8080 $container2

# Configure default CMD
buildah config --cmd /hello-world $container2

# Configure default user
buildah config --user=1001 $container2

# Commit final image
buildah commit $container2 helloworld

# Remove build containers
buildah rm $container1 $container2
```

En el ejemplo anterior, creamos los dos contenedores de trabajo y utilizamos las variables relacionadas `container1` y `container2` que almacenan los identificadores de los contenedores.

Además, observa el comando `buildah copy`, donde hemos definido el contenedor de origen con la opción `--from`, y utilizado la opción `--chown` para definir los propietarios de usuario y grupo del recurso copiado. Este enfoque demuestra ser más flexible que el flujo de trabajo basado en Dockerfile, ya que podemos enriquecer nuestro script con variables, condicionales y bucles.

Por ejemplo, realizamos pruebas con la condición `if` en el script de Bash para verificar la existencia de los archivos `go.mod` y `main.go` antes de copiarlos dentro del contenedor de trabajo dedicado a la compilación.

Agreguemos ahora una característica adicional al script. En el siguiente ejemplo, evolucionamos el anterior agregando control de versiones semántico (*semantic versioning*) para la compilación y creando un archivo comprimido de la versión antes de comenzar la compilación de la imagen de tiempo de ejecución final:

> [!IMPORTANT]
> El concepto de control de versiones semántico tiene como objetivo proporcionar una forma clara y estandarizada de gestionar las versiones de software y la gestión de dependencias. Es un conjunto de reglas estándar cuyo propósito es definir cómo se aplican las versiones de lanzamiento de software, y sigue el patrón de versiones X.Y.Z, donde X es la versión principal (*major*), Y es la versión secundaria (*minor*) y Z es la versión de parche (*patch*). Para obtener más información, consulta las especificaciones oficiales: [https://semver.org/](https://semver.org/).

`Chapter07/http_hello_world/buildah-commands-release.sh`:

```bash
#!/bin/bash
# Define builder and runtime images
BUILDER=docker.io/library/golang
RUNTIME=registry.access.redhat.com/ubi9/ubi-micro:latest
RELEASE=1.0.0

# Create builder container
container1=$(buildah from $BUILDER)

# Copy files from host
if [ -f go.mod ]; then
    buildah copy $container1 'go.mod' '/go/src/hello-world/'
else
    exit 1
fi

if [ -f main.go ]; then
    buildah copy $container1 'main.go' '/go/src/hello-world/'
else
    exit 1
fi

# Configure and start build
buildah config --workingdir /go/src/hello-world $container1
buildah run $container1 go get -d -v ./...
buildah run $container1 go build -v ./...

# Extract build artifact and create a version archive
buildah unshare --mount mnt=$container1 \
    sh -c 'cp $mnt/go/src/hello-world/hello-world .'

cat > README << EOF
Version $RELEASE release notes:
- Implement basic features
EOF

tar zcf hello-world-${RELEASE}.tar.gz hello-world README
rm -f hello-world README

# Create runtime container
container2=$(buildah from $RUNTIME)

# Copy files from the builder container
buildah copy --chown=1001:1001 \
    --from=$container1 $container2 \
    '/go/src/hello-world/hello-world' '/'

# Configure exposed ports
buildah config --port 8080 $container2

# Configure default CMD
buildah config --cmd /hello-world $container2

# Configure default user
buildah config --user=1001 $container2

# Commit final image
buildah commit $container2 helloworld:$RELEASE

# Remove build containers
buildah rm $container1 $container2
```

Primero, agregamos una variable `RELEASE` que rastrea la versión de lanzamiento de la aplicación. Luego, extrajimos el artefacto de compilación usando el comando `buildah unshare`, seguido de la opción `--mount` para pasar el punto de montaje del contenedor. El namespace de usuario de unshare fue necesario para que el script pudiera ejecutarse en modo rootless.

Después de extraer el artefacto, creamos un archivo comprimido gzip usando la variable `$RELEASE` dentro del nombre del archivo y eliminamos los archivos temporales.

Finalmente, iniciamos la compilación de la imagen de tiempo de ejecución y confirmamos (*committed*) usando la variable `$RELEASE` nuevamente como la etiqueta de la imagen.

En esta sección, hemos aprendido a ejecutar compilaciones multietapa con Buildah utilizando tanto Dockerfiles/Containerfiles como comandos nativos. En la siguiente sección, aprenderemos cómo aislar las compilaciones de Buildah dentro de un contenedor.

---

### Ejecución de Buildah dentro de un contenedor

Podman y Buildah siguen un enfoque fork/exec que los hace muy fáciles de ejecutar dentro de un contenedor, incluidos los escenarios de contenedores rootless.

Existen muchos casos de uso que implican la necesidad de compilaciones en contenedores. Hoy en día, uno de los escenarios de adopción más comunes es el flujo de trabajo de compilación de aplicaciones que se ejecuta sobre un clúster de Kubernetes.

Kubernetes es básicamente un orquestador de contenedores que administra la programación de contenedores desde un plano de control sobre un conjunto de nodos de trabajo que ejecutan un motor de contenedores compatible con la Container Runtime Interface (CRI). Su diseño permite una gran flexibilidad para personalizar redes, almacenamiento y tiempos de ejecución, y conduce al gran florecimiento de proyectos paralelos que ahora se están incubando o madurando dentro de la Cloud Native Computing Foundation (CNCF).

Kubernetes Vanilla (que es la versión comunitaria básica sin ninguna personalización ni complementos) no tiene una función de compilación nativa, pero ofrece el marco adecuado para implementar una. Con el tiempo, aparecieron muchas soluciones tratando de abordar esta necesidad.

Por ejemplo, Red Hat OpenShift introdujo, en la época en que se lanzó Kubernetes 1.0, sus propias API de compilación y el kit de herramientas Source-to-Image para crear imágenes de contenedores a partir del código fuente directamente sobre el clúster de OpenShift.

Otras soluciones interesantes son kaniko de Google, que es una herramienta de compilación para crear imágenes de contenedores dentro de un clúster de Kubernetes que ejecuta cada paso de compilación dentro del espacio de usuario, y Cloud Native Buildpacks (CNB), que ofrecen un enfoque similar a Source-to-Image con funciones avanzadas de gestión de listas de materiales únicas y procesos múltiples.

Además de utilizar soluciones ya implementadas, podemos diseñar la nuestra ejecutando Buildah dentro de contenedores orquestados por Kubernetes. También podemos aprovechar el diseño preparado para rootless para implementar flujos de trabajo de compilación seguros.

Es posible ejecutar canalizaciones de CI/CD sobre un clúster de Kubernetes e integrar compilaciones en contenedores dentro de una canalización. Uno de los proyectos más interesantes de la CNCF, Tekton Pipelines, ofrece un enfoque nativo de la nube para lograr este objetivo. Tekton permite ejecutar canalizaciones que son impulsadas por recursos personalizados de Kubernetes: API especiales que amplían el conjunto de API básico.

Las canalizaciones de Tekton están compuestas por muchas tareas diferentes, y los usuarios pueden crear las suyas propias o tomarlas de Tekton Hub ([https://hub.tekton.dev/](https://hub.tekton.dev/)), un repositorio gratuito donde muchas tareas preparadas previamente están disponibles para ser consumidas inmediatamente, incluidos ejemplos de Buildah ([https://hub.tekton.dev/tekton/task/buildah](https://hub.tekton.dev/tekton/task/buildah)).

Los ejemplos anteriores son útiles para comprender por qué las compilaciones en contenedores son importantes. En este libro, queremos centrarnos en los detalles de la ejecución de compilaciones dentro de contenedores, prestando especial atención a las restricciones relacionadas con la seguridad.

#### Ejecución de contenedores Buildah rootless con almacenes de volúmenes

Para los ejemplos de esta subsección, se utilizará la imagen estable de Buildah de origen ascendente (*upstream*) `quay.io/buildah/stable`. Esta imagen ya incorpora el binario estable más reciente de Buildah.

Ejecutemos nuestro primer ejemplo con un contenedor rootless que compila el contenido del directorio `~/build` en el host y almacena la salida en un volumen local llamado `storevol`:

```bash
$ podman run --device /dev/fuse \
    -v ./http_hello_world:/build:z \
    -v storevol:/var/lib/containers quay.io/buildah/stable \
    buildah build -t build_test1 /build
```

Este ejemplo contiene algunas opciones peculiares que merecen atención:

- La opción `--device /dev/fuse` carga el módulo del kernel fuse en el contenedor, que es necesario para ejecutar comandos de fuse-overlay y montar el sistema de archivos del contenedor.
- La opción `-v ~/build:/build:z` monta por bind el directorio `/build` dentro del contenedor, asignando el etiquetado SELinux adecuado con el sufijo `:z`.
- La opción `-v storevol:/var/lib/containers` crea un volumen nuevo montado en el almacén de contenedores predeterminado, donde se crean todas las capas.

Cuando se completa la compilación, podemos ejecutar un nuevo contenedor utilizando el mismo volumen e inspeccionar o manipular la imagen compilada:

```bash
$ podman run --rm -v storevol:/var/lib/containers quay.io/buildah/stable buildah images
REPOSITORY                                TAG      IMAGE ID       CREATED          SIZE
localhost/build_test1                     latest   3605829966b5   41 seconds ago   33.9 MB
registry.access.redhat.com/ubi9/ubi-micro latest   e279e18c7ef8   3 days ago       26.4 MB
docker.io/library/golang                  latest   0457bb691895   9 days ago       862 MB
```

Hemos creado con éxito una imagen cuyas capas se han almacenado dentro del volumen `storevol`. Para enumerar recursivamente el contenido del almacén, podemos extraer el punto de montaje del volumen con el comando `podman volume inspect`:

```bash
$ ls -alR \
    $(podman volume inspect storevol --format '{{.Mountpoint}}')
```

A partir de ahora, es posible lanzar un nuevo contenedor Buildah para autenticarse con el registro remoto, etiquetar la imagen y enviarla. En el siguiente ejemplo, Buildah etiqueta la imagen resultante, se autentica con el registro remoto y luego envía la imagen:

```bash
$ podman run --rm -v storevol:/var/lib/containers \
    quay.io/buildah/stable \
    sh -c 'buildah tag build_test1 \
    registry.example.com/build_test1 \
    && buildah login -u=<USERNAME> -p=<PASSWORD> \
    registry.example.com && \
    buildah push registry.example.com/build_test1'
```

Cuando la imagen se haya enviado correctamente, finalmente será seguro eliminar el volumen:

```bash
# podman volume rm storevol
```

A pesar de funcionar perfectamente, este enfoque tiene algunos límites que vale la pena discutir.

El primer límite que podemos notar es que el volumen de almacenamiento no está aislado y, por lo tanto, cualquier otro contenedor puede acceder a su contenido. Para superar este problema, podemos usar la Seguridad Multicategoría (MCS) de SELinux con el sufijo `:Z` para aplicar categorías al volumen y hacerlo accesible exclusivamente para el contenedor en ejecución.

Sin embargo, dado que un segundo contenedor se ejecutaría de forma predeterminada con diferentes etiquetas de categoría, deberíamos tomar las categorías del volumen y ejecutar el segundo contenedor de etiquetado/envío con la opción `--security-opt label=level:s0:<CAT1>,<CAT2>`.

Alternativamente, podemos simplemente ejecutar los comandos de compilación, etiquetado y envío en un solo contenedor, como se muestra en el siguiente ejemplo:

```bash
$ podman run --device /dev/fuse \
    -v ~/build:/build \
    -v secure_storevol:/var/lib/containers:Z \
    quay.io/buildah/stable \
    sh -c 'buildah build -t test2 /build && \
    buildah tag test2 registry.example.com/build_test2 && \
    buildah login -u=<USERNAME> \
    -p=<PASSWORD> \
    registry.example.com && \
    buildah push registry.example.com/build_test2'
```

> [!IMPORTANT]
> En los ejemplos anteriores, utilizamos el inicio de sesión de Buildah pasando directamente el nombre de usuario y la contraseña en el comando. No hace falta decir que esto dista mucho de ser una práctica de seguridad aceptable.
> En lugar de pasar datos confidenciales en la línea de comandos, podemos montar el archivo de autenticación que contiene un token de sesión válido como un volumen dentro del contenedor.

El siguiente ejemplo monta un archivo `auth.json` válido, almacenado bajo el tmpfs `/run/user/<UID>`, dentro del contenedor de compilación, y luego se pasa la opción `--authfile /auth.json` al comando `buildah push`:

```bash
$ podman run --device /dev/fuse \
    -v ~/build:/build \
    -v /run/user/<UID>/containers/auth.json:/auth.json:z \
    -v secure_storevol:/var/lib/containers:Z \
    quay.io/buildah/stable \
    sh -c 'buildah build -t test3 /build && \
    buildah tag test3 registry.example.com/build_test3 && \
    buildah push --authfile /auth.json \
    registry.example.com/build_test3'
```

Finalmente, tenemos un ejemplo de trabajo que evita exponer credenciales en texto plano en los comandos pasados al contenedor.

Para proporcionar un archivo de autenticación funcional, debemos autenticarnos desde el host que ejecutará la compilación en contenedor o copiar un archivo de autenticación válido. Para autenticarnos con Podman, usaremos el siguiente comando:

```bash
$ podman login -u <USERNAME> -p <PASSWORD> <REGISTRY>
```

Si el proceso de autenticación tiene éxito, el token obtenido se almacena en el archivo `/run/user/<UID>/containers/auth.json`, que guarda un objeto codificado en JSON con una estructura similar al siguiente ejemplo:

```json
{
    "auths": {
        "registry.example.com": {
            "auth": "<base64_encoded_token>"
        }
    }
}
```

> [!CAUTION]
> ¡Alerta de seguridad! Si el archivo de autenticación montado dentro del contenedor tiene múltiples registros de autenticación para diferentes registros, estos quedarán expuestos dentro del contenedor de compilación. Esto puede generar posibles problemas de seguridad, ya que el contenedor podrá autenticarse en esos registros utilizando los tokens especificados en el archivo.

El enfoque basado en volúmenes que acabamos de describir tiene un pequeño impacto en el rendimiento en comparación con una compilación nativa en el host, pero proporciona un mejor aislamiento del proceso de compilación, una superficie de ataque reducida (gracias a la ejecución rootless) y la estandarización del entorno de compilación en diferentes hosts.

Inspeccionemos ahora cómo ejecutar compilaciones en contenedores utilizando almacenes montados por bind.

#### Ejecución de contenedores Buildah con almacenes montados por bind (*bind-mounted*)

En el escenario de mayor aislamiento, donde los equipos de DevOps siguen un enfoque de confianza cero (*zero-trust*), cada contenedor de compilación debe tener su propio almacén aislado poblado al comienzo de la compilación y destruido al finalizar. El aislamiento se puede lograr fácilmente con la seguridad MCS de SELinux.

Para probar este enfoque, comencemos creando un directorio temporal que alojará las capas de compilación. También queremos generar un sufijo aleatorio para un nombre a fin de alojar múltiples compilaciones sin conflictos:

```bash
# BUILD_STORE=/var/lib/containers-$(echo $RANDOM | md5sum | head -c 8)
# mkdir $BUILD_STORE
```

> [!NOTE]
> El ejemplo anterior y las siguientes compilaciones se ejecutan como root.

Ahora podemos ejecutar la compilación y montar por bind el nuevo directorio en la carpeta `/var/lib/containers` dentro del contenedor, además de agregar el sufijo `:Z` para garantizar el aislamiento de seguridad multicategoría:

```bash
# podman run --device /dev/fuse \
    -v ./build:/build:z \
    -v $BUILD_STORE:/var/lib/containers:Z \
    -v /run/containers/0/auth.json:/auth.json \
    quay.io/buildah/stable \
    sh -c 'set -euo pipefail; \
    buildah build -t registry.example.com/test4 /build; \
    buildah push --authfile /auth.json \
    registry.example.com/test4'
```

El aislamiento MCS garantiza el aislamiento de otros contenedores. Cada contenedor de compilación tendrá su propio almacén personalizado, y esto implica la necesidad de volver a descargar las capas de la imagen base en cada ejecución, ya que nunca se almacenan en caché.

A pesar de ser el más seguro en términos de aislamiento, este enfoque también ofrece el rendimiento más lento debido a las continuas descargas de la imagen base durante la ejecución de la compilación.

Por otro lado, el enfoque menos seguro no espera ningún aislamiento del almacén, y todos los contenedores de compilación montan el almacén predeterminado del host bajo `/var/lib/containers`. Este enfoque proporciona un mejor rendimiento, ya que permite la reutilización de capas en caché del almacén del host.

SELinux no permitirá que un proceso en contenedor acceda al almacén del host; por lo tanto, debemos relajar las restricciones de seguridad de SELinux para ejecutar el siguiente ejemplo utilizando la opción `--security-opt label=disable`.

El siguiente ejemplo ejecuta otra compilación utilizando el almacén predeterminado del host:

```bash
# podman run --device /dev/fuse \
    -v ./build:/build:z \
    -v /var/lib/containers:/var/lib/containers \
    --security-opt label=disable \
    -v /run/containers/0/auth.json:/auth.json \
    quay.io/buildah/stable \
    sh -c 'set -euo pipefail; \
    buildah build -t registry.example.com/test5 /build; \
    buildah push --authfile /auth.json \
    registry.example.com/test5'
```

El enfoque descrito en este ejemplo es el opuesto al anterior: mejor rendimiento pero peor aislamiento de seguridad.

Un buen compromiso entre ambos implica el uso de un almacén de imágenes secundario de solo lectura para proporcionar acceso a las capas en caché. Buildah admite el uso de múltiples almacenes de imágenes, y el archivo `/etc/containers/storage.conf` dentro de la imagen estable de Buildah ya configura la carpeta `/var/lib/shared` para este propósito.

Para demostrar esto, podemos inspeccionar el contenido del archivo `/etc/containers/storage.conf`, donde se define la siguiente sección:

```toml
# AdditionalImageStores is used to pass paths to additional Read/Only image stores
# Must be comma separated list.
additionalimagestores = [
    "/var/lib/shared",
]
```

De esta manera, podemos obtener un buen aislamiento y un mejor rendimiento, ya que las imágenes almacenadas en caché del host ya estarán disponibles en el almacén de solo lectura. El almacén de solo lectura se puede completar previamente con las imágenes más utilizadas para acelerar las compilaciones, o se puede montar desde un recurso compartido de red.

El siguiente ejemplo muestra este enfoque, montando por bind el almacén de solo lectura en el contenedor y ejecutando la compilación con la ventaja de reutilizar imágenes descargadas previamente:

```bash
# podman run --device /dev/fuse \
    -v ./build:/build:z \
    -v $BUILD_STORE:/var/lib/containers:Z \
    -v /var/lib/containers/storage:/var/lib/shared:ro \
    -v /run/containers/0/auth.json:/auth.json:z \
    quay.io/buildah/stable \
    bash -c 'set -euo pipefail; \
    buildah build -t registry.example.com/test6 /build; \
    buildah push --authfile /auth.json \
    registry.example.com/test6'
```

Los ejemplos mostrados en esta subsección también están inspirados en un gran artículo técnico escrito por Dan Walsh (uno de los líderes de los proyectos Buildah y Podman) en el blog de Red Hat Developer; consulta la sección *Lecturas adicionales* para obtener el enlace al artículo original. Cerremos esta sección con un ejemplo de comandos nativos de Buildah.

#### Ejecución de comandos nativos de Buildah dentro de contenedores

Hasta ahora hemos ilustrado ejemplos usando Dockerfiles/Containerfiles, pero nada nos impide ejecutar comandos nativos de Buildah en contenedores. El siguiente ejemplo crea una imagen de Python personalizada construida a partir de una imagen base de Fedora:

```bash
# BUILD_STORE=/var/lib/containers-$(echo $RANDOM | md5sum | head -c 8)
# mkdir $BUILD_STORE
# podman run --device /dev/fuse \
    -e REGISTRY=<USER_DEFINED_REGISTRY:PORT> \
    --security-opt label=disable \
    -v $BUILD_STORE:/var/lib/containers:Z \
    -v /var/lib/containers/storage:/var/lib/shared:ro \
    -v /run/containers/0:/run/containers/0 \
    quay.io/buildah/stable \
    sh -c 'set -euo pipefail; \
    container=$(buildah from fedora); \
    buildah run $container dnf install -y python3 python3; \
    buildah commit $container $REGISTRY/python_demo; \
    buildah push -authfile \
    /run/containers/0/auth.json $REGISTRY/python_demo'
```

Desde el punto de vista del rendimiento, así como del proceso de compilación, nada cambia con respecto a los ejemplos anteriores. Como ya se indicó, este enfoque proporciona más flexibilidad en las operaciones de compilación.

Si los comandos que se van a pasar son demasiados, una buena solución alternativa puede ser crear un script de shell e inyectarlo en la imagen de Buildah mediante un volumen dedicado:

```bash
# BUILD_STORE=/var/lib/containers-$(echo $RANDOM | md5sum | head -c 8)
# PATH_TO_SCRIPT=/path/to/script
# REGISTRY=<USER_DEFINED_REGISTRY:PORT>
# mkdir $BUILD_STORE
# podman run --device /dev/fuse \
    -v $BUILD_STORE:/var/lib/containers:Z \
    -v /var/lib/containers/storage:/var/lib/shared:ro \
    -v /run/containers/0:/run/containers/0 \
    -v $PATH_TO_SCRIPT:/root:z \
    quay.io/buildah/stable /root/build.sh
```

`build.sh` es el nombre del archivo de script de shell que contiene todos los comandos personalizados de compilación.

En esta sección, hemos aprendido a ejecutar Buildah en contenedores cubriendo tanto los montajes de volumen como los montajes bind. Hemos aprendido a ejecutar contenedores de compilación rootless que se pueden integrar fácilmente en canalizaciones o clústeres de Kubernetes para proporcionar un flujo de trabajo de ciclo de vida de aplicación de extremo a extremo. Esto se debe a la naturaleza flexible de Buildah, y por la misma razón, es muy fácil integrar Buildah dentro de compiladores personalizados, como veremos en la siguiente sección.

---

### Integración de Buildah con compiladores personalizados

Como vimos en la sección anterior de este capítulo, Buildah es un componente clave del ecosistema de contenedores de Podman. Buildah es una herramienta dinámica y flexible que se puede adaptar a diferentes escenarios para compilar contenedores completamente nuevos. Tiene varias opciones y configuraciones disponibles, pero nuestra exploración aún no ha terminado.

Podman y todos los proyectos desarrollados a su alrededor se han creado teniendo en cuenta la extensibilidad, haciendo que cada interfaz programable esté disponible para ser reutilizada desde el mundo exterior.

Podman, por ejemplo, hereda las capacidades de Buildah para compilar contenedores completamente nuevos a través del comando `podman build`; con el mismo principio, podemos integrar las interfaces de Buildah y su motor en nuestro propio compilador personalizado.

Veamos cómo construir un compilador personalizado en el lenguaje Go; veremos que el proceso es bastante sencillo, porque Podman, Buildah y muchos otros proyectos en este ecosistema están escritos realmente en el lenguaje Go.

#### Inclusión de Buildah en nuestra herramienta de compilación Go

Como primer paso, necesitamos preparar nuestro entorno de desarrollo, descargando e instalando todas las herramientas y bibliotecas necesarias para crear nuestra herramienta de compilación personalizada.

> [!IMPORTANT]
> Ten en cuenta que este es un escenario no compatible y no se garantiza la estabilidad en términos de obsolescencias y cambios de las API de Go de Buildah. Para casos de uso compatibles y estables, consulta las API REST de Podman.

En el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, vimos varios métodos de instalación de Podman. En la siguiente sección, utilizaremos un procedimiento similar mientras revisamos los pasos preliminares para compilar un proyecto de Buildah desde cero, descargando su archivo fuente para incluirlo en nuestro compilador personalizado.

En primer lugar, asegurémonos de tener todos los paquetes necesarios instalados en nuestro sistema host de desarrollo:

```bash
# dnf install -y golang git go-md2man btrfs-progs-devel gpgme-devel device-mapper-devel
Fedora 40 - x86_64                                  253 kB/s |  28 kB     00:00    
Fedora 40 openh264 (From Cisco) - x86_64            9.5 kB/s | 989  B     00:00    
Fedora 40 - x86_64 - Updates                        141 kB/s |  25 kB     00:00    
Fedora 40 - x86_64 - Updates                        3.9 MB/s | 6.2 MB     00:01    
Dependencies resolved.
==============================================================================================================================================================================================
 Package                                       Architecture            Version                       Repository               Size
==============================================================================================================================================================================================
Installing:
 btrfs-progs-devel                             x86_64                  6.11-1.fc40                   updates                  49 k
 device-mapper-devel                           x86_64                  1.02.199-1.fc40               updates                  41 k
 git                                           x86_64                  2.47.0-1.fc40                 updates                  52 k
 golang                                        x86_64                  1.22.7-1.fc40                 updates                 666 k
 golang-github-cpuguy83-md2man                 x86_64                  2.0.3-3.fc40                  fedora                  748 k
 gpgme-devel                                   x86_64                  1.23.2-3.fc40                 fedora                  167 k
 [... omitted output]
```

Después de instalar las bibliotecas centrales del lenguaje Go y algunas otras herramientas de desarrollo, estamos listos para crear la estructura de directorios para nuestro proyecto e inicializarlo:

```bash
$ mkdir ~/custombuilder
$ cd ~/custombuilder
[custombuilder]$ export GOPATH=`pwd`
```

Como se muestra en el ejemplo anterior, seguimos estos pasos:

1. Creamos el directorio raíz del proyecto.
2. Definimos la ruta raíz del lenguaje Go que vamos a utilizar.

Ahora estamos listos para crear nuestro módulo de Go que creará nuestra imagen de contenedor personalizada con unos sencillos pasos.

Para acelerar el ejemplo y evitar errores de escritura, podemos descargar el código del lenguaje Go que vamos a utilizar para esta prueba desde el repositorio oficial de GitHub de este libro:

Ve a [https://github.com/PacktPublishing/Podman-for-DevOps](https://github.com/PacktPublishing/Podman-for-DevOps) o ejecuta el siguiente comando:

```bash
$ git clone https://github.com/PacktPublishing/Podman-for-DevOps
```

Después de eso, copia los archivos proporcionados en el directorio `Chapter07/*` en el directorio `~/custombuilder/` recién creado. Deberías tener los siguientes archivos en tu directorio en este punto:

```bash
$ cd ~/custombuilder/src/builder
$ ls -la
total 148
drwxrwxr-x. 1 alex alex     74  9 nov 15.22 .
drwxrwxr-x. 1 alex alex     14  9 nov 14.10 ..
-rw-rw-r--. 1 alex alex   1466  9 nov 14.10 custombuilder.go
-rw-rw-r--. 1 alex alex    161  9 nov 15.22 go.mod
-rw-rw-r--. 1 alex alex 135471  9 nov 15.22 go.sum
-rw-rw-r--. 1 alex alex    337  9 nov 14.17 script.js
```

En este punto, podemos ejecutar el siguiente comando para que las herramientas de Go adquieran todas las dependencias necesarias para preparar el módulo para su ejecución:

```bash
$ go mod tidy -v
go: finding module for package github.com/containers/storage/pkg/unshare
go: finding module for package github.com/containers/image/v5/storage
[...omitted output...]
```

La herramienta analizó el archivo `custombuilder.go` proporcionado y encontró todas las bibliotecas requeridas, completando el archivo `go.mod`.

> [!IMPORTANT]
> Ten en cuenta que el comando anterior verificará si un módulo está disponible y, si no lo está, la herramienta comenzará a descargarlo de Internet. Por lo tanto, ¡sé paciente durante este paso!

Podemos verificar que los comandos anteriores descargaron todos los paquetes requeridos inspeccionando la estructura de directorios que creamos anteriormente:

```bash
$ cd ~/custombuilder
[custombuilder]$ ls
pkg  src
[custombuilder]$ ls -la pkg/
total 0
drwxrwxr-x. 1 alex alex  28  9 nov 18.27 .
drwxrwxr-x. 1 alex alex  12  9 nov 18.18 ..
drwxrwxr-x. 1 alex alex  20  9 nov 18.27 linux_amd64
drwxrwxr-x. 1 alex alex 196  9 nov 18.27 mod
[custombuilder]$ ls -la pkg/mod/
total 0
drwxrwxr-x. 1 alex alex 196  9 nov 18.27 .
drwxrwxr-x. 1 alex alex  28  9 nov 18.27 ..
drwxrwxr-x. 1 alex alex  22  9 nov 18.18 cache
drwxrwxr-x. 1 alex alex 918  9 nov 18.27 github.com
drwxrwxr-x. 1 alex alex  24  9 nov 18.27 go.etcd.io
drwxrwxr-x. 1 alex alex   2  9 nov 18.27 golang.org
[... omitted output]
[custombuilder]$ ls -la pkg/mod/github.com/
[... omitted output]
drwxrwxr-x. 1 alex alex  98  9 nov 18.27 containerd
drwxrwxr-x. 1 alex alex  20  9 nov 18.27 containernetworking
drwxrwxr-x. 1 alex alex 184  9 nov 18.27 containers
drwxrwxr-x. 1 alex alex 110  9 nov 18.27 coreos
[... omitted output]
```

Ahora estamos listos para ejecutar nuestro módulo de compilador personalizado, pero antes de avanzar, echemos un vistazo a los elementos clave contenidos en el archivo fuente de Go.

> [!IMPORTANT]
> Considera que en Fedora 40, al igual que para otras distribuciones de Linux, es posible que también necesites paquetes de desarrollo adicionales de los repositorios de tu distribución. Para Fedora, por ejemplo, para ejecutar con éxito el programa Go, es posible que también necesites instalar los paquetes `btrfs-progs-devel` y `libgpgme-devel`. Consulta la documentación de Podman para obtener más información:
> [https://podman.io/docs/installation#build-and-run-dependencies](https://podman.io/docs/installation#build-and-run-dependencies)

Si comenzamos a mirar el archivo `custombuilder.go`, justo después de definir el paquete y las bibliotecas a usar, definimos la función principal (`main`) de nuestro módulo.

En la función `main`, al principio de la definición, insertamos un bloque de código fundamental:

```go
if buildah.InitReexec() {
	return
}
unshare.MaybeReexecUsingUserNamespace(false)
```

Este fragmento de código habilita el uso del modo rootless aprovechando el paquete `unshare` de Go, disponible a través de `github.com/containers/storage/pkg/unshare`.

Para aprovechar las funciones de compilación de Buildah, tenemos que instanciar `buildah.Builder`. Este objeto tiene todos los métodos para definir los pasos de compilación, configurar la compilación y finalmente ejecutarla.

Para crear `Builder`, necesitamos un objeto llamado `storage.Store` del paquete `github.com/containers/storage`. Este elemento es responsable de almacenar las imágenes de contenedor intermedias y resultantes. Veamos el bloque de código que estamos discutiendo:

```go
buildStoreOptions, err := storage.DefaultStoreOptions()
buildStore, err := storage.GetStore(buildStoreOptions)
```

Como puedes ver en el ejemplo anterior, obtenemos las opciones predeterminadas y las pasamos al módulo de almacenamiento para solicitar un objeto `Store`.

Otro elemento que necesitamos para crear `Builder` es el objeto `BuilderOptions`. Este elemento contiene todas las opciones predeterminadas y personalizadas que podríamos asignar a `Builder` de Buildah. Veamos cómo definirlo:

```go
builderOpts := buildah.BuilderOptions{
	FromImage: "node:23-alpine", // Starting image
	Isolation: define.IsolationChroot, // Isolation environment
	CommonBuildOpts: &define.CommonBuildOptions{},
	ConfigureNetwork: define.NetworkDefault,
	SystemContext: &types.SystemContext{},
}
```

En el bloque de código anterior, definimos un objeto `BuilderOptions` que contiene lo siguiente:

- Una imagen inicial que vamos a utilizar para compilar nuestra imagen de contenedor de destino: en este caso, elegimos la imagen de Node.js basada en la distribución Alpine Linux. Esto se debe a que, en nuestro ejemplo, estamos simulando el proceso de compilación de una aplicación Node.js.
- Modo de aislamiento a adoptar una vez que se inicia la compilación: en este caso, vamos a utilizar el aislamiento `chroot`, que se adapta bien a muchos escenarios de compilación: menos aislamiento pero menos requisitos.
- Algunas opciones predeterminadas para los contextos de compilación, red y sistema: los objetos `SystemContext` definen la información contenida en los archivos de configuración como parámetros.

Ahora que tenemos todos los datos necesarios para instanciar `Builder`, hagámoslo:

```go
builder, err := buildah.NewBuilder(context.TODO(), buildStore, builderOpts)
```

Como puedes ver, estamos llamando a la función `NewBuilder`, con todas las opciones requeridas que creamos en el código anteriormente en esta sección, para tener `Builder` listo para crear nuestra imagen de contenedor personalizada.

Ahora estamos listos para instruir a `Builder` con las opciones requeridas para crear la imagen personalizada. Primero agreguemos a la imagen del contenedor el archivo JavaScript que contiene nuestra aplicación, para la cual estamos creando esta imagen de contenedor:

```go
err = builder.Add("/home/node/", false, buildah.AddAndCopyOptions{}, "script.js")
```

Asumimos que el archivo principal de JavaScript está almacenado junto al módulo de Go que estamos escribiendo y usando en este ejemplo, y estamos copiando este archivo en el directorio `/home/node/`, que es la ruta predeterminada donde la imagen del contenedor base espera encontrar este tipo de datos.

El programa JavaScript que vamos a copiar en la imagen del contenedor y usar para esta prueba es realmente simple; inspeccionémoslo:

```javascript
var http = require("http");

http.createServer(function(request, response) {
	response.writeHead(200, {"Content-Type": "text/plain"});
	response.write("Hello Podman and Buildah friends. This page is provided to you through a container running Node.js version: ");
	response.write(process.version);
	response.end();
}).listen(8080);
```

Sin profundizar en la sintaxis del lenguaje JavaScript y sus conceptos, podemos notar, al mirar el archivo JavaScript, que estamos usando la biblioteca HTTP para escuchar en el puerto 8080 las solicitudes entrantes, respondiendo a estas solicitudes con un mensaje de bienvenida predeterminado: *Hello Podman and Buildah friends. This page is provided to you through a container running Node.js version:* y anexando la versión de Node.js a la cadena de respuesta.

> [!NOTE]
> Ten en cuenta que JavaScript, también conocido como JS, es un lenguaje de programación de alto nivel que se compila justo a tiempo (*just-in-time*). Como dijimos anteriormente, no vamos a profundizar en la definición del lenguaje JavaScript ni en su entorno de ejecución más famoso, Node.js.

Después de eso, configuramos el comando predeterminado que se ejecutará para nuestra imagen de contenedor personalizada:

```go
builder.SetCmd([]string{"node", "/home/node/script.js"})
```

Simplemente configuramos el comando para ejecutar el entorno de ejecución de Node.js, haciendo referencia al programa JavaScript que acabamos de agregar a la imagen del contenedor.

Para confirmar (*commit*) los cambios que hicimos, necesitamos obtener la referencia de la imagen en la que estamos trabajando. Al mismo tiempo, también definiremos el nombre de la imagen del contenedor que creará `Builder`:

```go
imageRef, err := is.Transport.ParseStoreReference(buildStore, "podmanbook/nodejs-welcome")
```

Ahora, estamos listos para confirmar los cambios y llamar a la función `Commit` de `Builder`:

```go
imageId, _, _, err := builder.Commit(context.TODO(), imageRef, define.CommitOptions{})
fmt.Printf("Image built! %s\n", imageId)
```

Como podemos ver, simplemente le solicitamos a `Builder` que confirme los cambios, pasando la referencia de la imagen que obtuvimos anteriormente, y luego finalmente la imprimimos como referencia.

¡Ahora estamos listos para ejecutar nuestro programa! Ejecutémoslo:

```bash
[builder]$ go run custombuilder.go
Image built! e60fa98051522a51f4585e46829ad6a18df704dde774634dbc010baae4404849
```

Ahora podemos probar la imagen de contenedor personalizada que acabamos de compilar:

```bash
[builder]$ podman run -dt -p 8080:8080/tcp podmanbook/nodejs-welcome:latest
747805c1b59558a70c4a2f1a1d258913cae5ffc08cc026c74ad3ac21aab18974
[builder]$ curl localhost:8080
Hello Podman and Buildah friends. This page is provided to you through a container running Node.js version: v23.1.0
```

Como podemos ver en el bloque de código anterior, estamos ejecutando la imagen de contenedor que acabamos de crear con las siguientes opciones:

- `-d`: Modo desconectado (*detached*), que ejecuta el contenedor en segundo plano.
- `-t`: Asigna una nueva pseudo-TTY.
- `-p`: Publica el puerto del contenedor en el sistema host.
- `podmanbook/nodejs-welcome:latest`: El nombre de nuestra imagen de contenedor personalizada.

Finalmente, utilizamos la herramienta de línea de comandos `curl` para solicitar e imprimir la respuesta HTTP proporcionada por nuestro programa JavaScript, ¡que está contenerizado en la imagen de contenedor personalizada que creamos!

> [!NOTE]
> El ejemplo descrito en esta sección es solo una descripción general simple de todas las excelentes características que el módulo Buildah Go puede habilitar para nuestros compiladores de imágenes personalizados. Para obtener más información sobre las diversas funciones, variables y documentación del código, puedes consultar los documentos en [https://pkg.go.dev/github.com/containers/buildah](https://pkg.go.dev/github.com/containers/buildah).

Como vimos en esta sección, Buildah es una herramienta realmente flexible y, con sus bibliotecas, puede admitir compiladores personalizados en muchos escenarios diferentes.

Si intentamos buscar en Internet, podemos encontrar muchos ejemplos de Buildah que respaldan la creación de imágenes de contenedores personalizadas. Veamos algunos de ellos.

#### Ejecución de ejecutables nativos de Quarkus en contenedores

Quarkus se define como la pila de Java nativa de Kubernetes que aprovecha el proyecto OpenJDK (el kit de desarrollo de Java abierto) y el proyecto GraalVM. GraalVM es una máquina virtual Java que tiene muchas características especiales, como la compilación de aplicaciones Java para un inicio rápido y una huella de memoria baja.

> [!NOTE]
> No entraremos en los detalles de Quarkus, GraalVM y ningún otro proyecto complementario. El ejemplo en el que profundizaremos es solo para tu referencia. Te recomendamos que obtengas más información sobre estos proyectos visitando sus páginas web y leyendo la documentación relacionada.

Si echamos un vistazo a la página web de documentación de Quarkus, podemos descubrir fácilmente que, después de un largo tutorial en el que podemos aprender a compilar un ejecutable nativo de Quarkus, podemos empaquetar y ejecutar este ejecutable en una imagen de contenedor.

Los pasos proporcionados en la documentación de Quarkus aprovechan un envoltorio (*wrapper*) de Maven con una opción especial. Maven se creó como una herramienta de automatización de compilaciones de Java, pero luego también se extendió a otros lenguajes de programación. Si echamos un vistazo rápido a este comando, notaremos que se menciona a `podman`:

```bash
$ ./mvnw package -Pnative -Dquarkus.native.container-build=true -Dquarkus.native.container-runtime=podman
```

Esto significa que el programa contenedor de Maven invocará una compilación de Podman para crear una imagen de contenedor con el entorno preconfigurado distribuido por el proyecto Quarkus y la aplicación binaria que estamos desarrollando.

Se menciona a Podman porque, como vimos en el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), *Conoce Buildah – Construcción de Contenedores desde Cero*, toma prestada la lógica de compilación de Buildah al incluir sus bibliotecas.

Para explorar este ejemplo más a fondo, consulta [https://quarkus.io/guides/building-native-image](https://quarkus.io/guides/building-native-image).

---

### Resumen

En este capítulo, hemos aprendido a aprovechar el compañero de Podman, Buildah, en algunos escenarios avanzados para respaldar nuestros proyectos de desarrollo.

Vimos cómo usar Buildah para la creación de imágenes de contenedores multietapa, lo que nos permite crear compilaciones con múltiples etapas utilizando diferentes instrucciones `FROM` y, posteriormente, tener imágenes que tomen contenidos de las anteriores.

Luego, descubrimos que hay muchos casos de uso que implican la necesidad de compilaciones en contenedores. Hoy en día, uno de los escenarios de adopción más comunes es el flujo de trabajo de compilación de aplicaciones que se ejecuta sobre un clúster de Kubernetes. Por esta razón, profundizamos en los detalles de la contenerización de Buildah.

Finalmente, aprendimos, a través de muchos ejemplos interesantes, cómo integrar Buildah para crear compiladores personalizados para imágenes de contenedores. Como vimos en este capítulo, existen varias opciones y métodos para compilar realmente una imagen de contenedor con las herramientas del ecosistema de Podman. La mayoría de las veces, comenzamos desde una imagen base para personalizar y extender una capa previa del sistema operativo para que se ajuste a nuestros casos de uso.

En el próximo capítulo, aprenderemos más sobre las imágenes base de contenedores, cómo elegirlas y qué tener en cuenta al tomar nuestra decisión.

---

### Lecturas adicionales

- Una lista de proyectos de la CNCF: [https://landscape.cncf.io/](https://landscape.cncf.io/)
- Mejores prácticas para ejecutar Buildah en un contenedor: [https://developers.redhat.com/blog/2019/08/14/best-practices-for-running-buildah-in-a-container](https://developers.redhat.com/blog/2019/08/14/best-practices-for-running-buildah-in-a-container)
- Documentación del módulo Buildah Go: [https://pkg.go.dev/github.com/containers/buildah](https://pkg.go.dev/github.com/containers/buildah)
- Ejecutables nativos de Quarkus: [https://quarkus.io/guides/building-native-image](https://quarkus.io/guides/building-native-image)
