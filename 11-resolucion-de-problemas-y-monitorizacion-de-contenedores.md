# Parte 3: Administración e Integración Segura de Contenedores

## Capítulo 11: Resolución de Problemas y Monitorización de Contenedores

Ejecutar un contenedor podría confundirse con el objetivo final de un equipo de DevOps, pero en cambio, este es solo el primer paso de un largo viaje. Los administradores de sistemas deben asegurarse de que sus sistemas funcionen correctamente para mantener los servicios en marcha; de la misma manera, el equipo de DevOps debe asegurarse de que sus contenedores funcionen correctamente.

En las actividades de gestión de contenedores, tener el conocimiento adecuado de las técnicas de resolución de problemas (*troubleshooting*) realmente podría ayudar a minimizar cualquier impacto en los servicios finales, reduciendo el tiempo de inactividad. Hablando de problemas y solución de problemas, una buena práctica es mantener la monitorización de los contenedores para interceptar fácilmente cualquier problema o error y acelerar la recuperación.

En este capítulo, vamos a cubrir los siguientes temas principales:

- Resolución de problemas en contenedores en ejecución
- Monitorización de contenedores con comprobaciones de estado (*health checks*)
- Inspección de los resultados de compilación de nuestros contenedores
- Resolución avanzada de problemas con `nsenter`

---

### Requisitos técnicos

Antes de continuar con la información y los ejemplos del capítulo, se requiere una máquina con una instalación de Podman en funcionamiento. Como se indicó en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, todos los ejemplos del libro se ejecutan en un sistema Fedora 40 o posterior, pero se pueden reproducir en el sistema operativo de tu elección.

Una buena comprensión de los temas tratados en el [Capítulo 4](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/4), *Administración de Contenedores en Ejecución*, y el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*, será útil para asimilar fácilmente los conceptos relacionados con los registros de contenedores.

---

### Resolución de problemas en contenedores en ejecución

La resolución de problemas en contenedores es una práctica importante con la que necesitamos experiencia para resolver problemas comunes e investigar cualquier error que podamos encontrar en la capa del contenedor o en la aplicación que se ejecuta dentro de nuestros contenedores.

A partir del [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, comenzamos a trabajar con comandos básicos de Podman para ejecutar y luego inspeccionar contenedores en nuestro sistema host. Vimos cómo podemos recopilar registros con el comando `podman logs`, y también aprendimos a usar la información proporcionada por el comando `podman inspect`. Finalmente, también debemos considerar echar un vistazo a la salida del útil comando `podman system df`, que informará sobre el uso del almacenamiento para nuestros contenedores e imágenes, y también al útil comando `podman info`, que mostrará información útil sobre el host donde estamos ejecutando Podman.

En general, siempre debemos considerar que el contenedor en ejecución es simplemente un proceso en el sistema host, por lo que siempre tenemos todas las herramientas y comandos disponibles para solucionar problemas del sistema operativo subyacente y sus recursos disponibles.

Una mejor práctica para la resolución de problemas de contenedores podría ser un enfoque de arriba hacia abajo (*top-down*), analizando primero la capa de la aplicación, luego pasando a la capa del contenedor y finalmente bajando al sistema host base.

A nivel de contenedor, muchos de los problemas que podemos encontrar han sido resumidos por el equipo del proyecto Podman en una lista completa en la página del proyecto (consulta el enlace compartido en la sección *Lecturas adicionales*). Cubriremos algunos de los más útiles en las siguientes secciones.

#### Permiso denegado al usar volúmenes de almacenamiento

Un problema muy común que podemos encontrar durante nuestras actividades en RHEL, Fedora o cualquier distribución de Linux que utilice el subsistema de seguridad SELinux está relacionado con los permisos de almacenamiento. El error que se describe a continuación se desencadena cuando SELinux se establece en modo Enforcing, que también es el enfoque sugerido para garantizar plenamente las características de seguridad de acceso obligatorio de SELinux.

##### La capa de SELinux: contextos de seguridad

La primera barrera suele ser el subsistema de seguridad SELinux. SELinux utiliza etiquetas (atributos extendidos del sistema de archivos) para garantizar que los procesos de un contenedor no puedan escapar y acceder a archivos que pertenecen al host o a otros contenedores.

De forma predeterminada, si creas un directorio en tu host y lo montas en un contenedor, el proceso en contenedor no tendrá la etiqueta SELinux correcta para interactuar con él.

```bash
$ mkdir ~/mycontent
$ podman run -v ~/mycontent:/content fedora \
  touch /content/file
touch: cannot touch '/content/file': Permission denied
```

Como podemos ver, el comando `touch` informa un error de `Permission denied`, porque en realidad no puede escribir en el sistema de archivos.

Como vimos en detalle en el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/5), *Implementación del Almacenamiento para los Datos del Contenedor*, SELinux aplica etiquetas de forma recursiva a archivos y directorios para definir su contexto. Esas etiquetas generalmente se almacenan como atributos extendidos del sistema de archivos. SELinux utiliza contextos para administrar políticas y definir qué procesos pueden acceder a un recurso específico.

El contenedor que acabamos de ejecutar obtuvo su propio espacio de nombres de Linux y una etiqueta SELinux que es completamente diferente del usuario local en la estación de trabajo Fedora, razón por la cual obtuvimos ese error antes.

Sin una etiqueta adecuada, el sistema SELinux evita que los procesos que se ejecutan en el contenedor accedan al contenido. Esto también se debe a que Podman no cambia las etiquetas establecidas por el sistema operativo si no se solicita explícitamente a través de una opción de comando.

##### La solución: reetiquetado con :z y :Z

Para permitir que Podman cambie la etiqueta de un contenedor, podemos usar cualquiera de los dos sufijos, `:z` o `:Z`, para el montaje del volumen. Estas opciones le indican a Podman que vuelva a etiquetar los objetos de archivo en el volumen.

Para indicarle a Podman que ajuste automáticamente las etiquetas SELinux para el volumen montado, usamos los sufijos `:z` o `:Z`:

- `:z` (Compartido / *Shared*): Le indica a Podman que el contenido del volumen se compartirá entre varios contenedores. Aplica una etiqueta de contenido compartido.
- `:Z` (Privado / *Private*): Le indica a Podman que aplique una etiqueta privada no compartida. Esta es la opción más segura si solo un contenedor necesita acceso a esos datos específicos. El comando resultaría en algo como esto:

```bash
# Using :Z to grant exclusive access to the container
$ podman run -v ~/mycontent:/content:Z fedora \
  touch /content/file
```

Como podemos ver, el comando no arrojó ningún error; funcionó.

##### La capa rootless: espacios de nombres de usuario (UID/GID)

Incluso si resuelves el problema de SELinux, es posible que aún encuentres el error `Permission denied`. En un entorno rootless, esto generalmente se debe a cómo Podman asigna usuarios entre el host y el contenedor, una característica llamada espacios de nombres de usuario (*user namespaces*).

En el mundo de los contenedores Linux, un espacio de nombres de usuario es una característica del kernel que proporciona aislamiento de ID de usuario (UID) e ID de grupo (GID). Básicamente permite que un proceso tenga una identidad diferente dentro del espacio de nombres que la que tiene en el sistema host.

En una configuración rootless, tú eres tú en el host (por ejemplo, UID 1000), pero dentro del contenedor, normalmente apareces como root (UID 0). Sin embargo, tu identidad de "root" dentro del contenedor es en realidad un alias para tu usuario en el host. Si el contenedor intenta ejecutar un proceso como un usuario diferente (como un usuario MySQL o nginx con UID 999), es posible que ese UID no tenga permiso para escribir en la carpeta host que acabas de montar.

##### Identificación del desajuste

Puedes ver esta asignación en acción usando `podman top`. Si ejecutas un contenedor como un usuario específico, Podman asigna ese ID a un sub-UID de rango alto en tu host, definido en `/etc/subuid`.

```bash
$ podman run -d --name web -v ~/data:/data:Z nginx
$ podman top web user,huser
USER    HUSER
root    1000      # Inside is root, outside is your local user
nginx   100098    # Inside is nginx (999), outside is a sub-UID
```

En este ejemplo, el proceso nginx dentro del contenedor se ejecuta en realidad como el UID de host 100098. Dado que es probable que tu carpeta de host `~/data` sea propiedad del UID 1000, el usuario de nginx está bloqueado.

##### La solución: el indicador :U

La forma más eficaz de resolver esto en el Podman moderno, versión 3.1 y posteriores, es la opción de montaje `:U`. Esto le indica a Podman que cambie automáticamente la propiedad (*chown*) del directorio host para que coincida con el UID y GID utilizados dentro del contenedor.

```bash
$ podman run -v ~/mycontent:/content:Z,U fedora touch /content/file
```

##### La solución: podman unshare

Si necesitas corregir manualmente los permisos para un directorio de host de modo que un contenedor rootless pueda usarlo, no puedes simplemente usar el comando estándar `sudo chown`. Debes usar `podman unshare`, que ejecuta un comando dentro del mismo espacio de nombres de usuario que usa el contenedor.

```bash
# This sets the host directory to be owned by the internal 'root' of the container
$ podman unshare chown -R 0:0 ~/mycontent
```

##### La solución: --userns=keep-id

Si tu aplicación en contenedor necesita ver exactamente el mismo UID que tu usuario de host, lo cual es común para las herramientas de desarrollo, usa el indicador `--userns=keep-id`. Esto garantiza que el UID 1000 en el host siga siendo el UID 1000 dentro del contenedor.

```bash
$ podman run --userns=keep-id -v ~/mycontent:/content:Z fedora ...
```

Dependiendo del problema que encuentres, es posible que tengas una o varias formas de superar y solucionar el problema para continuar trabajando. Sigamos viendo cómo solucionar otros problemas en la siguiente sección.

#### Problemas con el comando ping en contenedores rootless

En algunos sistemas Linux reforzados (*hardened*), la ejecución del comando `ping` podría limitarse a un grupo restringido de usuarios. Esto podría provocar el fallo del comando `ping` utilizado en un contenedor.

Como vimos en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/3), *Ejecución del Primer Contenedor*, al iniciar el contenedor, el sistema operativo base asociará con él un ID de usuario diferente al utilizado en el propio contenedor. El ID de usuario asociado con el contenedor puede quedar fuera del rango permitido de ID de usuario autorizados a usar el comando ping.

En una instalación de estación de trabajo Fedora, la configuración predeterminada permitirá que cualquier contenedor ejecute el comando ping sin problemas. Para administrar las restricciones sobre el uso del comando ping, Fedora usa el parámetro del kernel `ping_group_range`, que define los grupos del sistema permitidos que pueden ejecutar el comando ping.

Si echamos un vistazo a una estación de trabajo Fedora recién instalada, el rango predeterminado es el siguiente:

```bash
$ cat /proc/sys/net/ipv4/ping_group_range
0 2147483647
```

Por lo tanto, no hay nada de qué preocuparse en un sistema Fedora nuevo. Pero, ¿qué pasa si el rango es menor que este?

Bueno, probamos este comportamiento cambiando el rango permitido con un comando simple. En este ejemplo, vamos a restringir el rango y ver que el comando ping fallará:

```bash
$ sudo sysctl -w "net.ipv4.ping_group_range=0 0"
```

En caso de que el rango sea menor que el reportado en la salida anterior, podemos hacerlo persistente agregando un archivo a `/etc/sysctl.d` que contenga `net.ipv4.ping_group_range=0 0`.

El cambio aplicado en el rango del grupo de ping afectará a los privilegios de usuario asignados para ejecutar el comando ping dentro del contenedor.

Comencemos construyendo una imagen basada en Fedora con el paquete `iputils` (no incluido de forma predeterminada) usando Buildah:

```bash
$ container=$(buildah from docker.io/library/fedora) && \
  buildah run $container -- dnf install -y iputils && \
  buildah commit $container ping_example
```

Podemos probarlo ejecutando el siguiente comando dentro de un contenedor:

```bash
$ podman run --rm ping_example ping -W10 -c1 redhat.com
PING redhat.com (209.132.183.105): 56 data bytes
--- redhat.com ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
```

El comando, ejecutado en un sistema con un rango restringido, produce una pérdida de paquetes del 100% ya que el comando ping no puede enviar paquetes a través de un socket sin procesar (*raw socket*).

> [!IMPORTANT]
> No olvides restaurar el `ping_group_range` original antes de continuar con los siguientes ejemplos. En Fedora, la configuración predeterminada se puede restaurar con el comando `sudo sysctl -w "net.ipv4.ping_group_range=0 2147483647"` y eliminando cualquier configuración persistente aplicada en `/etc/sysctl.d` durante el ejercicio. Para una imagen base de contenedor que estamos construyendo a través de un Dockerfile, es posible que necesitemos agregar un nuevo usuario con un UID/GID grande. Esto creará un archivo `/var/log/lastlog` grande y disperso (*sparse*), y esto puede hacer que la compilación se bloquee indefinidamente. Este problema está relacionado con el lenguaje Go, que no admite correctamente archivos dispersos, lo que lleva a la creación de este archivo enorme en la imagen del contenedor.

El ejemplo demuestra cómo una restricción en `ping_group_range` afecta la ejecución de ping dentro de un contenedor rootless. Al establecer el rango en un valor lo suficientemente grande como para incluir el GID del grupo privado del usuario (o uno de los grupos secundarios del usuario), el comando ping podrá enviar paquetes ICMP correctamente.

> [!NOTE]
> El archivo `/var/log/lastlog` es un archivo binario y disperso (*sparse file*) que contiene información sobre la última vez que los usuarios iniciaron sesión en el sistema. El tamaño aparente de un archivo disperso informado por `ls -l` es mayor que el uso real del disco. Un archivo disperso intenta utilizar el espacio del sistema de archivos de una manera más eficiente, escribiendo en el disco los metadatos que representan los bloques vacíos en lugar del espacio vacío que debería almacenarse en el bloque. Esto utilizará menos espacio en disco.

Como se mencionó en los primeros párrafos de esta sección, el equipo de Podman ha creado una lista larga pero no exhaustiva de problemas comunes. Sugerimos encarecidamente echarle un vistazo si se encuentra algún problema: [https://github.com/containers/podman/blob/main/troubleshooting.md](https://github.com/containers/podman/blob/main/troubleshooting.md).

La resolución de problemas puede ser compleja, pero el primer paso es siempre la identificación de un problema. Por este motivo, una herramienta de monitorización podría ayudar a alertar lo antes posible en caso de problemas. Veamos cómo monitorizar contenedores con comprobaciones de estado en la siguiente sección.

---

### Monitorización de contenedores con comprobaciones de estado (*Health Checks*)

Podman admite la opción de agregar una comprobación de estado (*health check*) a los contenedores. Profundizaremos en estas comprobaciones de estado en esta sección y en cómo utilizarlas.

Una comprobación de estado es una función de Podman que puede ayudar a determinar el estado o la preparación del proceso que se ejecuta en un contenedor. Podría ser tan simple como verificar que el proceso del contenedor se esté ejecutando, pero también más sofisticado, como verificar que tanto el contenedor como sus aplicaciones respondan utilizando, por ejemplo, conexiones de red.

En el mundo de los contenedores, existe una clara diferencia entre un contenedor que está en ejecución (*running*) y un contenedor que está en buen estado (*healthy*). De forma predeterminada, Podman considera que un contenedor está activo siempre que su proceso principal (PID 1) no haya salido. Sin embargo, un proceso de servidor web podría seguir ejecutándose incluso si está bloqueado (*deadlocked*), sin memoria o sin poder conectarse a su base de datos.

Una comprobación de estado es un mecanismo de monitorización proactivo que se utiliza para garantizar que el servicio dentro del contenedor realmente funcione según lo previsto. En lugar de simplemente verificar si el proceso existe, Podman ejecuta un comando específico dentro del contenedor a intervalos regulares para verificar su estado interno.

Si bien puedes definir manualmente una comprobación de estado al iniciar un contenedor, es importante tener en cuenta que muchas imágenes de nivel empresarial vienen con comprobaciones de estado integradas directamente en la configuración de la imagen:

- **Comprobaciones de estado integradas (*Built-in health checks*)**: Muchas imágenes oficiales (como las de bases de datos o servicios web especializados) incluyen una instrucción `HEALTHCHECK` en su Containerfile. Cuando ejecutas estas imágenes, Podman inicia automáticamente la lógica de monitorización de estado sin ninguna configuración adicional por parte del administrador.
- **Comprobaciones de estado manuales (*Manual health checks*)**: Puedes anular el valor predeterminado de una imagen o agregar una nueva comprobación usando el indicador `--health-cmd`. Esto es útil para aplicaciones personalizadas donde quizás desees verificar un punto de conexión de API específico o la existencia de un archivo de bloqueo (*lock file*).

Cuando una comprobación de estado está activa, el estado del contenedor en `podman ps` pasará por varias fases:

- **Starting**: El contenedor se acaba de iniciar y el retraso inicial, o período de gracia, está activo.
- **Healthy**: El comando de comprobación de estado salió correctamente: código de salida 0.
- **Unhealthy**: El comando falló un número consecutivo de veces, definido por el límite de reintentos (*retries*).

Una comprobación de estado se compone de cinco componentes principales. El primero es el elemento principal que indicará a Podman la comprobación particular a ejecutar; los demás se utilizan para configurar la programación de la comprobación de estado. Veamos estos elementos en detalle:

- **Command**: Este es el comando que Podman ejecutará dentro del contenedor de destino. El estado del contenedor y su proceso se determinarán esperando un éxito (código de retorno 0) o un fallo (cualquier otro código de salida). Si nuestro contenedor proporciona un servidor web, por ejemplo, nuestro comando de comprobación de estado podría ser algo realmente simple, como un comando curl que intentará conectarse al puerto del servidor web para asegurarse de que responde.
- **Retries**: Define el número de comandos fallidos consecutivos que Podman tiene que ejecutar antes de que el contenedor se marque como no saludable (*unhealthy*). Si un comando se ejecuta con éxito, Podman restablecerá el contador de reintentos.
- **Interval**: Esta opción define el tiempo de intervalo entre comprobaciones de estado. Encontrar el tiempo de intervalo correcto puede ser realmente difícil y requiere algo de prueba y error. Si lo configuramos en un valor pequeño, nuestro sistema puede pasar mucho tiempo ejecutando las comprobaciones de estado. Pero si lo configuramos en un valor grande, podemos tener dificultades y encontrarnos con tiempos de espera agotados. Este valor se puede definir con un formato de hora ampliamente utilizado: `30s` o `1h5m`.
- **Start period**: En este período, Podman ignorará los fallos de comprobación de estado. Podemos considerar esto como un período de gracia que debe usarse para permitir que nuestra aplicación esté activa y comience a responder correctamente a cualquier cliente, así como a nuestras comprobaciones de estado.
- **Timeout**: Define el período de tiempo en el que la comprobación de estado debe completarse antes de que se considere fallida. Ten en cuenta que el tiempo de espera, el período de inicio y los reintentos son opcionales, mientras que el comando y el intervalo deben establecerse para una comprobación de estado exitosa.

Echemos un vistazo a un ejemplo real, suponiendo que queremos definir una comprobación de estado para un contenedor y ejecutar esa comprobación de estado manualmente:

```bash
$ podman run -dt --name healthtest1 --health-cmd 'curl http://localhost || exit 1' --health-interval=0 quay.io/libpod/alpine_nginx:latest
89e3df8713d24fb56bfe3cfd545826d605581ead6ec1bec31c3b1363428355a2
$ podman ps
CONTAINER ID  IMAGE                              COMMAND               CREATED        STATUS                  PORTS     NAMES
89e3df8713d2  quay.io/libpod/alpine_nginx:latest  nginx -g daemon o...  7 seconds ago  Up 8 seconds (starting) 80/tcp    healthtest1
```

Como podemos ver en el bloque de comandos anterior, acabamos de iniciar un nuevo contenedor llamado `healthtest1`, definiendo un comando healthcheck que ejecutará el comando curl en la dirección localhost dentro del contenedor de destino. Una vez que se inició el contenedor, permaneció en estado `starting` hasta que ejecutamos manualmente healthcheck; probémoslo con el siguiente comando:

```bash
$ podman healthcheck run healthtest1
$ echo $?
0
$ podman ps
CONTAINER ID  IMAGE                              COMMAND               CREATED         STATUS                  PORTS     NAMES
89e3df8713d2  quay.io/libpod/alpine_nginx:latest  nginx -g daemon o...  41 seconds ago  Up 41 seconds (healthy) 80/tcp    healthtest1
```

Podemos ver en la salida anterior que después de ejecutar manualmente healthcheck, su código de salida fue 0, lo que significa que la verificación se completó con éxito y nuestro contenedor está marcado como saludable (*healthy*). En el ejemplo anterior, también usamos la opción `--healthcheck-interval=0` para deshabilitar realmente el intervalo de ejecución y hacer que la comprobación de estado sea manual.

Ten en cuenta que el ejemplo anterior solo es necesario cuando se especifica `--health-interval=0`, lo que evita que Podman programe automáticamente la comprobación de estado. En el uso en el mundo real, Podman ejecutará la comprobación de estado automáticamente y el usuario nunca tendrá que ejecutar el comando `podman healthcheck run`.

Podman utiliza temporizadores de systemd (*systemd timers*) para programar comprobaciones de estado. Por esta razón, es obligatorio si queremos programar una comprobación de estado para nuestros contenedores. Por supuesto, si algunos de nuestros sistemas no utilizan systemd como administrador de demonios predeterminado, podríamos usar diferentes herramientas, como cron, para programar las comprobaciones de estado, pero estas deberían configurarse manualmente.

Inspeccionemos cómo funciona esta integración automática con systemd creando una comprobación de estado con un intervalo:

```bash
$ podman run -dt --name healthtest2 --health-cmd 'curl http://localhost || exit 1' --healthcheck-interval=10s quay.io/libpod/alpine_nginx:latest
69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6
$ podman ps
CONTAINER ID  IMAGE                              COMMAND               CREATED        STATUS                PORTS     NAMES
69f3ca0ce7aa  quay.io/libpod/alpine_nginx:latest  nginx -g daemon o...  4 seconds ago  Up 4 seconds (healthy) 80/tcp    healthtest2 
```

Como podemos ver en el bloque de código anterior, acabamos de iniciar un nuevo contenedor llamado `healthtest2`, definiendo el mismo `health-cmd` del ejemplo anterior pero ahora especificando la opción `--health-interval=10s` para programar la comprobación cada 10 segundos.

Después del comando `podman run`, también ejecutamos el comando `podman ps` para inspeccionar si la comprobación de estado está funcionando correctamente y, como podemos ver en la salida, tenemos el estado `healthy` para nuestro nuevo contenedor.

Pero, ¿cómo funciona esta integración? Tomemos el ID del contenedor y busquémoslo en el siguiente directorio:

```bash
$ ls /run/user/$UID/systemd/transient/69f3*
/run/user/1000/systemd/transient/69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6-6433a92c3f3c8479.service
/run/user/1000/systemd/transient/69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6-6433a92c3f3c8479.timer
```

El directorio que se muestra en el ejemplo anterior contiene todos los recursos de systemd en uso para nuestro usuario actual. En particular, miramos dentro del directorio `transient`, que contiene archivos de unidad temporales para nuestro usuario actual.

Cuando iniciamos un contenedor con una comprobación de estado y un intervalo de programación, Podman realizará una configuración transitoria de un servicio systemd y un archivo de unidad de temporizador. Esto significa que estos archivos de unidad no son permanentes y se pueden perder al reiniciar, pero se recrean la próxima vez que se reinicia el contenedor.

Inspeccionemos lo que se define dentro de estos archivos:

```ini
$ cat /run/user/1000/systemd/transient/69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6-6433a92c3f3c8479.service
# This is a transient unit file, created programmatically via the systemd API. Do not edit.
[Unit]
Description=/usr/bin/podman healthcheck run 69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6

[Service]
LogLevelMax=5
Environment="PATH=/home/vagrant/.local/bin:/home/vagrant/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin"
ExecStart=
ExecStart="/usr/bin/podman" "healthcheck" "run" "69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6"
```

```ini
$ cat /run/user/1000/systemd/transient/69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6-6433a92c3f3c8479.timer
# This is a transient unit file, created programmatically via the systemd API. Do not edit.
[Unit]
Description=/usr/bin/podman healthcheck run 69f3ca0ce7aa96401363db07daefe29184a1ede49f242cc4d3f8ab13a50ac5f6

[Timer]
OnUnitInactiveSec=10s
AccuracySec=1s
```

Como podemos ver en el ejemplo anterior, el archivo de unidad de servicio contiene el comando healthcheck de Podman, mientras que el archivo de unidad de temporizador define el intervalo de programación.

Finalmente, simplemente porque quizás queramos una forma rápida de identificar contenedores saludables o no saludables, podemos usar el siguiente comando para mostrarlos rápidamente:

```bash
$ podman ps -a --filter health=healthy
CONTAINER ID  IMAGE                              COMMAND               CREATED         STATUS                  PORTS     NAMES
6a98d7bc448d  quay.io/libpod/alpine_nginx:latest  nginx -g daemon o...  18 seconds ago  Up 18 seconds (healthy) 80/tcp    healthtest1
05678f1bf846  quay.io/libpod/alpine_nginx:latest  nginx -g daemon o...  5 seconds ago   Up 4 seconds (healthy)  80/tcp    healthtest2
```

En este ejemplo, utilizamos la opción `--filter health=healthy` para mostrar solo los contenedores saludables con el comando `podman ps`.

De forma predeterminada, cuando la comprobación de estado de un contenedor falla repetidamente, Podman simplemente marca el contenedor como no saludable (*unhealthy*). Si bien esto es útil para herramientas de monitorización externas, no resuelve el problema de un servicio detenido. Para cerrar esta brecha, Podman proporciona el indicador `--health-on-failure`, lo que permite que el motor de contenedores tome medidas inmediatas y automatizadas cuando falla una comprobación de estado.

Este indicador transforma una comprobación de estado pasiva en un mecanismo activo de autorrecuperación (*self-healing*). Puedes elegir entre varias estrategias diferentes, dependiendo de cómo desees que se maneje la recuperación:

- `none` (predeterminado): Podman solo actualiza el estado de salud a `unhealthy` y no realiza ninguna otra acción.
- `restart`: Podman reiniciará automáticamente el contenedor. Este es el enfoque de autorrecuperación más común para servicios que ocasionalmente pueden bloquearse o sufrir fugas de recursos.
- `stop`: El contenedor simplemente se detiene. Esto es útil en escenarios donde un servicio con fallos podría dañar los datos si se le permite seguir ejecutándose.
- `kill`: Podman envía una señal `SIGKILL` para finalizar inmediatamente el proceso del contenedor.

Para ejecutar un servidor web que se reinicie automáticamente si deja de responder, combinarías tu comando de comprobación de estado con la política de reinicio:

```bash
$ podman run -d \
  --name self_healing_web \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-on-failure=restart \
  nginx
```

El uso de este enfoque garantiza que tu servicio mantenga el máximo tiempo de actividad sin requerir la intervención manual de un administrador del sistema. Efectivamente, aporta una capacidad de mini-orquestador directamente a la CLI de Podman.

Si bien definir comprobaciones de estado en tiempo de ejecución a través de la CLI es flexible, la forma más sólida de garantizar que un servicio se monitorice correctamente es incrustar la comprobación de estado dentro de la propia imagen del contenedor. Al agregar una instrucción `HEALTHCHECK` a tu Containerfile (o Dockerfile), codificas la definición de salud directamente en los metadatos de la aplicación.

Este enfoque garantiza que cualquier persona que descargue tu imagen, ya sea un desarrollador en Fedora o un sistema automatizado en un clúster de producción, se beneficie de la misma lógica de monitorización sin tener que buscar los indicadores de CLI correctos.

Veamos un Containerfile que compila un servicio web simple utilizando la Universal Base Image (UBI) de Red Hat e incluye una comprobación de estado integrada:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-minimal
# Install a simple web server (microdnf is the package manager for ubi-minimal)
RUN microdnf install -y python3 && microdnf clean all
# Define the health check: try to fetch the local index every 10 seconds
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s \
  --retries=3 CMD curl -f http://localhost:8080/ || exit 1
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
```

Hemos aprendido a solucionar problemas y monitorizar nuestros contenedores hasta ahora, pero ¿qué pasa con el proceso de compilación de contenedores? Descubramos más sobre la inspección de compilación de contenedores en la siguiente sección.

---

### Inspección de los resultados de compilación de nuestros contenedores

En el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/6), *Conoce Buildah – Construcción de Contenedores desde Cero*, discutimos en detalle el proceso de compilación de contenedores y aprendimos a crear imágenes personalizadas utilizando Dockerfiles/Containerfiles o comandos nativos de Buildah. También ilustramos cómo el segundo enfoque ayuda a lograr un mayor grado de control sobre el flujo de trabajo de compilación.

Esta sección ayuda a proporcionar algunas de las mejores prácticas para inspeccionar los resultados de la compilación y comprender los problemas potencialmente relacionados.

#### Resolución de problemas de compilaciones a partir de Dockerfiles

Cuando se utiliza Podman o Buildah para ejecutar una compilación basada en un Dockerfile/Containerfile, el proceso de compilación imprime todas las salidas de las instrucciones y los errores relacionados en el stdout de la terminal. Para todas las instrucciones `RUN`, los errores generados por los comandos ejecutados se propagan y se imprimen con fines de depuración.

Intentemos ahora probar algunos posibles problemas de compilación. Esta no es una lista exhaustiva de errores; el propósito es proporcionar un método para analizar la causa raíz.

El primer ejemplo muestra una compilación mínima donde una instrucción `RUN` falla debido a un error en el comando ejecutado. Los errores en las instrucciones `RUN` pueden cubrir una amplia gama de casos, pero la regla general es la siguiente: el comando ejecutado devuelve un código de salida y, si este no es cero, la compilación falla y el error, junto con el estado de salida, se imprime.

En el siguiente ejemplo, usamos el comando `yum` para instalar el paquete `httpd`, pero intencionalmente hemos cometido un error tipográfico en el nombre del paquete para generar un error. Aquí está la transcripción del Dockerfile (`Chapter11/RUN_command_error/Dockerfile`):

```dockerfile
FROM registry.access.redhat.com/ubi8
# Update image and install httpd
RUN yum install -y htpd && yum clean all -y
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Si intentamos ejecutar el comando, obtendremos un error generado por el comando `yum` que no puede encontrar el paquete `htpd` faltante:

```text
$ buildah build -t custom_httpd .
STEP 1/4: FROM registry.access.redhat.com/ubi8
STEP 2/4: RUN yum install -y htpd && yum clean all -y
Updating Subscription Management repositories.
Unable to read consumer identity
This system is not registered with an entitlement server. You can use subscription-manager to register.
Red Hat Universal Base Image 8 (RPMs) - BaseOS   3.9 MB/s | 796 kB     00:00    
Red Hat Universal Base Image 8 (RPMs) - AppStre  6.2 MB/s | 2.6 MB     00:00    
Red Hat Universal Base Image 8 (RPMs) - CodeRea  171 kB/s |  16 kB     00:00    
No match for argument: htpd
Error: Unable to find a match: htpd
error building at STEP "RUN yum install -y htpd && yum clean all -y": error while running runtime: exit status 1
```

Las dos primeras líneas imprimen el mensaje de error generado por el comando `yum`, como en un entorno de línea de comandos estándar.

A continuación, Buildah (y, de la misma manera, Podman) produce un mensaje para informarnos sobre el paso que generó el error. Este mensaje se administra en el paquete `imagebuildah` mediante el ejecutor de etapas (*stage executor*), que maneja, como su nombre indica, la ejecución de las etapas de compilación y sus estados. El código fuente se puede inspeccionar en el repositorio de Buildah en GitHub: [https://github.com/containers/buildah/blob/main/imagebuildah/stage_executor.go](https://github.com/containers/buildah/blob/main/imagebuildah/stage_executor.go).

El mensaje incluye la instrucción Dockerfile y el error generado, junto con el estado de salida.

La última línea incluye el error y el estado de salida final, 1, relacionado con la ejecución del comando `buildah`.

**Solución**: Utiliza el mensaje de error para encontrar la instrucción `RUN` que contiene el comando que falla y corrige o soluciona el error del comando.

Otra razón de fallo muy común en las compilaciones es la falta de la imagen principal (*parent image*). Podría estar relacionada con un nombre de repositorio mal escrito, una etiqueta faltante o un registro inaccesible.

El siguiente ejemplo muestra otra variación del Dockerfile anterior, donde el nombre del repositorio de imágenes está mal escrito y, por lo tanto, no existe en el registro remoto (`Chapter11/FROM_repo_not_found/Dockerfile`):

```dockerfile
FROM registry.access.redhat.com/ubi_8
# Update image and install httpd
RUN yum install -y httpd && yum clean all -y
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Al ejecutar una compilación desde este Dockerfile, nos encontraremos con un error causado por la falta del repositorio de imágenes, como en el siguiente ejemplo:

```text
$ buildah build -t custom_httpd .
STEP 1/4: FROM registry.access.redhat.com/ubi_8
Trying to pull registry.access.redhat.com/ubi_8:latest...
error creating build container: initializing source docker://registry.access.redhat.com/ubi_8:latest: reading manifest latest in registry.access.redhat.com/ubi_8: name unknown: Repo not found
```

La última línea produce un error diferente. Es un error muy fácil de solucionar y solo requiere pasar un repositorio válido a la instrucción `FROM`.

**Solución**: Corrige el nombre del repositorio y reinicia el proceso de compilación. Alternativamente, verifica que el registro de destino contenga el repositorio deseado.

¿Qué pasa si escribimos mal la etiqueta de la imagen? El siguiente fragmento de Dockerfile muestra una etiqueta no válida para la imagen oficial de Fedora (`Chapter11/FROM_tag_not_found/Dockerfile`):

```dockerfile
FROM docker.io/library/fedora:sometag
# Update image and install httpd
RUN dnf install -y httpd && dnf clean all -y
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Esta vez, cuando compilemos la imagen, obtendremos un error 404 producido por el registro, que no puede encontrar un manifiesto asociado para la etiqueta `sometag`:

```text
$ buildah build -t custom_httpd .
STEP 1/4: FROM docker.io/library/fedora:sometag
Trying to pull docker.io/library/fedora:sometag...
error creating build container: initializing source docker://fedora:sometag: reading manifest sometag in docker.io/library/fedora: manifest unknown: manifest unknown
```

La etiqueta faltante generará nuevamente un error que es fácil de solucionar, diciéndonos que no se puede encontrar el manifiesto `sometag`.

**Solución**: Encuentra una etiqueta válida para usar en el proceso de compilación. Usa `skopeo list-tags` para encontrar todas las etiquetas disponibles en un repositorio determinado, como se muestra en el siguiente ejemplo:

```json
$ skopeo list-tags docker://docker.io/library/fedora
{
    "Repository": "docker.io/library/fedora",
    "Tags": [
        ...omitted output...
        "39",
        "40",
        "branched",
        "heisenbug",
        "latest",
        "modular",
        "rawhide"
    ]
}
```

Ten en cuenta también que, a veces, el error detectado en la instrucción `FROM` se debe al intento de acceder a un registro privado sin autenticación. Este es un error muy común y simplemente requiere un paso de autenticación en el registro de destino antes de que se realice cualquier acción de compilación.

En el siguiente ejemplo, tenemos un Dockerfile que usa una imagen de un registro privado genérico que se ejecuta usando las API de Docker Registry v2 (`Chapter11/FROM_auth_error/Dockerfile`):

```dockerfile
FROM local-registry.example.com/ubi8
# Update image and install httpd
RUN yum install -y httpd && yum clean all -y
# Expose the default httpd port 80
EXPOSE 80
# Run the httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Intentemos compilar la imagen y ver qué sucede:

```text
$ buildah build -t test3 .
STEP 1/4: FROM local-registry.example.com/ubi8
Trying to pull local-registry.example.com/ubi8:latest...
error creating build container: initializing source docker://local-registry.example.com/ubi8:latest: reading manifest latest in local-registry.example.com/ubi8: unauthorized: authentication required
```

En este caso de uso, el error es muy claro. No estamos autorizados a descargar la imagen del registro de destino y, por lo tanto, debemos autenticarnos con un token de autenticación válido para acceder a ella.

**Solución**: Autentícate con `podman login` o `buildah login` en el registro para recuperar el token o proporciona un archivo de autenticación con un token válido.

Hasta ahora, hemos inspeccionado los errores generados por compilaciones con Dockerfiles. Veamos ahora el comportamiento de Buildah en el caso de errores al usar sus instrucciones de línea de comandos.

#### Resolución de problemas de compilaciones con comandos nativos de Buildah

Al ejecutar comandos de Buildah, es una práctica común colocarlos dentro de un script de shell o una canalización (*pipeline*).

En este ejemplo, usaremos Bash como intérprete. De forma predeterminada, Bash ejecuta el script hasta el final, independientemente de los errores intermedios. Este comportamiento puede generar errores inesperados si falla una instrucción de Buildah dentro del script. Por esta razón, la mejor práctica es agregar el siguiente comando al principio del script:

```bash
set -euo pipefail
```

La configuración resultante es una especie de red de seguridad que bloquea la ejecución del script tan pronto como encontramos un error y evita errores comunes, como variables no establecidas.

El comando `set` es una instrucción interna de Bash que configura el shell para la ejecución de scripts. La opción `-e` dentro de esta instrucción le dice al shell que salga inmediatamente si una canalización o un solo comando falla, y la opción `-o pipefail` le dice al shell que salga con el código de error del comando más a la derecha de una canalización fallida que produjo un código de salida distinto de cero. La opción `-u` le dice al shell que trate las variables y parámetros no establecidos como un error durante la expansión de parámetros. Esto nos mantiene a salvo de la expansión faltante de variables no establecidas.

El siguiente script incorpora la lógica de una compilación simple de un servidor httpd sobre la imagen de Fedora:

```bash
#!/bin/bash
set -euo pipefail

# Trying to pull a non-existing tag of Fedora official image
container=$(buildah from docker.io/library/fedora:non-existing-tag)
buildah run $container -- dnf install -y httpd; dnf clean all -y
buildah config --cmd "httpd -DFOREGROUND" $container
buildah config --port 80 $container
buildah commit $container custom-httpd
buildah tag custom-httpd registry.example.com/custom-httpd:v0.0.1
```

La etiqueta de la imagen se configuró mal a propósito. Veamos los resultados de la ejecución del script:

```text
$ ./custom-httpd.sh
Trying to pull docker.io/library/fedora:non-existing-tag...
initializing source docker://fedora:non-existing-tag: reading manifest non-existing-tag in docker.io/library/fedora: manifest unknown: manifest unknown
```

La compilación produce un error de `manifest unknown`, al igual que el intento similar con el Dockerfile.

A partir de esta salida, también podemos aprender que Buildah (y Podman, que utiliza bibliotecas de Buildah para su implementación de compilación) produce los mismos mensajes que una compilación estándar con un Dockerfile/Containerfile, con la única excepción de no mencionar el paso de compilación, lo cual es obvio ya que estamos ejecutando comandos libres dentro de un script.

**Solución**: Encuentra una etiqueta válida para usar en el proceso de compilación. Usa `skopeo list-tags` para encontrar todas las etiquetas disponibles en un repositorio determinado.

En esta sección, hemos aprendido a analizar y solucionar errores de compilación, pero ¿qué podemos hacer cuando los errores ocurren en tiempo de ejecución dentro del contenedor y no tenemos las herramientas adecuadas para solucionar problemas dentro de la imagen? Para este propósito, tenemos una herramienta nativa de Linux que puede considerarse la verdadera navaja suiza de los espacios de nombres: `nsenter`.

---

### Resolución avanzada de problemas con nsenter

Comencemos con una frase dramática: la resolución de problemas en tiempo de ejecución a veces puede ser compleja.

Además, comprender y solucionar problemas de tiempo de ejecución dentro de un contenedor implica comprender cómo funcionan los contenedores en GNU/Linux. Explicamos estos conceptos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), *Introducción a la Tecnología de Contenedores*.

A veces, la resolución de problemas puede ser muy fácil y, como se indicó en las secciones anteriores, el uso de comandos básicos, como `podman logs`, `podman inspect` y `podman exec`, junto con el uso de comprobaciones de estado personalizadas, puede ayudarnos a obtener acceso a la información necesaria para completar nuestro análisis con éxito.

Las imágenes hoy en día tienden a ser lo más pequeñas posible. ¿Qué sucede cuando necesitamos herramientas de resolución de problemas más especializadas y no están disponibles dentro de la imagen? Podrías pensar en ejecutar un proceso de shell dentro del contenedor e instalar la herramienta que falta, pero a veces (y este es un patrón de seguridad creciente), los administradores de paquetes no están disponibles dentro de las imágenes de contenedor, ¡a veces ni siquiera los comandos `curl` o `wget`!

Podemos sentirnos un poco perdidos, pero debemos recordar que los contenedores son procesos ejecutados dentro de namespaces y cgroups dedicados. ¿Qué pasaría si tuviéramos una herramienta que pudiera permitirnos ejecutar comandos dentro de uno o más namespaces mientras mantenemos nuestro acceso a las herramientas del host? Esa herramienta existe y se llama `nsenter` (accede a la página del manual con `man nsenter`). No está afiliada a ningún motor o tiempo de ejecución de contenedores y proporciona una forma sencilla de ejecutar comandos dentro de uno o varios espacios de nombres no compartidos para un proceso (el proceso principal del contenedor).

Antes de sumergirnos en ejemplos reales, analicemos las principales opciones y argumentos de `nsenter` ejecutándolo con la opción `--help`:

```text
$ nsenter --help

Usage:
 nsenter [options] [<program> [<argument>...]]

Run a program with namespaces of other processes.

Options:
 -a, --all                      enter all namespaces
 -t, --target <pid>             target process to get namespaces from
 -m, --mount[=<file>]           enter mount namespace
 -u, --uts[=<file>]             enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]             enter System V IPC namespace
 -n, --net[=<file>]             enter network namespace
 -p, --pid[=<file>]             enter pid namespace
 -C, --cgroup[=<file>]          enter cgroup namespace
 -U, --user[=<file>]            enter user namespace
 -T, --time[=<file>]            enter time namespace
 -S, --setuid <uid>             set uid in entered namespace
 -G, --setgid <gid>             set gid in entered namespace
     --preserve-credentials     do not touch uids or gids
 -r, --root[=<dir>]             set the root directory
 -w, --wd[=<dir>]               set the working directory
 -F, --no-fork                  do not fork before exec'ing <program>
 -Z, --follow-context           set SELinux context according to --target PID

 -h, --help                     display this help
 -V, --version                  display version

For more details see nsenter(1).
```

A partir del resultado de este comando, es fácil detectar que hay tantas opciones como la cantidad de namespaces disponibles.

Gracias a `nsenter`, podemos capturar el PID del proceso principal de un contenedor y luego ejecutar comandos (incluido un shell) dentro de los namespaces relacionados.

Para extraer el PID principal del contenedor, podemos usar el siguiente comando:

```bash
$ podman inspect <Container_Name> --format '{{ .State.Pid }}'
```

La salida se puede insertar dentro de una variable para un acceso más fácil:

```bash
$ CNT_PID=$(podman inspect <Container_Name> \
  --format '{{ .State.Pid }}')
```

> [!TIP]
> Todos los namespaces asociados con un proceso se representan dentro del directorio `/proc/[pid]/ns`. Este directorio contiene una serie de enlaces simbólicos que se asignan a un tipo de namespace y su número de inodo correspondiente. En esencia, un inodo es una estructura de datos que almacena información crucial sobre un archivo o directorio, mientras que el número de inodo es un número entero único que identifica ese inodo específico dentro de un sistema de archivos determinado.
>
> El siguiente comando muestra los namespaces asociados con el proceso ejecutado por el contenedor: `ls -al /proc/$CNT_PID/ns`.

Vamos a aprender a usar `nsenter` con un ejemplo práctico. En la siguiente subsección, intentaremos solucionar los problemas de red de una aplicación cliente de base de datos que devuelve un error interno del servidor HTTP sin mencionar ninguna información útil en los registros de la aplicación.

#### Resolución de problemas de un cliente de base de datos con nsenter

No es raro trabajar en aplicaciones alfa que aún no tienen el registro implementado correctamente o que tienen un manejo deficiente de los mensajes de registro.

El siguiente ejemplo es una aplicación web que extrae campos de una base de datos Postgres e imprime un objeto JSON con todas las ocurrencias. La verbosidad de los registros de la aplicación se ha dejado intencionalmente al mínimo y no se producen errores de conexión o consulta.

Considera que, con este último ejemplo, estamos simulando un flujo de trabajo real para un desarrollador de aplicaciones que intenta aprovechar una conexión de base de datos con la aplicación recién desarrollada.

Por razones de espacio, no imprimiremos el código fuente de la aplicación en el libro; sin embargo, está disponible en la siguiente URL para su inspección: [https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition/tree/main/Chapter11/students](https://github.com/PacktPublishing/Podman-for-DevOps-Second-Edition/tree/main/Chapter11/students).

La carpeta también contiene un script SQL para poblar una base de datos de muestra. La aplicación se compila utilizando el siguiente Dockerfile (`Chapter11/students/Dockerfile`):

```dockerfile
FROM docker.io/library/golang AS builder
# Copy files for build
RUN mkdir -p /go/src/students/models
COPY go.mod main.go /go/src/students
COPY models/main.go /go/src/students/models
# Set the working directory
WORKDIR /go/src/students
# Download dependencies
RUN go get -d -v ./...
# Install the package
RUN go build -v
# Runtime image
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest as bin
COPY --from=builder /go/src/students /usr/local/bin
COPY entrypoint.sh /
EXPOSE 8080
ENTRYPOINT ["/entrypoint.sh"]
```

Como de costumbre, vamos a compilar el contenedor con Buildah:

```bash
# buildah build -t students-image .
```

> [!IMPORTANT]
> Si bien estamos compilando y ejecutando este contenedor en modo rootful para explorar la segregación tradicional de namespaces de red, es importante distinguir esto de cómo Podman maneja las redes rootless.
>
> Incluso en modo rootless, los contenedores permanecen totalmente segregados dentro de sus propios namespaces de red. La diferencia clave radica en el controlador. El valor predeterminado actual, `pasta`, proporciona una capa de traducción de alto rendimiento entre la interfaz de capa 2 del contenedor y los sockets nativos de capa 4 del host. Esto permite un aislamiento y una seguridad de red fluidos sin necesidad de privilegios elevados ni configuraciones de IP complejas en el host. Exploraremos la mecánica de `pasta` y otras características avanzadas de red en el [Capítulo 12](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/12).

El contenedor acepta un conjunto de indicadores personalizados para definir la base de datos, el host, el puerto y las credenciales. Para ver la ayuda, simplemente ejecuta el siguiente comando:

```text
# podman run students-image students -help
%!(EXTRA string=students)
  -database string
    	Default application database (default "students")
  -host string
    	Default host running the database (default "localhost")
  -password string
    	Default database password (default "password"
  -port string
    	Default database port (default "5432")
  -username string
    	Default database username (default "admin")
```

Se nos ha informado que la base de datos se está ejecutando en el host `pghost.example.com` en el puerto `5432`, con el nombre de usuario `students` y la contraseña `Podman_R0cks#`.

El siguiente comando ejecuta la aplicación web `students` con los argumentos personalizados:

```bash
# podman run --rm -d -p 8080:8080 \
  --name students_app students-image \
  students -host pghost.example.com \
  -port 5432 \
  -username students \
  -password Podman_R0cks#
```

El contenedor se inicia correctamente y el único mensaje de registro impreso es el siguiente:

```text
# podman logs students_app
2021/12/27 21:51:31 Connecting to host pghost.example.com:5432, database students
```

Finalmente podemos probar la aplicación y ver qué sucede cuando ejecutamos una consulta:

```bash
$ curl localhost:8080/students
Internal Server Error
```

La aplicación puede tardar algún tiempo en responder, pero después de un tiempo, imprimirá un mensaje HTTP de error interno del servidor (500). Encontraremos la razón en los siguientes párrafos. Los registros no son útiles ya que no se imprime nada más que el primer mensaje de inicio. Además, el contenedor se creó con la imagen mínima de UBI, que tiene una huella pequeña de archivos binarios preinstalados y no tiene utilidades para solucionar problemas. Podemos usar `nsenter` para inspeccionar el comportamiento del contenedor, especialmente desde el punto de vista de la red, adjuntando nuestro programa de shell actual al namespace de red del contenedor mientras mantenemos el acceso a nuestros binarios del host.

En un nuevo shell, podemos averiguar el PID del proceso principal y completar una variable con su valor (observa el comando `sudo` para ejecutar la inspección con privilegios elevados para el contenedor rootful en ejecución):

```bash
$ CNT_PID=$(sudo podman inspect students_app --format '{{ .State.Pid }}')
```

El siguiente ejemplo ejecuta Bash en el namespace de red del contenedor, mientras conserva todos los demás namespaces del host (nuevamente, observa el comando `sudo` para ejecutar `nsenter` con privilegios elevados):

```bash
$ sudo nsenter -t $CNT_PID -n /bin/bash
```

> [!NOTE]
> Es posible ejecutar cualquier binario de host directamente desde `nsenter`. Un comando como el siguiente es perfectamente legítimo:
>
> ```bash
> $ sudo nsenter -t $CNT_PID -n ip addr show
> ```

Para demostrar que realmente estamos ejecutando un shell adjunto al namespace de red del contenedor, podemos iniciar el comando `ip addr show`:

```text
# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: tap0: <BROADCAST,UP,LOWER_UP> mtu 65520 qdisc fq_codel state UNKNOWN group default qlen 1000
    link/ether fa:0b:50:ed:9d:37 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.100/24 brd 10.0.2.255 scope global tap0
       valid_lft forever preferred_lft forever
    inet6 fe80::f80b:50ff:feed:9d37/64 scope link 
       valid_lft forever preferred_lft forever
# ip route
default via 10.0.2.2 dev tap0 
10.0.2.0/24 dev tap0 proto kernel scope link src 10.0.2.100
```

El primer comando, `ip addr show`, imprime la configuración de IP, con una interfaz `tap0` básica conectada al host y la interfaz de bucle invertido (*loopback*).

El segundo comando, `ip route`, muestra la tabla de enrutamiento predeterminada dentro del namespace de red del contenedor.

Podemos echar un primer vistazo a las conexiones activas utilizando la herramienta `ss`, ya disponible en nuestro host Fedora:

```text
# ss -atunp
Netid State      Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess                                                                      
tcp   TIME-WAIT  0      0       10.0.2.100:50728     10.0.2.100:8080                                                                            
tcp   LISTEN     0      128              *:8080               *:*     users:(("students",pid=402788,fd=3))
```

Detectamos de inmediato que no hay conexiones establecidas entre la aplicación y el host de la base de datos, lo que nos indica que el problema probablemente esté relacionado con el enrutamiento, las reglas de firewall o los problemas de resolución de nombres que nos impiden llegar al host correctamente.

El siguiente paso es intentar conectarse manualmente a la base de datos con la herramienta cliente `psql`, disponible en el paquete rpm `postgresql`:

```text
# psql -h pghost.example.com
psql: error: could not translate host name "pghost.example.com" to address: Name or service not known
```

Este mensaje es bastante claro: el servicio DNS no resuelve el host y hace que la aplicación falle. Para confirmarlo finalmente, podemos ejecutar el comando `dig`, que devuelve un error `NXDOMAIN`, un mensaje típico de un servidor DNS para indicar que el dominio no se puede resolver y no existe:

```text
# dig pghost.example.com

; <<>> DiG 9.16.23-RH <<>> pghost.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 40669
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;pghost.example.com.		IN	A

;; Query time: 0 msec
;; SERVER: 192.168.200.1#53(192.168.200.1)
;; WHEN: Mon Dec 27 23:26:47 CET 2021
;; MSG SIZE  rcvd: 47
```

Después de consultar con el equipo de desarrollo, descubrimos que al nombre de la base de datos le faltaba un guión, por lo que estaba mal escrito, y el nombre correcto era `pg-host.example.com`. Ahora podemos solucionar el problema ejecutando el contenedor con el nombre correcto.

Ahora esperamos ver los resultados correctos al iniciar la consulta nuevamente:

```bash
$ curl localhost:8080/students
{"Id":10149,"FirstName":"Frank","MiddleName":"Vincent","LastName":"Zappa","Class":"3A","Course":"Composition"}
```

En este ejemplo, nos hemos centrado en la resolución de problemas del namespace de red, pero es posible adjuntar nuestro programa de shell actual a múltiples namespaces simplemente agregando los indicadores relacionados.

También podemos simular `podman exec` ejecutando el comando con la opción `-a`:

```bash
$ sudo nsenter -t $CNT_PID -a /bin/bash
```

Este comando adjunta el proceso a todos los namespaces no compartidos, incluido el namespace de montaje, lo que brinda la misma vista del árbol del sistema de archivos que ven los procesos dentro del contenedor.

Si bien `podman exec` es la forma estándar de ejecutar un proceso dentro de un contenedor existente, tiene una limitación importante: solo puede ejecutar binarios que ya existen dentro de la imagen del contenedor.

En un entorno de producción, es probable que utilices imágenes mínimas, como `ubi-micro` o `ubi-minimal` de Red Hat, para reducir la superficie de ataque y el tamaño de la imagen. Estas imágenes a menudo carecen de herramientas básicas de resolución de problemas como `ip`, `ps`, `tcpdump` o incluso un shell como `bash`. Si intentas usar `podman exec` para solucionar problemas en dicho contenedor, te encontrarás con un error porque la herramienta que necesitas simplemente no está allí.

---

### Resumen

En este capítulo, nos centramos en la resolución de problemas de contenedores, intentando proporcionar un conjunto de mejores prácticas y herramientas para encontrar y corregir problemas dentro de un contenedor en tiempo de compilación o en tiempo de ejecución.

Comenzamos mostrando algunos casos de uso comunes durante las etapas de ejecución y compilación de contenedores, y sus soluciones relacionadas.

Posteriormente, presentamos el concepto de comprobaciones de estado (*health checks*) e ilustramos cómo implementar sondas sólidas en contenedores para monitorizar sus estados, al tiempo que mostramos los conceptos arquitectónicos detrás de ellas.

En la tercera sección, aprendimos sobre una serie de escenarios de error comunes relacionados con las compilaciones y mostramos cómo resolverlos rápidamente.

En la sección final, presentamos el comando `nsenter` y simulamos una aplicación frontend web que necesitaba resolución de problemas de red para descubrir la causa de un error interno del servidor. Gracias a este ejemplo, aprendimos a realizar una resolución avanzada de problemas dentro de los namespaces del contenedor.

Ahora que hemos profundizado en las técnicas de resolución de problemas para contenedores, estamos listos para pasar al siguiente capítulo, donde veremos de forma avanzada las redes para contenedores.

---

### Lecturas adicionales

- Directrices de resolución de problemas de Podman: [https://github.com/containers/podman/blob/main/troubleshooting.md](https://github.com/containers/podman/blob/main/troubleshooting.md)
