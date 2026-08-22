# Parte 2: Construyendo tu Contenedor desde Cero con Buildah

## Capítulo 10: Aseguramiento de Contenedores

La seguridad se está convirtiendo en el tema más candente de los tiempos actuales. Las empresas y organizaciones de todo el mundo están realizando enormes inversiones en prácticas y herramientas de seguridad que deberían ayudar a proteger sus sistemas de ataques internos o externos.

Como vimos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*, los contenedores y sus sistemas host pueden considerarse un medio para ejecutar y mantener en funcionamiento una aplicación de destino. La seguridad debe aplicarse a todos los niveles de la arquitectura del servicio, desde la infraestructura base hasta el código de la aplicación de destino, todo mientras pasa por la capa de virtualización o contenedorización.

En este capítulo, veremos las mejores prácticas y herramientas que podrían ayudar a mejorar la seguridad general de nuestra capa de contenedorización. En particular, vamos a cubrir los siguientes temas principales:

- Ejecución de contenedores rootless con Podman
- Evitar la ejecución de contenedores con UID 0
- Firma de nuestras imágenes de contenedores
- Personalización de las capacidades del kernel de Linux
- Comprensión de la interacción de SELinux con los contenedores

---

### Requisitos técnicos

Para completar los ejemplos de este capítulo, necesitarás una máquina con una instalación de Podman en funcionamiento. Como mencionamos en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos de este libro se han ejecutado en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Tener una buena comprensión de los temas tratados en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*, y el [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*, te ayudará a comprender mejor los temas de seguridad de contenedores discutidos en este capítulo.

---

### Ejecución de contenedores rootless con Podman

Como vimos brevemente en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, es posible que Podman permita a los usuarios estándar sin privilegios administrativos ejecutar contenedores en un host Linux. Estos contenedores a menudo se denominan contenedores sin raíz o *rootless containers*.

Los contenedores rootless tienen muchas ventajas, incluidas las siguientes:

- Crean una capa de seguridad adicional que podría bloquear a los atacantes que intentan obtener privilegios de root en el host, incluso si el motor de contenedores, el tiempo de ejecución o el orquestador se han visto comprometidos. Esto previene contra problemas de vulnerabilidad de escape de contenedores (*container escape*), potencialmente cualquier fallo que permita a un proceso en un contenedor escapar de las restricciones de seguridad impuestas. Si ocurre en un contenedor rootless, el proceso que ha escapado solo puede obtener privilegios de usuario sin privilegios (*non-root*).
- Pueden permitir que muchos usuarios sin privilegios ejecuten contenedores en el mismo host, aprovechando al máximo los entornos de computación de alto rendimiento.

Pensemos en el enfoque utilizado por cualquier sistema Linux para manejar los servicios de procesos tradicionales. Por lo general, los mantenedores de paquetes tienden a crear un usuario dedicado para programar y ejecutar el proceso de destino. Si intentamos instalar un servidor web Apache en nuestra distribución de Linux favorita a través del administrador de paquetes predeterminado, podemos descubrir que el servicio instalado se ejecutará a través de un usuario dedicado llamado `apache`.

Este enfoque ha sido la mejor práctica durante años porque, desde una perspectiva de seguridad, otorgar menos privilegios mejora la seguridad.

Utilizar el mismo enfoque pero con un contenedor rootless nos permite ejecutar el proceso del contenedor sin necesidad de una escalada de privilegios adicional. Además, Podman no tiene demonio (*daemonless*), por lo que simplemente creará un proceso hijo.

Ejecutar contenedores rootless en Podman es bastante sencillo y, como vimos en los capítulos anteriores, muchos de los ejemplos de este libro se pueden ejecutar como usuarios estándar sin privilegios. Ahora, aprendamos qué hay detrás de la ejecución de un contenedor rootless.

#### La navaja suiza de Podman: subuid y subgid

Las distribuciones modernas de Linux utilizan una versión del paquete `shadow-utils` que aprovecha dos archivos: `/etc/subuid` y `/etc/subgid`. Estos archivos se utilizan para determinar qué UID y GID se pueden utilizar para asignar un espacio de nombres de usuario (*user namespace*).

La asignación predeterminada para cada usuario es de 65536 UID y 65536 GID.

Podemos ejecutar el siguiente comando simple para verificar cómo funciona la asignación de subuid y subgid en contenedores rootless:

```bash
$ id
uid=1000(alex) gid=1000(alex) groups=1000(alex),10(wheel)
$ podman run alpine cat /proc/self/uid_map /proc/self/gid_map
Resolved "alpine" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/alpine:latest...
Getting image source signatures
Copying blob 59bf1c3509f3 done
Copying config c059bfaa84 done
Writing manifest to image destination
Storing signatures
         0       1000          1
         1     100000      65536
         0       1000          1
         1     100000      65536
```

Como podemos ver, ambos archivos indican que comienzan asignando el UID y GID 0 con el UID/GID del usuario actual con el que acabamos de ejecutar el contenedor; es decir, 1000. Después de eso, asigna el UID y GID 1, comenzando desde 100000 y llegando a 165536. Esto se calcula sumando el punto de partida, 100000, y el rango predeterminado, 65536.

Para una comparación, veamos el mismo ejemplo en el host:

```bash
$ id
uid=1000(alex) gid=1000(alex) groups=1000(alex),10(wheel)
$ cat /proc/self/uid_map /proc/self/gid_map
         0          0 4294967295
         0          0 4294967295
```

Debido a que comienza en 0 para ambos lados y cubre el rango completo de UID de 32 bits disponible en Linux, 4294967295, significa que el UID/GID 1000 en el sistema host es visto como UID/GID 1000 por el proceso. No hay ninguna traducción en curso.

El uso de contenedores rootless no es la única mejor práctica que podemos implementar para nuestros entornos de contenedores. En la siguiente sección, aprenderemos por qué no debemos ejecutar un contenedor con UID 0.

---

### Evitar la ejecución de contenedores con UID 0

Se puede indicar a los tiempos de ejecución de contenedores que ejecuten procesos dentro de un contenedor con un ID de usuario diferente al que creó inicialmente el contenedor. Ejecutar los procesos dentro del contenedor como un usuario sin privilegios de root puede ser útil para fines de seguridad. Si bien los entornos de ejecución de contenedores utilizan espacios de nombres de Linux para aislar procesos, una vulnerabilidad en el motor de contenedores o en el kernel podría permitir que un usuario root (UID 0 en el contenedor) escape del contenedor y obtenga los mismos privilegios que el usuario root del host (UID 0 en el host). Usar un usuario sin privilegios (con un UID distinto de 0) en un contenedor puede limitar la escalada de privilegios y, por lo tanto, la superficie de ataque dentro y fuera del contenedor.

De forma predeterminada, un Dockerfile y un Containerfile pueden establecer el usuario predeterminado como root (es decir, UID=0). Para evitar esto, podemos aprovechar la instrucción `USER` en esos archivos de compilación (por ejemplo, `USER 1001`) para indicarle a Buildah u otras herramientas de compilación de contenedores que compilen y ejecuten la imagen del contenedor utilizando ese usuario en particular (con UID 1001). Como se explicó anteriormente, una posible fuga del contenedor permitirá, en el peor de los casos, el acceso a los archivos y recursos del usuario del host con el mismo UID, 1001.

Si queremos forzar un UID específico, debemos ajustar los permisos de cualquier archivo, carpeta o montaje que planeemos usar con nuestros contenedores en ejecución.

Ahora, aprendamos a adaptar una imagen existente para que pueda ejecutarse con un usuario estándar.

Podemos aprovechar algunas imágenes preconstruidas en Docker Hub o elegir una de las imágenes de contenedor oficiales de nginx. Primero, necesitamos crear un archivo de configuración básico de nginx:

```bash
$ cat hello-podman.conf
server {
    listen 80;
    location / {
        default_type text/plain;
        expires -1;
        return 200 'Hello Podman user!\nServer address: $server_addr:$server_port\n';
    }
}
```

El archivo de configuración de nginx es realmente simple: definimos el puerto de escucha (80) y el mensaje de contenido que se devolverá una vez que llegue una solicitud al servidor.

Luego, podemos crear un Dockerfile simple para aprovechar una de las imágenes de contenedor oficiales de nginx:

```bash
$ cat Dockerfile
FROM docker.io/library/nginx:mainline-alpine
RUN rm /etc/nginx/conf.d/*
ADD hello-podman.conf /etc/nginx/conf.d/
```

El Dockerfile contiene tres instrucciones:

- `FROM`: Para seleccionar la imagen oficial de nginx.
- `RUN`: Para limpiar el directorio de configuración de cualquier ejemplo de configuración predeterminado.
- `ADD`: Para copiar el archivo de configuración que acabamos de crear.

Ahora, compilemos la imagen del contenedor con Buildah:

```bash
$ buildah bud -t nginx-root:latest -f .
STEP 1/3: FROM docker.io/library/nginx:mainline-alpine
STEP 2/3: RUN rm /etc/nginx/conf.d/*
STEP 3/3: ADD hello-podman.conf /etc/nginx/conf.d/
COMMIT nginx-root:latest
Getting image source signatures
Copying blob 8d3ac3489996 done
...
Copying config 21c5f7d8d7 done
Writing manifest to image destination
Storing signatures
--> 21c5f7d8d70
Successfully tagged localhost/nginx-root:latest
21c5f7d8d709e7cfdf764a14fd6e95fb4611b2cde52b57aa46d43262a6489f41
```

Una vez que hayas creado la imagen, nómbrala `nginx-root`. Ahora, estamos listos para ejecutar nuestro contenedor:

```bash
$ podman run --name myrootnginx -p 127.0.0.1::80 -d nginx-root
364ec7f5979a5059ba841715484b7238db3313c78c5c577629364aa46b6d9bdc
```

Aquí, usamos la opción `-p` para publicar el puerto y hacerlo accesible desde el host. Averigüemos qué puerto local se ha elegido, al azar, en el sistema host:

```bash
$ podman port myrootnginx 80
127.0.0.1:38029
```

Finalmente, llamemos a nuestro servidor web en contenedor:

```bash
$ curl localhost:38029
Hello Podman user!
Server address: 10.0.2.100:80
```

El contenedor finalmente se está ejecutando, pero ¿qué usuario está usando nuestro contenedor? Averigüémoslo:

```bash
$ podman ps | grep root
364ec7f5979a  localhost/nginx-root:latest  nginx -g daemon o...  55 minutes ago  Up 55 minutes ago  0.0.0.0:38029->80/tcp  myrootnginx
$ podman exec 364ec7f5979a id
uid=0(root) gid=0(root)
```

Como era de esperar, ¡el contenedor se ejecuta como root!

Ahora, hagamos algunas modificaciones para cambiar el usuario. Primero, necesitamos cambiar el puerto de escucha en la configuración del servidor nginx:

```bash
$ cat hello-podman.conf
server {
    listen 8080;
    location / {
        default_type text/plain;
        expires -1;
        return 200 'Hello Podman user!\nServer address: $server_addr:$server_port\n';
    }
}
```

Aquí, reemplazamos el puerto de escucha (80) por el 8080; no podemos usar un puerto que esté por debajo de 1024 con usuarios sin privilegios.

Luego, necesitamos editar nuestro Dockerfile:

```bash
$ cat Dockerfile
FROM docker.io/library/nginx:mainline-alpine
RUN rm /etc/nginx/conf.d/*
ADD hello-podman.conf /etc/nginx/conf.d/
RUN chmod -R a+w /var/cache/nginx/ \
    && touch /var/run/nginx.pid \
    && chmod a+w /var/run/nginx.pid
EXPOSE 8080
USER nginx
```

Como puedes ver, corregimos los permisos para el archivo y la carpeta principales en el servidor nginx, expusimos el nuevo puerto 8080 y establecimos el usuario predeterminado en `nginx`.

Ahora, estamos listos para compilar una imagen de contenedor nueva. Llamémosla `nginx-user`:

```bash
$ buildah bud -t nginx-user:latest -f .
STEP 1/6: FROM docker.io/library/nginx:mainline-alpine
STEP 2/6: RUN rm /etc/nginx/conf.d/*
STEP 3/6: ADD hello-podman.conf /etc/nginx/conf.d/
STEP 4/6: RUN chmod -R a+w /var/cache/nginx/ && touch /var/run/nginx.pid && chmod a+w /var/run/nginx.pid
STEP 5/6: EXPOSE 8080
STEP 6/6: USER nginx
COMMIT nginx-user:latest
Getting image source signatures
Copying blob 8d3ac3489996 done
...
Copying config 7628852470 done
Writing manifest to image destination
Storing signatures
--> 76288524704
Successfully tagged localhost/nginx-user:latest
762885247041fd233c7b66029020c4da8e1e254288e1443b356cbee4d73adf3e
```

Ahora, podemos ejecutar el contenedor:

```bash
$ podman run --name myusernginx -p 127.0.0.1::8080 -d nginx-user
299e0fb727f339d87dd7ea67eac419905b10e36181dc1ca7e35dc7d0a9316243
```

Encuentra el puerto de host aleatorio asociado y verifica si el servidor web está funcionando:

```bash
$ podman port myusernginx 8080
127.0.0.1:42209
$ curl 127.0.0.1:42209
Hello Podman user!
Server address: 10.0.2.100:8080
```

Finalmente, veamos si cambiamos el usuario que ejecuta el proceso de destino en nuestro contenedor:

```bash
$ podman ps | grep user
299e0fb727f3  localhost/nginx-user:latest  nginx -g daemon o...  38 minutes ago  Up 38 minutes ago  127.0.0.1:42209->8080/tcp  myusernginx
$ podman exec 299e0fb727f3 id
uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

Como puedes ver, nuestro contenedor se ejecuta como un usuario sin privilegios, que es lo que queríamos.

Si deseas consultar un ejemplo listo para usar de esto, visita el repositorio de GitHub de este libro: [https://github.com/PacktPublishing/Podman-for-DevOps](https://github.com/PacktPublishing/Podman-for-DevOps).

Desafortunadamente, la seguridad no se trata solo de permisos y usuarios: también debemos ocuparnos de la imagen base y su origen, y verificar las firmas de las imágenes de contenedores. Aprenderemos sobre esto en la siguiente sección.

---

### Firma de nuestras imágenes de contenedores

Cuando tratamos con imágenes que se han descargado de registros externos, tendremos algunas preocupaciones de seguridad relacionadas con las posibles tácticas de ataque que se han llevado a cabo en los contenedores (ver [1] en la sección *Lecturas adicionales*), especialmente las técnicas de enmascaramiento (*masquerading*), que ayudan al atacante a manipular los componentes de la imagen para que parezcan legítimos. Esto también podría suceder debido a un ataque de intermediario o *man-in-the-middle* (MITM) realizado por un atacante en la red.

Para evitar ciertos tipos de ataques mientras administras contenedores, la mejor solución es utilizar una firma de imagen separada (*detached signature*) para confiar en el proveedor de la imagen y garantizar su confiabilidad.

Cuando se descarga una imagen, Podman puede verificar la validez de las firmas y rechazar imágenes sin firmas válidas.

Ahora, aprendamos a implementar un flujo de trabajo de firma de imágenes básico.

#### Firma de imágenes con Sigstore y Podman

En esta sección, crearemos un par de claves Sigstore básico y configuraremos Podman para publicar y firmar la imagen mientras almacena la firma en un almacén provisional (*staging store*). Para mayor claridad, ejecutaremos un registro utilizando la imagen de contenedor básica de Docker Registry V2 sin ninguna personalización.

Antes de probar el flujo de trabajo de descarga de imágenes y validación de firmas, veremos cómo maneja Podman la firma como un artefacto OCI o mediante un servidor web independiente.

Para crear firmas de imágenes con Sigstore, necesitamos crear un par de claves válido o usar uno existente. Por esta razón, proporcionaremos un breve resumen sobre pares de claves asimétricas para ayudarte a comprender cómo funcionan las firmas de imágenes.

Un par de claves se compone de una clave privada y una clave pública. La clave pública se puede compartir universalmente, mientras que la clave privada se mantiene privada y nunca se comparte con nadie. La clave privada la utiliza el remitente (el creador de la imagen) para firmar la imagen. De esta manera, cualquiera con acceso a la clave pública correspondiente puede verificar que la imagen no fue manipulada y realmente se originó en el remitente.

Podemos traducir fácilmente este concepto a imágenes de contenedores: el propietario de la imagen que la publica en el registro remoto puede firmarla utilizando un par de claves y almacenar la firma separada en un almacén de firmas (*signature store*) que sea públicamente accesible para los usuarios. Aquí, la firma está separada de la imagen en sí: el registro almacenará los blobs de la imagen mientras que el almacén de firmas contendrá y expondrá las firmas de la imagen.

Los usuarios que descarguen la imagen podrán validar la firma de la imagen utilizando la clave pública compartida previamente.

Ahora, volvamos a la creación del par de claves Sigstore. Vamos a crear uno simple con la utilidad Cosign, una herramienta ligera diseñada específicamente para firmar y verificar imágenes de contenedores. Se adhiere a los estándares de la Open Container Initiative (OCI), lo que garantiza la compatibilidad y la integración perfecta con los registros de contenedores más populares.

Comencemos con la instalación y configuración de Cosign. Como de costumbre, hay varios métodos de instalación disponibles; aquí, informamos la forma más estructurada de instalarlo a través de un administrador de paquetes en tu sistema operativo:

```bash
$ LATEST_VERSION=$(curl https://api.github.com/repos/sigstore/cosign/releases/latest | grep tag_name | cut -d : -f2 | tr -d "v\", ")
$ curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-${LATEST_VERSION}-1.x86_64.rpm"
$ sudo rpm -ivh cosign-${LATEST_VERSION}-1.x86_64.rpm
```

Podemos verificar si la instalación fue exitosa ejecutando el siguiente comando:

```bash
$ cosign version
...
cosign: A tool for Container Signing, Verification and Storage in an OCI registry.
GitVersion:    v3.0.4
GitCommit:     6832fba4928c1ad69400235bbc41212de5006176
GitTreeState:  clean
BuildDate:     2026-01-09T21:17:16Z
GoVersion:     go1.25.5
Compiler:      gc
Platform:      linux/amd64
```

Ahora estamos listos para generar nuestro primer par de claves ejecutando el siguiente comando:

```bash
$ cosign generate-key-pair
```

El comando anterior te pedirá que proporciones una frase de contraseña (*passphrase*) para proteger tu clave privada. A diferencia de GPG, que administra claves en una base de datos del sistema oculta, Cosign simplemente genera dos archivos en tu directorio actual.

La salida del par de claves debe ser similar a la siguiente:

```bash
$ cosign generate-key-pair
Enter password for private key: 
Enter password for private key again: 
Private key written to cosign.key
Public key written to cosign.pub
```

Los archivos ya están en un formato estándar Privacy Enhanced Mail (PEM) y no es necesario realizar pasos de exportación adicionales. El archivo `cosign.pub` será útil más adelante, cuando definamos la política de verificación de firmas de imágenes. El archivo `cosign.key` contiene tu clave privada y debe mantenerse seguro, ya que se utiliza para firmar tus imágenes durante el proceso de publicación (*push*).

Una vez generado el par de claves, podemos crear un registro básico que alojará nuestras imágenes de contenedores. Para hacerlo, reutilizaremos el ejemplo básico del [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*, y ejecutaremos el siguiente comando como root:

```bash
$ mkdir ./registry_data
$ podman run -d \
  --name local_registry \
  -p 5000:5000 \
  -v ./registry_data:/var/lib/registry:z \
  --restart=always registry:2
```

Ahora tenemos un registro local sin autenticación que se puede utilizar para enviar las imágenes de prueba. Como mencionamos anteriormente, el registro no tiene conocimiento de la firma separada de la imagen.

Podman debe poder escribir firmas en un sigstore provisional. Ya existe una configuración predeterminada en el archivo `/etc/containers/registries.d/default.yaml`, que se ve de la siguiente manera:

```yaml
default-docker:
  # sigstore: file:///var/lib/containers/sigstore
  sigstore-staging: file:///var/lib/containers/sigstore
  use-sigstore-attachments: true
```

La ruta `sigstore-staging` es donde Podman escribe las firmas de las imágenes; esta ruta debe ser una carpeta con permisos de escritura, y es posible personalizarla o mantener la configuración predeterminada tal como está. La opción `use-sigstore-attachments: true` le otorga a Podman permisos para cargar sigstores como archivos sidecar compatibles con el formato de artefacto OCI.

Para los usuarios rootless, definir una ruta personalizada en el directorio de inicio permite a Podman escribir firmas correctamente sin requerir privilegios elevados.

Si queremos crear múltiples sigstores relacionados con usuarios, podemos crear el archivo `$HOME/.config/containers/registries.d/default.yaml` y definir una ruta `sigstore-staging` personalizada en el directorio de inicio del usuario, siguiendo la misma sintaxis que se mostró en el ejemplo anterior. Esto permitirá a los usuarios ejecutar Podman en modo rootless y escribir con éxito en su sigstore.

> [!IMPORTANT]
> En Podman 5.x, el método preferido es almacenar firmas directamente en el registro como artefactos OCI. Esto elimina la necesidad de administrar un directorio sigstore independiente o preocuparse por los permisos del sistema de archivos entre usuarios.

Dado que queremos demostrar la integración nativa entre Podman y Cosign, utilizaremos las claves que generamos para firmar la imagen durante el proceso de envío (*push*). A diferencia de los flujos de trabajo más antiguos basados en GPG, Podman 5.x permite a los usuarios rootless firmar y enviar imágenes sin problemas siempre que tengan acceso a su archivo `cosign.key`.

El siguiente ejemplo muestra el Dockerfile de una imagen httpd personalizada que se ha compilado utilizando UBI 9 (`Chapter11/image_signature/Dockerfile`):

```dockerfile
FROM registry.access.redhat.com/ubi9
# Update image and install httpd
RUN yum install -y httpd && yum clean all
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Para compilar la imagen, podemos ejecutar el siguiente comando:

```bash
$ cd Chapter11/image_signature
$ podman build -t custom_httpd .
```

Ahora, podemos etiquetar la imagen con el nombre del registro local:

```bash
$ podman tag custom_httpd localhost:5000/custom_httpd
```

Finalmente, es hora de enviar la imagen al registro temporal y firmarla utilizando el par de claves generado. La opción `--sign-by` permite a los usuarios pasar un par de claves válido identificado por el correo electrónico del usuario:

```bash
$ podman push \
  --tls-verify=false \
  --sign-by-sigstore-private-key ./cosign.key \
  localhost:5000/custom_httpd
Getting image source signatures
Copying blob 3ba8c926eef9 done
Copying blob a59107c02e1f done
Copying blob 352ba846236b done
Copying config 569b015109 done
Writing manifest to image destination
Creating signature: Signing image using a sigstore signature
Storing signatures
```

El código anterior envió con éxito los blobs de la imagen al registro y almacenó la firma de la imagen. Con eso, hemos enviado y firmado la imagen con éxito, haciéndola más segura para su uso futuro. Ahora, aprendamos a configurar Podman para recuperar imágenes firmadas.

#### Configuración de Podman para descargar imágenes firmadas

Para descargar con éxito una imagen firmada, Podman debe poder recuperar la firma de un almacén de firmas y tener acceso a una clave pública para verificarla.

En el flujo de trabajo moderno de Sigstore utilizado por Podman 5.x, el registro reconoce las firmas. Esto significa que cuando envías una imagen con una firma, Podman almacena esa firma directamente en el registro como un artefacto OCI (un objeto sidecar).

Debido a que la firma ahora se almacena junto a la imagen, ya no necesitamos mantener un servidor web externo o un directorio separado para alojar archivos de firma. Esto simplifica enormemente la canalización de DevOps, ya que el registro se convierte en la única fuente de información confiable tanto para las capas de la imagen como para su prueba criptográfica.

Ahora, configuremos Podman para la descarga de imágenes definiendo qué registros pueden usar esta función. A diferencia del modelo separado heredado, Podman 5.x puede recuperar firmas directamente del registro. Para asegurarnos de que Podman busque estas firmas integradas, editamos el archivo `/etc/containers/registries.d/default.yaml` para habilitar los archivos adjuntos de Sigstore:

```yaml
docker:
  localhost:5000:
    use-sigstore-attachments: true
```

Si bien estamos utilizando firmas integradas para nuestro registro local, aún puedes definir almacenes de firmas externos para otros registros. Por ejemplo, el siguiente código muestra cómo Podman localiza firmas para el registro público de Red Hat:

```yaml
docker:
  registry.access.redhat.com:
    sigstore: https://access.redhat.com/webassets/docker/content/sigstore
```

Antes de probar las descargas de imágenes, debemos implementar la clave pública que utiliza Podman para verificar las firmas. Esta clave pública (`cosign.pub`) debe almacenarse en el host que descarga la imagen y pertenece al par de claves que se utilizó para firmar la imagen.

El archivo de configuración que se utiliza para aplicar estas reglas de seguridad es `/etc/containers/policy.json`.

El siguiente código muestra una configuración personalizada que requiere una firma Sigstore válida para cualquier imagen descargada de `localhost:5000`:

```json
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker": {
      "localhost:5000": [
        {
          "type": "sigstoreSigned",
          "keyPath": "/etc/pki/containers/cosign.pub",
          "signedIdentity": {
            "type": "matchRepository"
          }
        }
      ]
    },
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
```

Para verificar las firmas de las imágenes que se han descargado de `localhost:5000`, podemos usar una clave pública que está almacenada en la ruta definida por el campo `keyPath`. La clave pública debe existir en la ruta definida y ser legible por Podman.

También es importante tener en cuenta la instrucción `insecureAcceptAnything` en la sección `default`. La clave predeterminada representa la política de respaldo global para Podman. Si intenta descargar una imagen y no encuentra una regla específica para ese registro (como la que creamos para `localhost:5000`), utiliza de forma predeterminada esta instrucción, que le indica a Podman que no realice ninguna verificación criptográfica. En un contexto de seguridad, esta línea es el equivalente a dejar la puerta principal abierta de par en par y colgar un cartel de bienvenida para todos.

La mayoría de las distribuciones de Linux se envían con esta configuración habilitada de forma predeterminada para garantizar una experiencia lista para usar sin problemas.

Ahora, estamos listos para probar la descarga de la imagen y verificar su firma. Asegúrate de que tu archivo `cosign.pub` esté en la ruta definida en tu política, luego ejecuta lo siguiente:

```bash
$ podman pull --tls-verify=false localhost:5000/custom_httpd
Getting image source signatures
Checking if image destination supports signatures
Copying blob 23fdb56daf15 skipped: already exists
Copying blob d4f13fad8263 skipped: already exists
Copying blob 96b0fdd0552f done
Copying config 569b015109 done
Writing manifest to image destination
Storing signatures
569b015109d457ae5fabb969fd0dc3cce10a3e6683ab60dc10505fc2d68e769f
```

La imagen se descargó con éxito en el almacén local después de la verificación de la firma utilizando la clave pública proporcionada. Si la firma hubiera faltado o hubiera sido manipulada, Podman habría rechazado la descarga de inmediato.

Ahora, veamos cómo se comporta Podman cuando no puede verificar correctamente la firma.

#### Prueba de fallos de verificación de firmas

Una política de seguridad es tan buena como su capacidad para bloquear contenido no confiable. Para verificar que nuestra política esté funcionando, debemos probar qué sucede cuando falla la verificación. Primero, eliminemos nuestra imagen local para asegurarnos de que Podman se vea obligado a descargarla del registro y realizar la comprobación:

```bash
$ podman rmi localhost:5000/custom_httpd
```

¿Qué sucede si existe una imagen en el registro pero no ha sido firmada? Podemos simular esto enviando la misma imagen con una etiqueta diferente sin usar el indicador `--sign-by` y luego intentando descargar la imagen nuevamente:

```bash
$ podman tag custom_httpd localhost:5000/custom_httpd:unsigned
$ podman push --tls-verify=false localhost:5000/custom_httpd:unsigned
$ podman pull --tls-verify=false localhost:5000/custom_httpd:unsigned
Error: unable to copy from source docker://localhost:5000/custom_httpd:unsigned: Source image rejected: A signature was required, but no signature exists
```

El error anterior demuestra que, dado que nuestro `policy.json` requiere una firma para cualquier imagen de `localhost:5000`, Podman la rechazará automáticamente.

Otro fallo común ocurre cuando la clave pública utilizada para la verificación no coincide con la clave privada utilizada para firmar. Para probar esto, podemos modificar nuestro archivo `/etc/containers/policy.json` para que apunte a una clave pública incorrecta. Por ejemplo, podríamos apuntar temporalmente `keyPath` a una clave pública de otro proyecto o generar una clave ficticia:

```bash
# Generate a dummy "wrong" key
$ cosign generate-key-pair --output-key-prefix dummy
```

Actualiza `/etc/containers/policy.json` para usar la clave pública `dummy.pub`, luego intenta descargar la imagen firmada original:

```bash
$ podman pull --tls-verify=false localhost:5000/custom_httpd
Error: Source image rejected: Invalid signature: key mismatch
```

Estos errores confirman que Podman está aplicando activamente la relación de confianza. En un entorno de producción, esto evita ataques de intermediario o el despliegue accidental de imágenes no examinadas.

> [!IMPORTANT]
> No olvides restaurar la clave pública válida en el archivo `/etc/containers/policy.json` antes de continuar con los siguientes ejemplos.

Podman ofrece un control aún más granular sobre estas políticas y también ofrece comandos CLI dedicados para ayudarte a personalizar las políticas de seguridad, que veremos en la siguiente subsección.

#### Administración de claves con comandos de confianza de imagen de Podman

Es posible editar el archivo `/etc/containers/policy.json` y modificar sus objetos JSON para agregar o eliminar configuraciones para registros dedicados. Sin embargo, la edición manual puede ser propensa a errores y difícil de automatizar en un entorno DevOps de rápido movimiento.

Alternativamente, podemos usar el comando `podman image trust` para volcar o modificar la configuración actual.

El siguiente código muestra cómo imprimir la configuración actual con el comando `podman image trust show`:

```bash
$ podman image trust show
TRANSPORT   NAME               TYPE            ID    STORE
all         default            accept                
repository  localhost:5000     sigstoreSigned  N/A   
docker-daemon                  accept                
```

También es posible configurar nuevas confianzas. Por ejemplo, podemos agregar la clave GPG pública de Red Hat para verificar la firma de las imágenes UBI.

Primero, necesitamos descargar la clave pública de Red Hat:

```bash
$ sudo wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat \
  https://www.redhat.com/security/data/fd431d51.txt
```

> [!NOTE]
> Las claves de firma de productos de Red Hat, incluida la que se utilizó en este ejemplo, se pueden encontrar en [https://access.redhat.com/security/team/key](https://access.redhat.com/security/team/key).

Después de descargar la clave, debemos configurar la confianza de imagen para las imágenes UBI 9 que se han descargado de `registry.access.redhat.com` usando el comando `podman image trust set`:

```bash
$ sudo podman image trust set -f /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat registry.access.redhat.com/ubi9
```

Después de ejecutar el comando anterior, el archivo `/etc/containers/policy.json` cambiará de la siguiente manera:

```json
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker": {
      "localhost:5000": [
        {
          "type": "sigstoreSigned",
          "keyPath": "/etc/pki/containers/cosign.pub",
          "signedIdentity": {
            "type": "matchRepository"
          }
        }
      ],
      "registry.access.redhat.com/ubi9": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat"
        }
      ]
    },
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
```

Ten en cuenta que la entrada relacionada con `registry.access.redhat.com/ubi9` y la clave pública que se utilizó para verificar las firmas de las imágenes se han agregado al archivo.

La configuración de sigstore de Red Hat para este registro ya está disponible en el archivo `/etc/containers/registries.d/registry.access.redhat.com.yaml`, instalado con el paquete Podman:

```yaml
docker:
  registry.access.redhat.com:
    sigstore: https://access.redhat.com/webassets/docker/content/sigstore
```

> [!TIP]
> Los archivos de registro relacionados con Red Hat se han incluido en el repositorio `github.com/containers/common` para permitir una integración más sencilla con el ecosistema de Red Hat.

Es posible crear archivos de configuración de registro personalizados para diferentes proveedores en la carpeta `/etc/containers/registries.d`. Por ejemplo, el ejemplo anterior podría definirse en un archivo `/etc/containers/registries.d/redhat.yaml` dedicado. Esto te permite mantener y versionar fácilmente las configuraciones de sigstore del registro.

A partir de ahora, cada vez que se descargue una imagen UBI9 de `registry.access.redhat.com`, su firma se descargará del almacén de firmas de Red Hat y se validará utilizando la clave pública proporcionada.

Hasta ahora, hemos visto ejemplos de administración de claves en Podman, pero también es posible administrar la verificación de firmas con Skopeo. En la siguiente subsección, veremos algunos ejemplos básicos.

#### Administración de firmas con Skopeo

Podemos verificar la firma de una imagen usando Skopeo cuando descargamos una imagen desde un transporte válido.

El siguiente ejemplo utiliza el comando `skopeo copy` para descargar la imagen de nuestro registro al almacén local. Este comando tiene los mismos efectos que usar un comando `podman pull`, pero permite un mayor control sobre los transportes de origen y destino:

```bash
$ skopeo copy --src-tls-verify=false \
  docker://localhost:5000/custom_httpd \
  containers-storage:localhost:5000/custom_httpd
```

Skopeo no necesita ninguna configuración adicional porque hace referencia automáticamente a la ruta de la clave pública definida en `/etc/containers/policy.json`. Si la firma en el registro no coincide con nuestra clave pública, Skopeo se negará a copiar la imagen.

También podemos usar Skopeo para firmar una imagen con nuestra clave privada antes de copiarla a un transporte:

```bash
$ skopeo copy \
  --dest-tls-verify=false \
  --sign-by-sigstore-private-key ./cosign.key \
  containers-storage:localhost:5000/custom_httpd \
  docker://localhost:5000/custom_httpd
```

En esta sección, aprendimos a verificar firmas de imágenes y evitar posibles ataques MITM. En la siguiente sección, veremos cómo simplificar el proceso de creación de claves con la ayuda de Skopeo.

#### Generación de una clave Sigstore con Skopeo

Skopeo proporciona una opción útil que simplifica la creación de claves para firmar imágenes de contenedores.

En primer lugar, creemos un directorio para guardar las claves pública y privada que vamos a generar:

```bash
$ mkdir .skopeo
```

Ahora estamos listos para aprovechar Skopeo para el proceso de creación de claves:

```bash
$ skopeo generate-sigstore-key --output-prefix .skopeo/sig-ubi8-httpd
Passphrase for key .skopeo/sig-ubi8-httpd.private: 
Key written to ".skopeo/sig-ubi8-httpd.private" and ".skopeo/sig-ubi8-httpd.pub"
```

El comando también solicitará una frase de contraseña en caso de que queramos aumentar el nivel de seguridad de la clave. Una vez que generamos un nuevo par de claves privada y pública, podemos seguir las secciones anteriores para firmar las imágenes de contenedores y confiar en el contenido descargado por los registros de contenedores.

Acabamos de aprender los conceptos básicos de la firma y gestión de imágenes de contenedores. En la siguiente sección, veremos cómo la comunidad de código abierto intentó ayudar en este proceso mediante la creación de un conjunto de herramientas para la gestión de firmas de imágenes.

#### Uso de Rekor y Cosign para administrar firmas de imágenes

Esta sección explora cómo aprovechar Cosign y Rekor para establecer un sistema robusto y confiable para administrar las firmas de imágenes de contenedores, reforzando significativamente la seguridad de tu cadena de suministro de software.

Antes de sumergirnos en la implementación, aclaremos las funciones de estos dos actores clave:

- **Cosign**: Una herramienta liviana diseñada específicamente para firmar y verificar imágenes de contenedores. Se adhiere a los estándares de la Open Container Initiative (OCI), lo que garantiza la compatibilidad y la integración perfecta con los registros de contenedores más populares.
- **Rekor**: Un servidor de registro de transparencia (*transparency log server*) que proporciona un registro inmutable de todos los metadatos generados durante el proceso de firma. Esta inmutabilidad garantiza la integridad de las firmas, evitando cualquier manipulación o revocación retroactiva.

Ya tenemos nuestro binario `cosign` instalado en el sistema. Para este ejemplo, no vamos a desplegar un servidor Rekor, aunque la página del proyecto explica en detalle todos los requisitos previos necesarios. En su lugar, vamos a aprovechar el servidor público Rekor ofrecido por la comunidad de código abierto, disponible en [https://rekor.sigstore.dev](https://rekor.sigstore.dev/).

Ahora estamos listos para probar tanto Cosign como Rekor para firmar nuestra primera imagen de contenedor con la ayuda de estas herramientas.

Vamos a firmar la imagen de contenedor que creamos anteriormente, `custom_httpd`, a través de la clave `sig-ubi8-httpd` creada mediante Skopeo. Ejecutamos el proceso de firma mediante el siguiente comando:

```bash
$ cosign sign --key .cosign.key quay.io/alezzandro/ubi9-httpd@sha256:cda838d6ef3ddf951c1c7f1086d1aa1ed895159a4e794b8629da66b35b94c83f
Enter password for private key:
...
tlog entry created with index: 165166231
Pushing signature to: quay.io/alezzandro/ubi9-httpd
```

Como se muestra en el comando anterior, la mejor práctica es firmar una imagen de contenedor en función de su resumen (*digest*). Esto evita cualquier desajuste si sobrescribimos una etiqueta de imagen con una versión más nueva.

> [!NOTE]
> El comando anterior requiere que la imagen del contenedor resida en un registro de contenedores que también sea capaz de manejar las firmas de las imágenes. Debes estar autenticado contra el registro de imágenes del contenedor y debes tener los permisos correctos para ejecutar estas acciones.

Entre bastidores, Cosign genera una firma criptográfica para la imagen de tu contenedor y crea una entrada en el registro del servidor público de Rekor, registrando la firma junto con metadatos cruciales como el resumen de la imagen, la clave pública utilizada para firmar y la marca de tiempo.

Inspeccionemos la firma de la imagen y su registro almacenado en el servidor público de Rekor.

Primero, vamos a descargar una de las últimas interfaces de línea de comandos de Rekor desde el repositorio público de GitHub:

```bash
$ curl -O -L https://github.com/sigstore/rekor/releases/download/v1.3.8/rekor-cli-linux-amd64
$ chmod +x rekor-cli-linux-amd64
```

Ahora estamos listos para consultar el servidor público de Rekor con el siguiente comando:

```bash
$ ./rekor-cli-linux-amd64 get --log-index 165166231
LogID: c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d
Index: 165166231
IntegratedTime: 2025-01-24T11:20:22Z
UUID: 108e9186e8c5677af3fa03b736ca86f4605e0229fb620863ff8d0b550db9e43f0aad3b2da8c0df4f
Body: {
... omitted output ...
}
}
}
```

Finalmente, podemos verificar la firma de la imagen con la herramienta de línea de comandos Cosign:

```bash
$ cosign verify --key ./cosign.pub quay.io/alezzandro/ubi9-httpd@sha256:cda838d6ef3ddf951c1c7f1086d1aa1ed895159a4e794b8629da66b35b94c83f
Verification for quay.io/alezzandro/ubi9-httpd@sha256:cda838d6ef3ddf951c1c7f1086d1aa1ed895159a4e794b8629da66b35b94c83f --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
[{"critical":{"identity":{"docker-reference":"quay.io/alezzandro/ubi9-httpd"},"image":{"docker-manifest-digest":"sha256:cda838d6ef3ddf951c1c7f1086d1aa1ed895159a4e794b8629da66b35b94c83f"},"type":"cosign container image signature"},
... omitted output
```

Como se mostró anteriormente, el registro de Rekor proporciona un registro de auditoría innegable, evitando que cualquiera niegue su participación en la firma de una imagen. Cualquiera puede auditar de forma independiente el registro de Rekor para verificar la autenticidad y la integridad de las firmas de las imágenes. Rekor agrega una capa crítica de seguridad, lo que dificulta significativamente que los atacantes comprometan el proceso de firma de imágenes.

Para maximizar la eficiencia y la seguridad, podemos integrar Cosign y Rekor en nuestras canalizaciones de CI/CD. Esta automatización garantiza que todas las imágenes se firmen y verifiquen como parte de tu flujo de trabajo de desarrollo.

Finalmente, al combinar Cosign y Rekor, puedes establecer un sistema robusto y transparente para administrar las firmas de imágenes de contenedores, fortaleciendo significativamente tu cadena de suministro de software contra posibles amenazas.

En la siguiente sección, cambiaremos de enfoque y aprenderemos cómo ejecutar el tiempo de ejecución del contenedor personalizando las capacidades del kernel de Linux.

---

### Personalización de las capacidades del kernel de Linux

Las capacidades (*capabilities*) son características que se introdujeron en el kernel de Linux 2.2 con el propósito de dividir los privilegios elevados en unidades individuales que se pueden asignar arbitrariamente a un proceso o subproceso (*thread*).

En lugar de ejecutar un proceso como una instancia con privilegios totales con UID efectivo 0, podemos asignar un subconjunto limitado de capacidades específicas a un proceso sin privilegios. Al proporcionar un control más granular sobre el contexto de seguridad de la ejecución del proceso, este enfoque ayuda a mitigar las posibles tácticas de ataque.

Antes de hablar sobre las capacidades de los contenedores, recapitulemos cómo funcionan en un sistema Linux para que comprendamos su lógica interna.

#### Comprensión de las capacidades del kernel de Linux

Las capacidades se asocian con los ejecutables de archivos mediante atributos extendidos (ver `man xattr`) y se otorgan al proceso resultante tras una llamada al sistema `execve()`, sujeto a las reglas de transición de capacidades del kernel.

La lista de capacidades disponibles es bastante amplia y sigue creciendo; incluye acciones muy específicas que puede realizar un hilo. Algunos ejemplos básicos son los siguientes:

- **CAP_CHOWN**: Esta capacidad permite que un hilo modifique el UID y el GID de un archivo.
- **CAP_KILL**: Esta capacidad te permite omitir las comprobaciones de permisos para enviar una señal a un proceso.
- **CAP_MKNOD**: Esta capacidad te permite crear un archivo especial con la llamada al sistema `mknod()`.
- **CAP_NET_ADMIN**: Esta capacidad te permite operar varias acciones privilegiadas en la configuración de red del sistema, incluido cambiar la configuración de la interfaz, habilitar/deshabilitar el modo promiscuo para una interfaz, editar tablas de enrutamiento y habilitar/deshabilitar la multidifusión (*multicasting*).
- **CAP_NET_RAW**: Esta capacidad permite que un hilo use sockets RAW y PACKET. Puede ser utilizado por programas como `ping` para enviar paquetes ICMP sin la necesidad de privilegios elevados.
- **CAP_SYS_CHROOT**: Esta capacidad te permite usar la llamada al sistema `chroot()` y cambiar los espacios de nombres de montaje con la llamada al sistema `setns()`.
- **CAP_SYS_ADMIN**: Esta capacidad otorga privilegios de root casi totales, lo que permite a los procesos montar sistemas de archivos y eludir el aislamiento crítico del contenedor. Otorgar esta capacidad permite a los contenedores eludir capas de aislamiento vitales, lo que efectivamente le da a un proceso el control total sobre el kernel subyacente del host.
- **CAP_DAC_OVERRIDE**: Esta capacidad te permite omitir las comprobaciones de control de acceso discrecional (DAC) para la lectura, escritura y ejecución de archivos.
- **CAP_BPF**: Introducida en el kernel 5.8, esta capacidad emplea operaciones BPF privilegiadas.

Para obtener más detalles y una lista extensa de las capacidades disponibles, consulta la página de manual correspondiente (`man capabilities`).

Para asignar una capacidad a un ejecutable, podemos usar el comando `setcap`, como se muestra en el siguiente ejemplo, donde `CAP_NET_ADMIN` y `CAP_NET_RAW` se permiten en el ejecutable `/usr/bin/ping`:

```bash
$ sudo setcap 'cap_net_admin,cap_net_raw+p' /usr/bin/ping
```

El indicador `+p` en el comando anterior indica que las capacidades se han establecido en Permitidas (*Permitted*).

Para inspeccionar las capacidades de un archivo, podemos usar el comando `getcap`:

```bash
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_admin,cap_net_raw=p
```

Consulta `man getcap` y `man setcap` para obtener más detalles sobre estas utilidades.

Podemos inspeccionar las capacidades activas de un proceso en ejecución mirando el archivo `/proc/<PID>/status`. En el siguiente código, lanzamos un comando ping después de configurar las capacidades `CAP_NET_ADMIN` y `CAP_NET_RAW`. Queremos iniciar el proceso en segundo plano y verificar sus capacidades actuales:

```bash
$ ping example.com > /dev/null 2>&1 &
$ grep 'Cap.*' /proc/$(pgrep ping)/status
CapInh: 0000000000000000
CapPrm: 0000000000003000
CapEff: 0000000000000000
CapBnd: 000000ffffffffff
CapAmb: 0000000000000000
```

Aquí, nos interesa evaluar el mapa de bits en el campo `CapPrm`, que representa las capacidades permitidas. Para obtener un valor fácil de usar, podemos usar el comando `capsh` para decodificar el valor hexadecimal del mapa de bits:

```bash
$ capsh --decode=0000000000003000
0x0000000000003000=cap_net_admin,cap_net_raw
```

El resultado es similar a la salida del comando `getcap` en el archivo `/usr/bin/ping`, lo que demuestra que la ejecución del comando propagó las capacidades permitidas del archivo a su instancia de proceso.

Para obtener una lista completa de las constantes que se utilizaron para establecer los mapas de bits, así como sus capacidades, consulta el siguiente archivo de encabezado del kernel: [https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h).

> [!TIP]
> Las versiones anteriores en distribuciones como RHEL y CentOS utilizaban la configuración anterior para permitir que ping enviara paquetes ICMP con acceso de todos los usuarios sin que se ejecutaran como procesos privilegiados con setuid 0. Este era un enfoque inseguro en el que un atacante podía aprovechar una vulnerabilidad o error en el ejecutable para escalar privilegios y obtener el control del sistema.
>
> Fedora introdujo un enfoque nuevo y más seguro a partir de la versión 31 que se basa en el uso del parámetro del kernel de Linux `net.ipv4.ping_group_range`. Al establecer un rango extenso que cubre todos los grupos del sistema, este parámetro permite a los usuarios enviar paquetes ICMP sin la necesidad de habilitar las capacidades `CAP_NET_ADMIN` y `CAP_NET_RAW`. Este enfoque fue heredado por RHEL y muchas otras distribuciones similares y es el nuevo estándar más seguro.
>
> Para obtener más detalles, consulta la siguiente página wiki del Proyecto Fedora: [https://fedoraproject.org/wiki/Changes/EnableSysctlPingGroupRange](https://fedoraproject.org/wiki/Changes/EnableSysctlPingGroupRange).

Ahora que hemos proporcionado una descripción de alto nivel de las capacidades del kernel de Linux, aprendamos cómo se aplican a los contenedores.

#### Uso de capacidades en contenedores

Las capacidades se pueden aplicar dentro de los contenedores para permitir que se realicen acciones específicas. De forma predeterminada, Podman ejecuta contenedores utilizando un conjunto de capacidades del kernel de Linux que se definen en el archivo `/usr/share/containers/containers.conf`. Al momento de escribir este artículo, las siguientes capacidades están habilitadas y declaradas dentro de este archivo (con comentarios):

```text
#default_capabilities = [
#  "CHOWN",
#  "DAC_OVERRIDE",
#  "FOWNER",
#  "FSETID",
#  "KILL",
#  "NET_BIND_SERVICE",
#  "SETFCAP",
#  "SETGID",
#  "SETPCAP",
#  "SETUID",
#  "SYS_CHROOT"
#]
```

Podemos ejecutar una prueba simple para verificar que esas capacidades se hayan aplicado efectivamente a un proceso que se ejecuta dentro de un contenedor. Para esta prueba, usaremos la imagen oficial de nginx:

```bash
$ podman run -d --name cap_test docker.io/library/nginx
$ podman exec -it cap_test sh -c 'grep Cap /proc/1/status'
CapInh: 0000000000000000
CapPrm: 00000000800405fb
CapEff: 00000000800405fb
CapBnd: 00000000800405fb
CapAmb: 0000000000000000
```

Esto es lo que representa cada línea para el proceso:

- **CapInh (heredable / *inheritable*)**: Son capacidades que se pueden conservar a través de un `execve`. En este caso, son todos ceros, lo que significa que un proceso hijo no heredará automáticamente ninguna capacidad a menos que se configure específicamente.
- **CapPrm (permitido / *permitted*)**: Este es el conjunto limitante. Define las capacidades máximas que el proceso puede utilizar realmente. El proceso puede mover capacidades de este conjunto al conjunto efectivo.
- **CapEff (efectivo / *effective*)**: Este es el conjunto que el kernel realmente verifica. Por ejemplo, si un proceso intenta cambiar el reloj del sistema, el kernel busca el bit para `CAP_SYS_TIME` en este conjunto específico. Dado que coincide con `CapPrm`, el proceso está actualmente activo con todos sus privilegios permitidos.
- **CapBnd (delimitador / *bounding*)**: Es un mecanismo utilizado para limitar las capacidades que un proceso puede llegar a obtener. No puedes promover una capacidad a tu conjunto permitido si no está ya en tu conjunto delimitador.
- **CapAmb (ambiental / *ambient*)**: Un conjunto más nuevo (agregado en Linux 4.3) que resuelve el problema de localizar capacidades para usuarios no root. Permite conservar las capacidades al ejecutar un programa que no es setuid.

Aquí hemos extraído las capacidades actuales del proceso padre nginx (que se ejecuta con el PID 1 dentro del contenedor). Ahora, podemos verificar el mapa de bits con la utilidad `capsh`:

```bash
$ capsh --decode=00000000800405fb
0x00000000800405fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
```

La lista anterior de capacidades es la misma que la lista que se definió en la configuración predeterminada de Podman. Ten en cuenta que las capacidades se aplican tanto en modo rootless como en modo rootful.

> [!NOTE]
> Si tienes curiosidad, las capacidades para los procesos en contenedores son configuradas por el tiempo de ejecución del contenedor, que es `runc` o `crun`, según la distribución.

Ahora que sabemos cómo se configuran y aplican las capacidades dentro de los contenedores, aprendamos a personalizar las capacidades de un contenedor.

#### Personalización de las capacidades de un contenedor

Podemos agregar o quitar capacidades ya sea en tiempo de ejecución o estáticamente.

Para cambiar estáticamente las capacidades predeterminadas, simplemente podemos editar el campo `default_capabilities` en el archivo `/usr/share/containers/containers.conf` y agregarlas o eliminarlas según los resultados deseados.

Para modificar las capacidades en tiempo de ejecución, podemos usar las opciones `--cap-add` y `--cap-drop`, las cuales son proporcionadas por el comando `podman run`.

El siguiente código elimina la capacidad `CAP_DAC_OVERRIDE` de un contenedor:

```bash
$ podman run -d --name cap_test2 --cap-drop=DAC_OVERRIDE docker.io/library/nginx
```

Si miramos los mapas de bits de capacidad nuevamente, veremos que se actualizaron en consecuencia:

```bash
$ podman exec cap_test2 sh -c 'grep Cap /proc/1/status'
CapInh: 0000000000000000
CapPrm: 00000000800405f9
CapEff: 00000000800405f9
CapBnd: 00000000800405f9
CapAmb: 0000000000000000
$ capsh --decode=00000000800405f9
0x00000000800405f9=cap_chown,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap
```

Es posible pasar las opciones `--cap-add` y `--cap-drop` varias veces:

```bash
$ podman run -d --name cap_test3 \
  --cap-drop=KILL \
  --cap-drop=DAC_OVERRIDE \
  --cap-add=NET_RAW \
  --cap-add=NET_ADMIN \
  docker.io/library/nginx
```

Cuando tratamos con capacidades, debemos tener cuidado al eliminar una capacidad predeterminada. El siguiente código muestra un error en el contenedor nginx al eliminar la capacidad `CAP_CHOWN`:

```bash
$ podman run --name cap_test4 \
  --cap-drop=CHOWN \
  docker.io/library/nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/01/06 23:19:39 [emerg] 1#1: chown("/var/cache/nginx/client_temp", 101) failed (1: Operation not permitted)
nginx: [emerg] chown("/var/cache/nginx/client_temp", 101) failed (1: Operation not permitted)
```

Aquí, el contenedor falla. En la salida, podemos ver que el proceso nginx no pudo mostrar el directorio `/var/cache/nginx/client_temp`. Esta es una consecuencia directa de la eliminación de la capacidad `CAP_CHOWN`.

Las capacidades se pueden aplicar a un contenedor rootless, pero son capacidades con espacio de nombres (*namespaced capabilities*), que son más limitadas que las normales y no pueden realizar acciones que alterarían el sistema host. Por ejemplo, si intentamos aplicar la capacidad `CAP_MKNOD` a un contenedor rootless, el kernel no permitirá ningún intento de crear un archivo especial dentro de un contenedor rootless:

```bash
$ podman run -it --cap-add=MKNOD \
  docker.io/library/busybox /bin/sh
/ # mkdir -p /test/dev
/ # mknod -m 666 /test/dev/urandom c 1 8
mknod: /test/dev/urandom: Operation not permitted
```

En cambio, si ejecutamos el contenedor con privilegios de root elevados, la capacidad se puede asignar con éxito:

```bash
# podman run -it --cap-add=MKNOD \
  docker.io/library/busybox /bin/sh
/ # mkdir -p /test/dev
/ # mknod -m 666 /test/dev/urandom c 1 8
/ # stat /test/dev/urandom
  File: /test/dev/urandom
  Size: 0         	Blocks: 0          IO Block: 4096   character special file
Device: 31h/49d	Inode: 530019      Links: 1     Device type: 1,8
Access: (0666/crw-rw-rw-)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-01-06 23:50:06.056650747 +0000
Modify: 2022-01-06 23:50:06.056650747 +0000
Change: 2022-01-06 23:50:06.056650747 +0000
```

> [!NOTE]
> Generalmente, agregar capacidades a los contenedores implica ampliar la superficie de ataque potencial que un atacante malicioso podría usar. Si no es necesario, es una buena práctica mantener las capacidades predeterminadas y eliminar las no deseadas una vez analizados los posibles efectos secundarios.

En esta sección, aprendimos a administrar las capacidades dentro de los contenedores. Sin embargo, las capacidades no son el único aspecto de seguridad a considerar al proteger los contenedores. SELinux, como aprenderemos en la siguiente sección, tiene un papel crucial en garantizar el aislamiento de los contenedores.

---

### Comprensión de la interacción de SELinux con los contenedores

En esta sección, analizaremos las políticas de SELinux y presentaremos Udica, una herramienta utilizada para generar perfiles de SELinux para contenedores.

SELinux funciona directamente en el espacio del kernel y administra el aislamiento de objetos siguiendo un modelo de privilegio mínimo que contiene una serie de políticas que pueden manejar la aplicación o las excepciones. Para definir estos objetos, SELinux utiliza etiquetas (*labels*) que definen tipos. De forma predeterminada, SELinux funciona en modo Enforcing, denegando el acceso a los recursos con una serie de excepciones definidas por políticas. Para deshabilitar el modo Enforcing, SELinux se puede poner en modo Permissive, donde las violaciones solo se auditan, sin ser bloqueadas.

> [!CAUTION]
> Como mencionamos anteriormente, cambiar SELinux al modo Permissive o deshabilitarlo por completo no es una buena práctica, ya que te expone a posibles amenazas de seguridad. En lugar de hacer eso, los usuarios deben crear políticas personalizadas para administrar las excepciones necesarias.

De forma predeterminada, SELinux utiliza un tipo de política dirigida (*targeted policy*), que intenta apuntar y confinar tipos de objetos específicos (procesos, archivos, dispositivos, etc.) utilizando un conjunto de políticas predefinidas.

SELinux permite diferentes tipos de control de acceso. Se pueden resumir de la siguiente manera:

- **Type Enforcement (TE)**: Controla el acceso a los recursos según los tipos de proceso y archivo. Este es el caso de uso principal del control de acceso de SELinux.
- **Control de acceso basado en roles (*Role-Based Access Control* o RBAC)**: Controla el acceso a los recursos mediante usuarios de SELinux (que se pueden asignar a usuarios reales del sistema) y sus roles de SELinux asociados.
- **Seguridad multinivel (*Multi-Level Security* o MLS)**: Otorga acceso de lectura/escritura a los recursos a todos los procesos con el mismo nivel de sensibilidad.
- **Seguridad multicategoría (*Multi-Category Security* o MCS)**: Controla el acceso mediante categorías, que son etiquetas de texto sin formato que se aplican a los recursos. Las categorías se utilizan para crear compartimentos de objetos, junto con las otras etiquetas de SELinux. Solo los procesos que pertenecen a la misma categoría pueden acceder a un recurso determinado. En el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*, analizamos MCS y cómo podemos asignar categorías a los recursos a los que han accedido los contenedores.

Con la aplicación de tipos (Type Enforcement), los archivos del sistema reciben etiquetas llamadas **tipos**, mientras que los procesos reciben etiquetas llamadas **dominios**. Se puede permitir que un proceso que pertenece a un dominio acceda a un archivo que pertenece a un tipo dado, y este acceso puede ser auditado por SELinux.

Por ejemplo, según SELinux, el proceso Apache httpd, que está etiquetado con el dominio `httpd_t`, puede acceder a archivos o directorios con etiquetas `httpd_sys_content_t`.

Una política de tipo SELinux se basa en el siguiente patrón:

```text
POLICY DOMAIN TYPE:CLASS OPERATION;
```

Aquí, `POLICY` es el tipo de política (`allow`, `allowxperm`, `auditallow`, `neverallow`, `dontaudit`, etc.), `DOMAIN` es el dominio del proceso, `TYPE` es el contexto del tipo de recurso, `CLASS` es la categoría del objeto (por ejemplo, `file`, `dir`, `lnk_file`, `chr_file`, `blk_file`, `sock_file` o `fifo_file`), y `OPERATION` es una lista de acciones manejadas por la política (por ejemplo, `open`, `read`, `use`, `lock`, `getattr` o `revc`).

El siguiente ejemplo muestra una regla `allow` básica:

```text
allow myapp_t myapp_log_t:file { read_file_perms append_file_perms };
```

En este ejemplo, el proceso que se ejecuta en el dominio `myapp_t` tiene permitido acceder a archivos del tipo `myapp_log_t` y realizar las acciones `read_file_perms` y `append_file_perms`.

SELinux administra las políticas de manera modular, lo que permite a los usuarios cargar y descargar dinámicamente módulos de políticas sin la necesidad de recompilar todo el conjunto de políticas cada vez. Las políticas se pueden cargar y descargar utilizando la utilidad `semodule`, como se muestra en el siguiente ejemplo, que muestra cómo cargar una política personalizada:

```bash
# semodule -i custompolicy.pp
```

La utilidad `semodule` también se puede utilizar para ver todas las políticas cargadas:

```bash
# semodule -l
```

En Fedora, CentOS, RHEL y distribuciones derivadas, la política binaria actual se instala en el directorio `/etc/selinux/targeted/policy` en un archivo llamado `policy.XX`, donde `XX` representa la versión de la política.

En las mismas distribuciones, las políticas de contenedores se definen dentro del paquete `container-selinux`, que contiene el módulo SELinux ya compilado. El código fuente del paquete está disponible en GitHub si deseas verlo con más detalle: [https://github.com/containers/container-selinux](https://github.com/containers/container-selinux).

Al mirar el contenido del repositorio, encontraremos los tres archivos fuente de políticas más importantes para desarrollar cualquier módulo:

- `container.fc`: Este archivo define los archivos y directorios que están vinculados a los tipos definidos en el módulo.
- `container.te`: Este archivo define las reglas de política, los atributos y los alias.
- `container.if`: Este archivo define la interfaz del módulo. Contiene un conjunto de funciones macro públicas expuestas por el módulo.

Un proceso que se ejecuta dentro de un contenedor está etiquetado con el dominio `container_t`. Tiene acceso de lectura/escritura a recursos etiquetados con el contexto de tipo `container_file_t` y acceso de lectura/ejecución a recursos etiquetados con el contexto de tipo `container_share_t`.

Cuando se ejecuta un contenedor, el proceso `podman`, así como el tiempo de ejecución del contenedor y el proceso `conmon`, se ejecutan con el tipo de dominio `container_runtime_t` y solo se les permite ejecutar procesos que hacen la transición a tipos específicos. Esos tipos se agrupan en el atributo `container_domain` y se pueden inspeccionar con la utilidad `seinfo` (instalada con el paquete `setools-console` en Fedora), como se muestra en el siguiente código:

```bash
$ seinfo -a container_domain -x
Type Attributes: 1
   attribute container_domain;
	container_device_plugin_init_t
	container_device_plugin_t
	container_device_t
	container_engine_t
	container_init_t
	container_kvm_t
	container_logreader_t
	container_logwriter_t
	container_t
	container_userns_t
```

El atributo `container_domain` se declara en el archivo fuente `container.te` en el repositorio `container-policy` utilizando la palabra clave `attribute`:

```text
attribute container_domain;
attribute container_user_domain;
attribute container_net_domain;
```

Los atributos anteriores se asignan al tipo `container_t` mediante una declaración `typeattribute`:

```text
typeattribute container_t container_domain, container_net_domain, container_user_domain;
```

Siguiendo este enfoque, SELinux garantiza el aislamiento de procesos entre contenedores y entre un contenedor y su host. De esta manera, un proceso que escapa del contenedor (quizás aprovechando una vulnerabilidad) no puede acceder a recursos en el host o dentro de otros contenedores.

Cuando se crea un contenedor, las capas de solo lectura de la imagen, que forman el conjunto `LowerDirs` de OverlayFS, se etiquetan con el tipo `container_ro_file_t`, lo que evita que el contenedor escriba dentro de esos directorios. Al mismo tiempo, `MergedDir`, que es la suma de `LowerDirs` y `UpperDir`, es escribible y está etiquetado como `container_file_t`.

Para probar esto, ejecutemos un contenedor rootful con las categorías MCS `c1` y `c2`:

```bash
# podman run -d --name selinux_test1 --security-opt label=level:s0:c1,c2 nginx
```

Ahora, podemos encontrar todos los archivos etiquetados como `container_file_t:s0:c1,c2` en el sistema de archivos del host:

```bash
# find /var/lib/containers/storage/overlay -type f -context '*container_file_t:s0:c1,c2*' -printf '%-50Z%p\n'
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/x86_64-linux-gnu/libreadline.so.8.1
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/x86_64-linux-gnu/libhistory.so.8.1
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/x86_64-linux-gnu/libexpat.so.1.6.12
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/udev/rules.d/96-e2scrub.rules
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/terminfo/r/rxvt-unicode-256color
system_u:object_r:container_file_t:s0:c1,c2        /var/lib/containers/storage/overlay/4b147975bb5c336b10e71d21c49fe88ddb00d0569b77ddab1d7737f80056677b/merged/lib/terminfo/r/rxvt-unicode
[…output omitted...]
```

Como era de esperar, la etiqueta `container_file_t`, que está asociada con las categorías `c1` y `c2`, se aplica a todos los archivos dentro del contenedor `MergedDir`.

Al mismo tiempo, podemos demostrar que los `LowerDirs` del contenedor están etiquetados como `container_ro_file_t`. Primero, necesitamos extraer la lista de `LowerDirs` del contenedor:

```bash
# podman inspect selinux_test1 \
  --format '{{.GraphDriver.Data.LowerDir}}'
/var/lib/containers/storage/overlay/9566cbcf1773eac59951c14c52156a6164db1b0d8026d015e193774029db18a5/diff:/var/lib/containers/storage/overlay/24de59cced7931bbcc0c4a34d4369c15119a0b8b180f98a0434fa76a6dfcd490/diff:/var/lib/containers/storage/overlay/1bb84245b98b7e861c91ed4319972ed3287bdd2ef02a8657c696a76621854f3b/diff:/var/lib/containers/storage/overlay/97f26271fef21bda129ac431b5f0faa03ae0b2b50bda6af969315308fc16735b/diff:/var/lib/containers/storage/overlay/768ef71c8c91e4df0aa1caf96764ceec999d7eb0aa584e241246815c1fa85435/diff:/var/lib/containers/storage/overlay/2edcec3590a4ec7f40cf0743c15d78fb39d8326bc029073b41ef9727da6c851f/diff
```

El directorio situado más a la derecha representa la capa más baja del contenedor y suele ser el árbol del sistema de archivos base de la imagen. Inspeccionemos el contexto de tipo de este directorio:

```bash
# ls -alZ /var/lib/containers/storage/overlay/2edcec3590a4ec7f40cf0743c15d78fb39d8326bc029073b41ef9727da6c851f/diff
total 84
dr-xr-xr-x. 21 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Jan  5 23:16 .
drwx------.  6 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Jan  5 23:16 ..
drwxr-xr-x.  2 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 20 00:00 bin
drwxr-xr-x.  2 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 11 17:25 boot
drwxr-xr-x.  2 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 20 00:00 dev
drwxr-xr-x. 30 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 20 00:00 etc
drwxr-xr-x.  2 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 11 17:25 home
drwxr-xr-x.  8 root root unconfined_u:object_r:container_ro_file_t:s0 4096 Dec 20 00:00 lib
[...omitted output...]
```

La salida anterior también muestra otro aspecto interesante: dado que las capas `LowerDir` se comparten entre múltiples contenedores que usan la misma imagen, no encontraremos ninguna categoría MCS que se haya aplicado aquí.

Los contenedores no tienen acceso de lectura/escritura a archivos o directorios que no estén etiquetados como `container_file_t`. Anteriormente, vimos que es posible volver a etiquetar esos archivos aplicando el sufijo `:z` a los volúmenes montados o volviéndolos a etiquetar manualmente con anticipación antes de ejecutar los contenedores.

Sin embargo, volver a etiquetar directorios cruciales como `/home` o `/var/log` es una muy mala idea, ya que muchos otros procesos no contenedorizados ya no podrán acceder a ellos.

La única solución es crear manualmente políticas personalizadas que anulen el comportamiento predeterminado. Sin embargo, esto es demasiado complejo de gestionar en el uso diario y en entornos de producción.

Afortunadamente, podemos resolver esta limitación con una herramienta que genera perfiles de seguridad de SELinux personalizados para nuestros contenedores: **Udica**.

#### Presentación de Udica

Udica es un proyecto de código abierto ([https://github.com/containers/udica](https://github.com/containers/udica)) creado por Lukas Vrabec, evangelista de SELinux y líder de equipo de los equipos de ingeniería de Proyectos Especiales de Seguridad y SELinux en Red Hat.

Udica tiene como objetivo superar las rígidas limitaciones de políticas descritas anteriormente mediante la generación de perfiles de SELinux para contenedores, lo que les permite acceder a recursos que normalmente se evitarían con el dominio común `container_t`.

Para instalar Udica en Fedora, simplemente ejecuta el siguiente comando:

```bash
$ sudo dnf install -y udica setools-console container-selinux
```

En otras distribuciones, Udica se puede instalar desde su código fuente ejecutando los siguientes comandos:

```bash
$ sudo dnf install -y setools-console git container-selinux
$ git clone https://github.com/containers/udica.git
$ cd udica && sudo python3 ./setup.py install
```

Para demostrar cómo funciona Udica, vamos a crear un contenedor que escribe en el directorio `/var/log` del host, el cual se monta mediante un montaje de enlace (*bind mount*) cuando se crea el contenedor. De forma predeterminada, el proceso con el dominio `container_t` no podría escribir en un directorio etiquetado con el tipo `var_log_t`.

El siguiente script, que se ejecuta dentro del contenedor, es un bucle sin fin que escribe una línea de registro compuesta por la fecha actual y un contador (`Chapter10/custom_logger/logger.sh`):

```bash
#!/bin/bash
set -euo pipefail
trap "echo Exited; exit;" SIGINT SIGTERM

# Run an endless loop writing a simple log entry with date
count=1
while true; do
    echo "$(date +%y/%m/%d_%H:%M:%S) - Line #$count" | tee -a /var/log/custom.log
    count=$((count+1))
    sleep 2
done
```

El script anterior usa la opción `set -euo pipefail` para salir inmediatamente en caso de que ocurra un error, y la utilidad `tee` para escribir la salida del comando tanto en la salida estándar como en el archivo `/var/log/custom.log` en modo de adición (*append*). La variable `count` se incrementa en cada ciclo del bucle.

El Dockerfile para este contenedor se mantiene mínimo: solo copia el script de registro y lo ejecuta al iniciar el contenedor (`Chapter10/custom_logger/Dockerfile`):

```dockerfile
FROM docker.io/library/fedora
# Copy the logger.sh script
COPY --chmod=755 logger.sh /
# Exec the logger.sh script
CMD ["/logger.sh"]
```

> [!IMPORTANT]
> El script `logger.sh` se copia con la opción `--chmod=755` para configurar los permisos ejecutables adecuados para que se pueda iniciar correctamente al arrancar el contenedor.

La imagen del contenedor se compila con el nombre `custom_logger`:

```bash
# cd /Chapter10/custom_logger
# buildah build -t custom_logger .
```

Ahora, es el momento de probar el contenedor y ver cómo se comporta. El directorio `/var/log` se monta con permisos `rw` en la ruta del contenedor `/var/log`, sin alterar su contexto de tipo. Debemos mantener la ejecución en primer plano para ver la salida inmediata:

```bash
# podman run -v /var/log:/var/log:rw \
  --name custom_logger1 custom_logger
tee: /var/log/custom.log: Permission denied
22/01/08_09:09:33 - Custom log event #1
```

Como era de esperar, el script no pudo escribir en el archivo de destino. Podríamos solucionar esto cambiando el contexto del tipo de directorio a `container_file_t`, pero, como aprendimos anteriormente, es una mala idea ya que evitaría que otros procesos escriban sus registros.

En su lugar, podemos usar Udica para generar un perfil de seguridad de SELinux personalizado para el contenedor. En el siguiente código, las especificaciones del contenedor se exportan a un archivo `container.json` y luego Udica las analiza para generar un perfil personalizado llamado `custom_logger`:

```bash
# podman inspect custom_logger1 > container.json
# udica -j container.json custom_logger
Policy custom_logger created!
Please load these modules using:
# semodule -i custom_logger.cil /usr/share/udica/templates/{base_container.cil,log_container.cil}
Restart the container with: "--security-opt label=type:custom_logger.process" parameter
```

Una vez que se ha generado el perfil, Udica genera las instrucciones para configurar el contenedor. Primero, necesitamos cargar la nueva política personalizada usando la utilidad `semodule`. El archivo generado está en formato Common Intermediate Language (CIL), un lenguaje de políticas intermedio para SELinux. Junto con el archivo CIL generado, el ejemplo carga algunas plantillas de Udica, `/usr/share/udica/templates/base_container.cil` y `/usr/share/udica/templates/log_container.cil`, cuyas reglas se heredan en el archivo de política de contenedor personalizado.

Carguemos los módulos usando el comando sugerido:

```bash
# semodule -i custom_logger.cil /usr/share/udica/templates/{base_container.cil,log_container.cil}
```

Después de cargar los módulos en SELinux, estamos listos para ejecutar el contenedor con la etiqueta personalizada `custom_logger.process`, que se pasa como argumento a la opción `--security-opt` de Podman.

Todas las demás opciones del contenedor se mantuvieron idénticas, excepto su nombre, que se actualizó a `custom_logger2` para diferenciarlo de la instancia anterior:

```bash
# podman run -v /var/log:/var/log:rw \
  --name custom_logger2 \
  --security-opt label=type:custom_logger.process \
  custom_logger
22/01/08_09:05:19 - Line #1
22/01/08_09:05:21 - Line #2
22/01/08_09:05:23 - Line #3
22/01/08_09:05:25 - Line #5
[...Omitted output...]
```

Esta vez, el script escribió correctamente en el archivo `/var/log/custom.log` gracias al perfil personalizado generado con Udica.

Ten en cuenta que los procesos del contenedor no se ejecutan con el dominio `container_t`, sino con el nuevo superconjunto `custom_logger.process`, que incluye reglas adicionales además de las predeterminadas.

Podemos confirmar esto ejecutando el siguiente comando en el host:

```bash
# ps auxZ | grep 'custom_logger.process'
unconfined_u:system_r:container_runtime_t:s0-s0:c0.c1023 root 26546 0.1  0.6 1365088 53768 pts/0 Sl+ 09:16   0:00 podman run -v /var/log:/var/log:rw --security-opt label=type:custom_logger.process custom_logger
system_u:system_r:custom_logger.process:s0:c159,c258 root 26633 0.0  0.0   4180  3136 ?        Ss   09:16   0:00 /bin/bash /logger.sh
system_u:system_r:custom_logger.process:s0:c159,c258 root 26881 0.0  0.0   2640  1104 ?        S    09:18   0:00 sleep 2
```

Udica crea la política personalizada analizando el archivo de especificaciones JSON y buscando los puntos de montaje, los puertos y las capacidades del contenedor. Veamos el contenido del archivo `custom_logger.cil` generado a partir de nuestro ejemplo:

```cil
(block custom_logger
    (blockinherit container)
    (allow process process ( capability ( chown dac_override fowner fsetid kill net_bind_service setfcap setgid setpcap setuid sys_chroot )))
    (blockinherit log_rw_container)
)
```

La sintaxis del lenguaje CIL está fuera del alcance de este libro, pero aún podemos notar algunas cosas interesantes:

- El perfil `custom_logger` se define mediante una declaración `block`.
- La regla `allow` habilita las capacidades predeterminadas para el contenedor.
- La política hereda los bloques `container` y `log_rw_container` con las declaraciones `blockinherit`.

El archivo CIL generado hereda los bloques que se han definido en las plantillas de Udica disponibles, cada una enfocada en acciones específicas. En Fedora, las plantillas se instalan mediante el paquete `container-selinux` y están disponibles en la carpeta `/usr/share/udica/templates/`:

```bash
# ls -1 /usr/share/udica/templates/
base_container.cil
config_container.cil
home_container.cil
log_container.cil
net_container.cil
tmp_container.cil
tty_container.cil
virt_container.cil
x_container.cil
```

Las plantillas disponibles se implementan para escenarios comunes, como acceder a directorios de registro o directorios de inicio de usuarios, o incluso abrir puertos de red. Entre ellas, la plantilla `base_container.cil` siempre es incluida por todas las políticas generadas por Udica como el bloque de construcción base utilizado para generar las políticas personalizadas.

Según el comportamiento del contenedor que se deriva del archivo de especificaciones, se incluyen otras plantillas. Por ejemplo, la política hereda el bloque `log_rw_container` de la plantilla `log_container.cil` para permitir que el contenedor del registrador personalizado acceda al directorio `/var/log`.

Udica es una gran herramienta para abordar problemas de aislamiento de contenedores y ayuda a los administradores a abordar los casos de uso de confinamiento de SELinux superando la complejidad de escribir reglas manualmente.

Los perfiles de seguridad generados también se pueden versionar dentro de un repositorio de GitHub y reutilizarse para contenedores similares en diferentes hosts.

---

### Resumen

En este capítulo, aprendimos cómo desarrollar y aplicar técnicas para mejorar la seguridad general de nuestra arquitectura de servicios basada en contenedores. Aprendimos cómo el aprovechamiento de contenedores rootless y el evitar el UID 0 pueden reducir la superficie de ataque de nuestros servicios. Luego, aprendimos cómo firmar y confiar en imágenes de contenedores para evitar ataques MITM. Finalmente, profundizamos en las herramientas de contenedores y analizamos las capacidades del kernel de Linux y el subsistema SELinux, que pueden ayudarnos a ajustar con precisión varios aspectos de seguridad para nuestros contenedores en ejecución.

Ahora que hemos profundizado en la seguridad, estamos listos para pasar al siguiente capítulo, donde veremos de manera avanzada las redes para contenedores.

---

### Lecturas adicionales

Para obtener más información sobre los temas que se trataron en este capítulo, echa un vistazo a los siguientes recursos:

- Matriz de contenedores MITRE ATT&CK: [https://attack.mitre.org/matrices/enterprise/containers/](https://attack.mitre.org/matrices/enterprise/containers/)
- Página de inicio del proyecto Sigstore: [https://docs.sigstore.dev/](https://docs.sigstore.dev/)
- Tutorial de firma de imágenes de Podman: [https://github.com/containers/podman/blob/main/docs/tutorials/image_signing.md](https://github.com/containers/podman/blob/main/docs/tutorials/image_signing.md)
- Blog de Lukas Vrabec: [https://lukas-vrabec.com/](https://lukas-vrabec.com/)
- Introducción y principios de diseño de CIL: [https://github.com/SELinuxProject/cl/wiki](https://github.com/SELinuxProject/cl/wiki)
- Introducción a Udica en el blog de Red Hat: [https://www.redhat.com/en/blog/generate-selinux-policies-containers-with-udica](https://www.redhat.com/en/blog/generate-selinux-policies-containers-with-udica)
