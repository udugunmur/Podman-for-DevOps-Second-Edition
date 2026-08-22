# Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

## Capítulo 1: Introducción a la Tecnología de Contenedores

La tecnología de contenedores tiene raíces antiguas en la historia de los sistemas operativos. Por ejemplo, ¿sabías que parte de la tecnología de contenedores nació en la década de 1970? A pesar de su enfoque simple e intuitivo, existen muchos conceptos detrás de los contenedores que merecen un análisis más profundo para comprender y apreciar plenamente cómo se abrieron camino en la industria de TI.

Exploraremos esta tecnología para comprender mejor cómo funciona bajo el capó, la teoría detrás de ella y sus conceptos básicos. Conocer la mecánica y la tecnología detrás de las herramientas te permitirá abordar y aprender fácilmente los conceptos clave de toda la tecnología.

Luego, también exploraremos el propósito de la tecnología de contenedores y por qué se ha extendido a todas las empresas hoy en día. ¿Sabías que el 66% de las organizaciones del mundo ejecutan la mitad de su base de aplicaciones como contenedores en producción en la actualidad? [1]

¡Sumerjámonos en esta gran tecnología!

En este capítulo, plantearemos las siguientes preguntas:

- ¿Qué son los contenedores?
- ¿Por qué necesito un contenedor?
- ¿De dónde provienen los contenedores?
- ¿Dónde se utilizan los contenedores hoy en día?

---

### Requisitos técnicos

Este capítulo no requiere ningún prerrequisito técnico, ¡así que siéntete libre de leerlo sin preocuparte por instalar o configurar ningún tipo de software en tu estación de trabajo!

Si eres nuevo en los contenedores, encontrarás aquí muchos conceptos técnicos útiles para comprender los próximos capítulos. Recomendamos leer este capítulo con atención y volver a él cuando sea necesario. Un conocimiento previo del sistema operativo Linux resultará útil para comprender los conceptos técnicos tratados en este libro.

---

### Convenciones del libro

En los siguientes capítulos, aprenderemos sobre muchos conceptos nuevos, con ejemplos prácticos que requerirán una interacción activa con un entorno de shell de Linux. En los ejemplos prácticos, utilizaremos las siguientes convenciones:

- Para cualquier comando de shell que esté precedido por el carácter `$`, utilizaremos un usuario estándar (no root) para el sistema Linux.
- Para cualquier comando de shell que esté precedido por el carácter `#`, utilizaremos el usuario root para el sistema Linux.
- Cualquier salida o comando de shell que sea demasiado largo para mostrarse en una sola línea en el bloque de código se interrumpirá con el carácter `\`, y luego continuará en una nueva línea.

---

### ¿Qué son los contenedores?

Esta sección describe la tecnología de contenedores desde cero, comenzando con conceptos básicos como procesos, sistemas de archivos, llamadas al sistema (*system calls*) y aislamiento de procesos, hasta llegar a los motores y entornos de ejecución de contenedores (*engines* y *runtimes*). El propósito de esta sección es describir cómo los contenedores implementan el aislamiento de procesos. También describimos qué diferencia a los contenedores de las máquinas virtuales (VM) y destacamos el mejor caso de uso para ambos escenarios.

Antes de preguntarnos qué es un contenedor, deberíamos responder a otra pregunta: ¿qué es un proceso?

Según *The Linux Programming Interface*, un magnífico libro [2] de Michael Kerrisk, un proceso es una instancia de un programa en ejecución. Un programa es un archivo que contiene la información necesaria para ejecutar el proceso. Un programa puede estar vinculado dinámicamente a bibliotecas externas o puede estar vinculado estáticamente en el propio programa (el lenguaje de programación Go utiliza este enfoque por defecto).

Esto nos lleva a un concepto importante: un proceso se ejecuta en la CPU de la máquina y asigna una porción de memoria que contiene el código del programa y las variables utilizadas por el propio código. El proceso se instancia en el espacio de usuario (*user space*) de la máquina y su ejecución es orquestada por el kernel del sistema operativo. Cuando se ejecuta un proceso, este necesita acceder a diferentes recursos de la máquina, como E/S (disco, red, terminales, etc.) o memoria. Cuando el proceso necesita acceder a esos recursos, realiza una llamada al sistema (*system call*) en el espacio del kernel (*kernel space*) (por ejemplo, para leer un bloque de disco o enviar paquetes a través de la interfaz de red).

El proceso interactúa indirectamente con los discos del host mediante un sistema de archivos, una abstracción de almacenamiento multicapa que facilita el acceso de lectura y escritura a archivos y directorios.

¿Cuántos procesos se ejecutan normalmente en una máquina? Muchos. Son orquestados por el kernel del sistema operativo mediante una lógica de planificación compleja que hace que los procesos se comporten como si se estuvieran ejecutando en un núcleo de CPU dedicado, mientras que en realidad se comparte entre muchos de ellos.

El mismo programa puede instanciar muchos procesos de su tipo (por ejemplo, múltiples instancias de un servidor web ejecutándose en la misma máquina). Los conflictos, como que varios procesos intenten acceder al mismo puerto de red, deben gestionarse adecuadamente.

Nada nos impide ejecutar una versión diferente del mismo programa en el host, asumiendo que los administradores de sistemas tendrán la carga de gestionar los posibles conflictos de binarios, bibliotecas y sus dependencias. Esto podría convertirse en una tarea compleja, que no siempre es fácil de resolver con las prácticas habituales.

Esta breve introducción era necesaria para establecer el contexto.

Los contenedores son una respuesta simple e inteligente a la necesidad de ejecutar instancias de procesos aisladas. Podemos afirmar con seguridad que los contenedores son una forma de aislamiento de aplicaciones que funciona en muchos niveles:

- **Aislamiento del sistema de archivos**: Los procesos en contenedores tienen una vista independiente del sistema de archivos y sus programas se ejecutan desde el propio sistema de archivos aislado.
- **Aislamiento del ID de proceso (PID)**: El proceso contenedorizado se ejecuta bajo un conjunto independiente de identificadores de proceso (PID).
- **Aislamiento de usuarios**: Los identificadores de usuario (UID) y de grupo (GID) están aislados dentro del contenedor. El UID y GID de un proceso pueden ser diferentes dentro de un contenedor y ejecutarse con un UID o GID privilegiado únicamente dentro del contenedor.
- **Aislamiento de red**: Este tipo de aislamiento se relaciona con los recursos de red del host, como dispositivos de red, pilas IPv4 e IPv6, tablas de enrutamiento y reglas de firewall.
- **Aislamiento de IPC**: Los contenedores proporcionan aislamiento para los recursos de comunicación entre procesos (IPC) del host, como las colas de mensajes POSIX o los objetos System V IPC.
- **Aislamiento del uso de recursos**: Los contenedores se basan en los grupos de control de Linux (*control groups* o *cgroups*) para limitar o monitorizar el uso de ciertos recursos, como CPU, memoria o disco. Hablaremos más sobre cgroups más adelante en este capítulo.

Desde el punto de vista de la adopción, el propósito principal de los contenedores, o al menos el caso de uso más común, es ejecutar aplicaciones en entornos aislados. Para comprender mejor este concepto, podemos observar el siguiente diagrama:

*Figura 1.1 – Aplicaciones nativas frente a aplicaciones contenedorizadas*

Las aplicaciones que se ejecutan de forma nativa en un sistema que no proporciona funciones de contenedorización comparten los mismos binarios y bibliotecas, así como el mismo kernel, sistema de archivos, red y usuarios. Esto puede generar muchos problemas cuando se despliega una versión actualizada de una aplicación, especialmente conflictos de bibliotecas o dependencias no satisfechas.

Por otro lado, los contenedores ofrecen una capa consistente de aislamiento para las aplicaciones y sus dependencias asociadas, lo que garantiza una coexistencia fluida en el mismo host. Un nuevo despliegue consiste únicamente en la ejecución de la nueva versión en contenedor, ya que no interactuará ni entrará en conflicto con los demás contenedores o aplicaciones nativas.

Los contenedores de Linux están habilitados por diferentes características nativas del kernel, siendo la más importante los **namespaces de Linux**. Los namespaces abstraen recursos específicos del sistema (en particular, los descritos anteriormente, como la red, los puntos de montaje del sistema de archivos, los usuarios, etc.) y hacen que parezcan únicos para el proceso aislado. De esta manera, el proceso tiene la ilusión de interactuar con el recurso del host (por ejemplo, el sistema de archivos del host), mientras se expone una versión alternativa y aislada.

Actualmente, disponemos de un total de ocho tipos de namespaces:

- **Namespaces de montaje (*Mount namespaces*)**: Proporcionan aislamiento de la lista de puntos de montaje que ven los procesos en el namespace.
- **Namespaces de PID**: Aíslan el número de ID de proceso en un espacio separado, lo que permite que los procesos en diferentes namespaces de PID conserven el mismo PID.
- **Namespaces de usuario (*User namespaces*)**: Aíslan los UID y GID, el directorio raíz, los llaveros (*keyrings*) y las capacidades (*capabilities*). Esto permite que un proceso tenga un UID y GID privilegiado dentro del contenedor mientras simultáneamente tiene unos no privilegiados fuera del namespace.
- **Namespaces de UTS**: Permiten el aislamiento del nombre de host (*hostname*) y del nombre de dominio NIS.
- **Namespaces de red (*Network namespaces*)**: Permiten el aislamiento de los recursos de red del sistema, como dispositivos de red, pilas de protocolos IPv4 e IPv6, tablas de enrutamiento, reglas de firewall, números de puerto, etc. Los usuarios pueden crear dispositivos de red virtuales llamados pares *veth* para construir túneles entre namespaces de red.
- **Namespaces de IPC**: Aíslan recursos IPC como objetos System V IPC y colas de mensajes POSIX. Los objetos creados en un namespace IPC solo pueden ser accedidos por los procesos que son miembros de dicho namespace. Los procesos utilizan IPC para intercambiar datos, eventos y mensajes mediante un mecanismo cliente-servidor.
- **Namespaces de cgroup**: Aíslan directorios cgroup, proporcionando una vista virtualizada de los cgroups del proceso.
- **Namespaces de tiempo (*Time namespaces*)**: Proporcionan una vista aislada de la hora del sistema, permitiendo que los procesos en el namespace se ejecuten con un desfase de tiempo respecto a la hora del host.

Ahora, pasemos al uso de recursos.

#### Uso de recursos con cgroups

Los **cgroups** son una característica nativa del kernel de Linux cuyo propósito es organizar los procesos en un árbol jerárquico y limitar o monitorizar el uso de sus recursos.

La interfaz de cgroups del kernel, de manera similar a lo que ocurre con `/proc`, se expone a través de un pseudo-sistema de archivos `cgroupfs`. Este sistema de archivos generalmente se monta bajo `/sys/fs/cgroup` en el host.

Los cgroups ofrecen una serie de controladores (también llamados subsistemas) que se pueden usar para diferentes propósitos, como limitar la cuota de tiempo de CPU de un proceso, el uso de memoria, congelar y reanudar procesos, entre otros.

La jerarquía organizativa de los controladores ha cambiado a lo largo del tiempo, y actualmente existen dos versiones: v1 y v2. En cgroups v1, se pueden montar diferentes controladores en diferentes jerarquías. En cambio, cgroups v2 proporciona una jerarquía unificada de controladores, donde los procesos residen en los nodos hoja del árbol. Actualmente, cgroups v1 está en proceso de quedar obsoleto en favor de v2, pero algunas distribuciones aún lo admiten por compatibilidad con versiones anteriores.

Los contenedores utilizan los cgroups para limitar el uso de CPU o memoria. Por ejemplo, los usuarios pueden limitar la cuota de CPU (*CPU quota*), lo que significa limitar la cantidad de microsegundos que el contenedor puede usar la CPU durante un período determinado, o limitar los recursos compartidos de CPU (*CPU shares*), que representa la proporción ponderada de ciclos de CPU para cada contenedor.

Ahora que hemos ilustrado cómo funciona el aislamiento de procesos (tanto para namespaces como para recursos), podemos mostrar algunos ejemplos básicos.

#### Ejecución de procesos aislados

Un dato útil que conviene saber es que los sistemas operativos GNU/Linux ofrecen todas las características necesarias para ejecutar un contenedor de forma manual. Este resultado se puede lograr trabajando con llamadas al sistema específicas (en particular `unshare()` y `clone()`) y utilidades como el comando `unshare`.

Por ejemplo, para ejecutar un proceso, digamos `/bin/sh`, en un namespace de PID aislado, los usuarios pueden confiar en el comando `unshare`:

```bash
# unshare --fork --pid --mount-proc /bin/sh
```

El resultado es la ejecución de un nuevo proceso de shell en un namespace de PID aislado. Los usuarios pueden intentar monitorizar la vista de procesos y obtendrán una salida como la siguiente:

```bash
sh-5.0# ps aux USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND root 1 0.0 0.0 226164 4012 pts/4 S 22:56 0:00 /bin/sh root 4 0.0 0.0 227968 3484 pts/4 R+ 22:56 0:00 ps aux
```

Es interesante notar que el proceso de shell del ejemplo anterior se está ejecutando con PID 1, lo cual es correcto, ya que es el primer proceso que se ejecuta en el nuevo namespace aislado.

De todos modos, el namespace de PID será el único que se abstraiga, mientras que todos los demás recursos del sistema seguirán siendo los originales del host. Si queremos añadir más aislamiento, por ejemplo, en la pila de red, podemos añadir el flag `--net` al comando anterior:

```bash
# unshare --fork --net --pid --mount-proc /bin/sh
```

El resultado es un proceso de shell aislado tanto en el namespace de PID como en el de red. Los usuarios pueden inspeccionar la configuración de IP de la red y comprobar que el proceso aislado ya no ve directamente los dispositivos nativos del host:

```bash
sh-5.0# ip addr show 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Los ejemplos anteriores son útiles para comprender un concepto muy importante: los contenedores están estrechamente relacionados con las características nativas de Linux. El sistema operativo proporcionó una interfaz sólida y completa que ayudó al desarrollo de los runtimes de contenedores, y la capacidad de aislar namespaces y recursos fue la clave que desbloqueó la adopción de los contenedores. El papel del runtime de contenedores es abstraer la complejidad de los mecanismos de aislamiento subyacentes, siendo el aislamiento de puntos de montaje probablemente el más crucial de ellos. Por lo tanto, merece una mejor explicación.

#### Aislamiento de montajes

Hasta ahora hemos visto ejemplos de `unshare` que no afectaron los puntos de montaje ni la vista del sistema de archivos por parte del proceso. Para obtener el aislamiento del sistema de archivos que evite conflictos de binarios y bibliotecas, los usuarios necesitan crear otra capa de abstracción para los puntos de montaje expuestos.

Este resultado se logra aprovechando los namespaces de montaje (*mount namespaces*) y los montajes vinculados (*bind mounts*). Introducidos por primera vez en 2002 con el kernel de Linux 2.4.19, los namespaces de montaje aíslan la lista de puntos de montaje que ve el proceso. Cada namespace de montaje expone una lista discreta de puntos de montaje, lo que hace que los procesos en diferentes namespaces conozcan diferentes jerarquías de directorios.

Con esta técnica, es posible exponer al proceso en ejecución un árbol de directorios alternativo que contenga todos los binarios y bibliotecas necesarios según la elección.

A pesar de parecer una tarea sencilla, la gestión de un namespace de montaje dista mucho de ser simple y fácil de dominar. Por ejemplo, los usuarios tendrían que manejar diferentes versiones de archivos comprimidos de árboles de directorios de diferentes distribuciones, extraerlos y montarlos mediante bind mount en namespaces separados. Veremos más adelante que los primeros enfoques con contenedores en Linux siguieron esta metodología.

El éxito de los contenedores también está vinculado a un enfoque innovador, multicapa y de copia en escritura (*copy-on-write*) para la gestión de árboles de directorios, el cual introdujo un método simple y rápido de copiar, desplegar y utilizar el árbol necesario para ejecutar el contenedor: las **imágenes de contenedores**.

#### Imágenes de contenedores al rescate

Debemos agradecer a Docker la introducción de este método inteligente de almacenamiento de datos para contenedores. Posteriormente, las imágenes se convertirían en una especificación estándar de la Open Container Initiative (OCI) ([https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)).

Las imágenes se pueden ver como un paquete del sistema de archivos (*filesystem bundle*) que se descarga (*pull*) y se desempaqueta en el host antes de ejecutar el contenedor por primera vez.

Las imágenes se descargan desde repositorios llamados **registros de imágenes** (*image registries*). Estos repositorios pueden verse como almacenamiento de objetos especializado que alberga los datos de la imagen y los metadatos asociados. Existen tanto registros públicos y de uso gratuito (como `quay.io` o `docker.io`) como registros privados que pueden ejecutarse en la infraestructura privada del cliente, ya sea de forma local (*on-premises*) o en la nube.

Las imágenes pueden ser construidas por equipos de DevOps para satisfacer necesidades especiales o incrustar artefactos que deben desplegarse y ejecutarse en un host.

Durante el proceso de construcción de imágenes, los desarrolladores pueden inyectar artefactos preconstruidos o código fuente que se puede compilar en el propio contenedor de construcción. Para optimizar el tamaño de la imagen, es posible crear compilaciones multietapa (*multi-stage builds*) con una primera etapa que compila el código fuente utilizando una imagen base con los compiladores y runtimes necesarios, y una segunda etapa donde los artefactos construidos se inyectan en una imagen mínima y ligera, optimizada para un inicio rápido y un impacto mínimo en el almacenamiento.

La receta del proceso de construcción se define en un archivo de texto especial llamado `Dockerfile`, que define todos los pasos necesarios para ensamblar la imagen final.

Después de construirlas, los usuarios pueden enviar (*push*) sus propias imágenes a registros públicos o privados para su uso posterior o para despliegues orquestados y complejos.

El siguiente diagrama resume el flujo de trabajo de construcción:

*Figura 1.2 – Flujo de trabajo de construcción de imágenes*

Trataremos el tema de la construcción con más detalle más adelante en este libro.

¿Qué hace que una imagen de contenedor sea tan especial? La idea inteligente detrás de las imágenes es que pueden considerarse como una tecnología de empaquetado. Cuando los usuarios construyen su propia imagen con todos los binarios y dependencias instalados en el árbol de directorios del sistema operativo, están creando efectivamente un objeto autoconsistente que se puede desplegar en cualquier lugar sin más dependencias de software. Desde este punto de vista, las imágenes de contenedores son una respuesta a la clásica frase: *"Funciona en mi máquina"*.

Los equipos de desarrollo las adoran porque pueden estar seguros del entorno de ejecución de sus aplicaciones, y los equipos de operaciones las aprecian porque simplifican el proceso de despliegue al eliminar la tediosa tarea de mantener y actualizar las dependencias de bibliotecas de un servidor.

Otra característica inteligente de las imágenes de contenedores es su enfoque multicapa de copia en escritura (*copy-on-write*). En lugar de tener un único archivo binario monolítico, una imagen está compuesta por muchos archivos tar llamados *blobs* o capas (*layers*). Las capas se componen juntas utilizando metadatos de imagen y se compactan (*squashed*) en una única vista del sistema de archivos. Este resultado se puede lograr de muchas maneras, pero el enfoque más común en la actualidad es mediante el uso de sistemas de archivos de unión (*union filesystems*).

**OverlayFS** ([https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)) es el sistema de archivos de unión más utilizado en la actualidad. Se mantiene en el árbol del kernel, a pesar de no ser completamente compatible con POSIX.

Según la documentación del kernel: *"Un sistema de archivos overlay combina dos sistemas de archivos: un sistema de archivos 'superior' (upper) y un sistema de archivos 'inferior' (lower)"*. Esto significa que puede combinar varios árboles de directorios y proporcionar una vista única y unificada. Los directorios son las capas y se denominan `lowerdir` y `upperdir` para definir respectivamente el directorio de bajo nivel y el apilado encima de él. La vista unificada se llama `merged`. Soporta hasta 128 capas.

OverlayFS no es consciente del concepto de imágenes de contenedores; simplemente se utiliza como una tecnología fundacional para implementar la solución multicapa utilizada por las imágenes OCI.

Las imágenes OCI también implementan el concepto de **inmutabilidad**. Las capas de una imagen son todas de solo lectura y no se pueden modificar. La única forma de cambiar algo en las capas inferiores es reconstruir la imagen con los cambios correspondientes.

La inmutabilidad es un pilar importante del enfoque de la computación en la nube. Simplemente significa que una infraestructura (como una instancia, un contenedor o incluso clústeres complejos) solo se puede reemplazar por una versión diferente y no modificarse para lograr el despliegue objetivo. Por lo tanto, por lo general no cambiamos nada dentro de un contenedor en ejecución (por ejemplo, instalando paquetes o actualizando archivos de configuración manualmente), aunque podría ser posible en algunos contextos. En su lugar, reemplazamos su imagen base con una nueva versión actualizada. Esto también garantiza que cada copia de los contenedores en ejecución se mantenga sincronizada con las demás.

Cuando se ejecuta un contenedor, se crea una nueva capa delgada de lectura/escritura (*R/W layer*) encima de la imagen. Esta capa es efímera, por lo que cualquier cambio realizado en ella se perderá tras destruir el contenedor:

*Figura 1.3 – Capas de un contenedor*

Esto nos lleva a otra afirmación importante: no almacenamos nada dentro de los contenedores. Su único propósito es ofrecer un entorno de ejecución funcional y consistente para nuestras aplicaciones. Los datos deben ser accedidos externamente, utilizando montajes vinculados (*bind mounts*) dentro del propio contenedor o almacenamiento de red (como Network File System (NFS), Simple Storage Service (S3), Internet Small Computer System Interface (iSCSI), etc.).

El aislamiento de montaje de los contenedores y el diseño de imágenes en capas proporcionan una infraestructura inmutable consistente, pero son necesarias más restricciones de seguridad para evitar que los procesos con comportamientos maliciosos escapen del aislamiento del contenedor (*sandbox*) para robar información confidencial del host o utilizar el host para atacar otras máquinas. La siguiente subsección introduce consideraciones de seguridad para aprender cómo los runtimes de contenedores pueden limitar esos comportamientos.

#### Consideraciones de seguridad

Desde el punto de vista de la seguridad, hay una cruda verdad que compartir: que un proceso se esté ejecutando dentro de un contenedor no significa simplemente que sea más seguro que otros.

Un atacante malicioso aún podría abrirse camino a través del sistema de archivos y los recursos de memoria del host. Para lograr un mejor aislamiento de seguridad, se encuentran disponibles características adicionales:

- **Control de acceso obligatorio (*Mandatory Access Control - MAC*)**: Security Enhanced Linux (SELinux) o AppArmor se pueden utilizar para reforzar el aislamiento del contenedor contra el host principal. Estos subsistemas y sus utilidades de línea de comandos asociadas utilizan un enfoque basado en políticas para aislar mejor los procesos en ejecución en términos de acceso al sistema de archivos y a la red.
- **Capacidades (*Capabilities*)**: Cuando se ejecuta un proceso no privilegiado en el sistema (lo que significa un proceso con un UID efectivo diferente de 0), está sujeto a una comprobación de permisos basada en las credenciales del proceso (su UID efectivo). Esos permisos, o privilegios, se denominan capacidades y se pueden habilitar de forma independiente, asignando a un proceso no privilegiado un conjunto restringido de permisos privilegiados para acceder a recursos específicos. Al ejecutar un contenedor, podemos agregar o descartar capacidades (*drop capabilities*).
- **Modo de computación segura (*Secure Computing Mode - Seccomp*)**: Esta es una característica nativa del kernel que se puede utilizar para restringir las llamadas al sistema que un proceso puede realizar desde el espacio de usuario al espacio del kernel. Al identificar los privilegios estrictamente necesarios que requiere un proceso para ejecutarse, los administradores pueden aplicar perfiles seccomp para limitar la superficie de ataque.

Aplicar las características de seguridad anteriores manualmente no siempre es fácil ni inmediato, ya que algunas de ellas requieren una curva de aprendizaje considerable. Las herramientas que automatizan y simplifican (posiblemente de manera declarativa) estas restricciones de seguridad aportan un gran valor.

Analizaremos los temas de seguridad con más detalle en el Capítulo 10.

#### Motores y entornos de ejecución de contenedores (*Container engines and runtimes*)

A pesar de ser viable y particularmente útil desde el punto de vista del aprendizaje, ejecutar y asegurar contenedores manualmente es un enfoque poco confiable y complejo. Es demasiado difícil de reproducir y automatizar en entornos de producción, y puede conducir fácilmente a discrepancias de configuración (*configuration drift*) entre diferentes hosts.

Esta es la razón por la que nacieron los motores y runtimes de contenedores: para ayudar a automatizar la creación de un contenedor y todas las tareas asociadas necesarias que culminan en un contenedor en ejecución.

Los dos conceptos son bastante diferentes y suelen confundirse con frecuencia, por lo que requieren una explicación:

Un **motor de contenedores (*container engine*)** es una herramienta de software que acepta y procesa solicitudes de los usuarios para crear un contenedor con todos los argumentos y parámetros necesarios. Puede verse como una especie de orquestador, ya que se encarga de poner en marcha todas las acciones necesarias para tener el contenedor listo y en ejecución; sin embargo, no es el ejecutor efectivo del contenedor (esa es la función del runtime del contenedor). Los motores suelen resolver los siguientes problemas:

- Proporcionar una línea de comandos y/o interfaz REST para la interacción del usuario.
- Descargar y extraer imágenes de contenedores (tratado más adelante en este libro).
- Gestionar el punto de montaje del contenedor y realizar el *bind mount* de la imagen extraída.
- Manejar los metadatos del contenedor.
- Interactuar con el runtime de contenedores.

Ya mencionamos que cuando se instancia un nuevo contenedor, se crea una capa delgada de R/W encima de la imagen; esta tarea la realiza el motor de contenedores, que se encarga de presentar una pila funcional de los directorios combinados al runtime del contenedor.

El ecosistema de contenedores ofrece una amplia variedad de motores de contenedores. Docker es, sin duda, la implementación de motor más conocida (a pesar de no ser la primera), junto con **Podman** (el tema central de este libro), CRI-O, containerd y LXD.

Un **entorno de ejecución de contenedores (*container runtime*)** es una pieza de software de nivel inferior utilizada por los motores de contenedores para ejecutar contenedores en el host. El runtime del contenedor proporciona las siguientes funcionalidades:

- Iniciar el proceso contenedorizado en el punto de montaje de destino (generalmente proporcionado por el motor de contenedores) con un conjunto de metadatos personalizados.
- Gestionar la asignación de recursos de los cgroups.
- Gestionar las políticas de control de acceso obligatorio (SELinux y AppArmor) y las capacidades.

Hoy en día existen muchos runtimes de contenedores, y la mayoría de ellos implementan la especificación de referencia del runtime de la OCI ([https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)). Este es un estándar de la industria que define cómo debe comportarse un runtime y la interfaz que debe implementar.

El runtime OCI más común es **runc**, utilizado por la mayoría de los motores notables, junto con otras implementaciones como **crun**, **kata-containers**, **youki** y **gVisor**.

Este enfoque modular permite que los motores de contenedores intercambien el runtime de contenedores según sea necesario. Por ejemplo, cuando se lanzó Fedora 33, introdujo una nueva jerarquía de cgroups por defecto llamada cgroups v2. runc inicialmente no era compatible con cgroups v2, y Podman simplemente cambió runc por otro runtime de contenedores compatible con OCI (crun) que ya era compatible con la nueva jerarquía. Ahora que runc finalmente admite cgroups v2, Podman podrá usarlo nuevamente de forma segura sin ningún impacto para el usuario final.

Después de presentar los runtimes y motores de contenedores, es hora de abordar una de las preguntas más debatidas y formuladas durante las introducciones a los contenedores: la diferencia entre contenedores y máquinas virtuales.

#### Contenedores frente a máquinas virtuales

Hasta ahora, hemos hablado del aislamiento logrado con las características nativas del sistema operativo y mejorado con los motores y runtimes de contenedores. Muchos usuarios podrían caer en el error de pensar que los contenedores son una forma de virtualización.

Nada más lejos de la realidad; **los contenedores no son máquinas virtuales**.

Entonces, ¿cuál es la principal diferencia entre un contenedor y una máquina virtual? Antes de responder, podemos observar el siguiente diagrama:

*Figura 1.4 – Una llamada al sistema a un kernel desde un contenedor*

Un contenedor, a pesar de estar aislado, contiene un proceso que interactúa directamente con el kernel del host mediante **llamadas al sistema (*System Calls*)**. Puede que el proceso no sea consciente de los namespaces del host, pero aún necesita cambiar de contexto al espacio del kernel (*Kernel Space*) para realizar operaciones como el acceso a E/S.

Por otro lado, una máquina virtual siempre se ejecuta sobre un hipervisor, ejecutando un sistema operativo invitado (*guest OS*) con su propio sistema de archivos, red, almacenamiento (generalmente como archivos de imagen) y kernel. El hipervisor es un software que proporciona una capa de abstracción y virtualización de hardware al sistema operativo invitado, permitiendo que una sola máquina física (*bare-metal*) que se ejecute en hardware adecuado instancie muchas máquinas virtuales. El hardware visto por el kernel del sistema operativo invitado es principalmente hardware virtualizado, con algunas excepciones:

*Figura 1.5 – Arquitectura: virtualización frente a contenedores*

Esto significa que cuando un proceso realiza una llamada al sistema dentro de una máquina virtual, siempre se dirige al kernel del sistema operativo invitado.

En resumen, podemos afirmar que **los contenedores comparten el mismo kernel con el host, mientras que las máquinas virtuales tienen su propio kernel de sistema operativo invitado**. Esta afirmación implica muchas consideraciones.

Desde el punto de vista de la seguridad, las máquinas virtuales proporcionan un mejor aislamiento frente a posibles ataques. De todos modos, algunos de los últimos ataques basados en CPU (Spectre o Meltdown, más notablemente) podrían aprovechar las vulnerabilidades de la CPU para acceder a los espacios de direcciones de las máquinas virtuales.

Los contenedores cuentan con funciones de aislamiento refinadas y se pueden configurar con políticas de seguridad estrictas (como CIS Docker, NIST, HIPAA, etc.) que hacen que sean bastante difíciles de vulnerar.

Desde el punto de vista de la escalabilidad, los contenedores son más rápidos de iniciar que las máquinas virtuales. Poner en marcha una nueva instancia de contenedor es cuestión de milisegundos si la imagen ya está disponible en el host. Estos resultados rápidos también se logran gracias a la naturaleza sin kernel (*kernel-less*) del contenedor. Las máquinas virtuales deben arrancar un kernel e initramfs, pivotar hacia el sistema de archivos raíz, ejecutar algún tipo de init (como systemd) e iniciar un número variable de servicios.

Una máquina virtual consumirá habitualmente más recursos que un contenedor. Para poner en marcha un sistema operativo invitado, normalmente necesitamos asignar más memoria RAM, CPU y almacenamiento que los recursos necesarios para iniciar un contenedor.

Otro gran diferenciador entre las máquinas virtuales y los contenedores es el enfoque en las cargas de trabajo. La mejor práctica para los contenedores es poner en marcha un contenedor para cada carga de trabajo específica. Por otro lado, una máquina virtual puede ejecutar diferentes cargas de trabajo juntas.

Imagina una arquitectura LAMP o WordPress: en entornos que no son de producción o en entornos de producción pequeños, no sería extraño tener todo (Apache, PHP, MySQL y WordPress) instalado en la misma máquina virtual. Este diseño se dividiría en una arquitectura de múltiples contenedores (o múltiples capas), con un contenedor ejecutando el frontend (Apache-PHP-WordPress) y un contenedor ejecutando la base de datos MySQL. El contenedor que ejecuta MySQL podría acceder a volúmenes de almacenamiento para persistir los archivos de la base de datos. Al mismo tiempo, sería más fácil escalar hacia arriba o hacia abajo los contenedores del frontend.

Ahora que comprendemos cómo funcionan los contenedores y qué los diferencia de las máquinas virtuales, podemos pasar a la siguiente gran pregunta: ¿Realmente puedo poner una máquina virtual dentro de un contenedor?

#### Máquinas virtuales en contenedores

En el panorama informático en constante evolución, las máquinas virtuales y los contenedores han surgido como dos actores principales. Imagina las máquinas virtuales como servidores completamente autónomos, completos con su propio sistema operativo y todo lo necesario para ejecutar aplicaciones. Esto las hace ideales para ejecutar software heredado que podría no ser compatible con tecnologías más nuevas. Por otro lado, los contenedores son paquetes ligeros que contienen solo las partes esenciales que una aplicación necesita para ejecutarse. Comparten los recursos del servidor en el que se encuentran, lo que los hace increíblemente eficientes.

Tradicionalmente, los contenedores se han colocado dentro de máquinas virtuales para combinar sus beneficios. Sin embargo, herramientas innovadoras como **KubeVirt** ahora nos permiten revertir esta relación y colocar máquinas virtuales dentro de contenedores. Este cambio de paradigma ofrece multitud de ventajas. Las empresas con software heredado que funciona mejor en máquinas virtuales ahora pueden integrarlo perfectamente con aplicaciones más nuevas basadas en contenedores, todo en la misma plataforma. Además, los contenedores proporcionan una capa adicional de seguridad, aislando las máquinas virtuales entre sí y del resto de los servidores. Kubernetes, una popular herramienta para gestionar contenedores, ahora también puede gestionar máquinas virtuales, optimizando las operaciones de los equipos de TI.

Para comprender cómo se ejecutan las máquinas virtuales en contenedores, necesitamos introducir dos componentes clave más: **QEMU** (*Quick EMUlator*) y **KVM** (*Kernel-based Virtual Machine*). QEMU es una herramienta versátil que emula varios componentes de hardware de servidores, engañando a una máquina virtual para que piense que se está ejecutando en su propia máquina dedicada, aunque en realidad esté en un contenedor. KVM, un módulo del kernel, mejora el rendimiento de QEMU al permitir que las máquinas virtuales accedan directamente al hardware del servidor.

En esencia, colocar máquinas virtuales en contenedores permite a las empresas combinar lo mejor de ambos mundos, mejorando la seguridad, la eficiencia y la gestión general de su infraestructura de software.

#### Contenedores arrancables (*Bootable containers*)

La evolución de la tecnología de contenedores ha llevado a nuevas y emocionantes fronteras, como los muy recientes **contenedores arrancables** (*bootable containers*). Impulsados por el proyecto de código abierto **bootc**, los contenedores arrancables representan una evolución significativa más allá de los contenedores tradicionales. En lugar de limitarse a encapsular aplicaciones, los contenedores arrancables están diseñados para encapsular un sistema operativo completo, desdibujando efectivamente las líneas divisorias entre contenedores, máquinas virtuales y sistemas físicos.

A un alto nivel, un contenedor arrancable es una imagen de contenedor especializada que se puede arrancar directamente en un sistema, de la misma manera que se arrancaría un sistema operativo tradicional. Esto se logra empaquetando un entorno de sistema operativo completo, incluidos el kernel, las bibliotecas esenciales y las herramientas del sistema, dentro de la propia imagen del contenedor. Los contenedores arrancables también cumplen con el estándar más reciente de la Open Container Initiative (OCI), que analizaremos en los párrafos siguientes.

Los beneficios de este enfoque son diversos. En comparación con los contenedores de aplicaciones estándar, los contenedores arrancables ofrecen un aislamiento y control mucho mayores. Dado que abarcan todo el sistema operativo, proporcionan un entorno seguro y autónomo para ejecutar aplicaciones, minimizando el riesgo de conflictos e interferencias.

Además, los contenedores arrancables ofrecen una alternativa atractiva a las máquinas virtuales tradicionales. Al eliminar la necesidad de un hipervisor y un sistema operativo invitado, reducen significativamente la sobrecarga de recursos, lo que resulta en tiempos de arranque más rápidos, menor consumo de memoria y un mejor rendimiento general. Esto los hace ideales para escenarios donde se desea un entorno de sistema operativo ligero, portátil y eficiente, como sistemas embebidos, computación en el borde (*edge computing*) o servidores especializados para aplicaciones de aprendizaje automático e inteligencia artificial.

Con herramientas como `bootc`, la creación y gestión de contenedores arrancables es cada vez más accesible, lo que permite a los desarrolladores y administradores de sistemas aprovechar el poder de los contenedores de formas nuevas y emocionantes. Por ejemplo, **Podman Desktop** cuenta con una extensión de contenedores arrancables que agiliza la construcción y gestión de estos contenedores únicos, proporcionando una interfaz intuitiva para la experimentación y el despliegue.

Ahora que hemos descubierto cómo se pueden utilizar los contenedores para crear máquinas virtuales o imágenes arrancables para sistemas físicos, podemos investigar por qué podríamos necesitar un contenedor.

---

### ¿Por qué necesito un contenedor?

Esta sección describe los beneficios y el valor de los contenedores en los sistemas de TI modernos, y cómo los contenedores pueden proporcionar beneficios tanto para la tecnología como para el negocio.

La pregunta anterior podría reformularse como: ¿cuál es el valor de adoptar contenedores en producción?

El sector de TI se ha convertido en un entorno rápido y orientado al mercado donde los cambios están dictados por el negocio y las mejoras tecnológicas. Al adoptar tecnologías emergentes, las empresas siempre están atentas a su Retorno de la Inversión (*Return on Investment* - ROI) mientras se esfuerzan por mantener el Coste Total de Propiedad (*Total Cost of Ownership* - TCO) bajo umbrales razonables. Esto no siempre es fácil de lograr.

Esta sección intentará descubrir las razones más importantes que hacen que los contenedores sean beneficiosos.

#### Código abierto (*Open source*)

Las tecnologías que impulsan los contenedores son de código abierto y se han convertido en estándares abiertos ampliamente adoptados por muchos proveedores y comunidades. El software de código abierto, adoptado hoy en día por grandes empresas, proveedores y proveedores de servicios en la nube, tiene muchas ventajas y proporciona un gran valor para la empresa. El código abierto a menudo se asocia con soluciones de alto valor e innovadoras: ¡esa es simplemente la verdad!

En primer lugar, los proyectos impulsados por la comunidad suelen tener un gran impulso evolutivo que ayuda a madurar el código y a incorporar nuevas funciones de forma continua. El software de código abierto está disponible para el público y se puede inspeccionar y analizar. Esta es una gran característica de transparencia que también tiene un impacto en la confiabilidad del software, tanto en términos de robustez como de seguridad.

Uno de los aspectos clave es que promueve un paradigma evolutivo donde solo se adopta, contribuye y respalda el mejor software; la tecnología de contenedores es un ejemplo perfecto de este comportamiento.

#### Portabilidad

Ya hemos mencionado que los contenedores son una tecnología que permite a los usuarios empaquetar y aislar aplicaciones con todo su entorno de ejecución, lo que incluye todos los archivos necesarios para ejecutarse. Esta característica desbloquea un beneficio clave: la **portabilidad**.

Esto significa que una imagen de contenedor se puede descargar y ejecutar en cualquier host que tenga un motor de contenedores en ejecución, independientemente de la distribución del sistema operativo subyacente. Se puede descargar indistintamente una imagen de CentOS o nginx desde una distribución de Linux Fedora o Debian que ejecute un motor de contenedores y ejecutarla con la misma configuración.

Nuevamente, si tenemos una flota de muchos hosts idénticos, podemos optar por programar la instancia de la aplicación en uno de ellos (por ejemplo, utilizando métricas de carga para elegir la mejor opción) con la seguridad de obtener el mismo resultado al ejecutar el contenedor.

La portabilidad de los contenedores también reduce la dependencia de proveedores (*vendor lock-in*) y proporciona una mejor interoperabilidad entre plataformas.

#### Facilitadores de DevOps

Como se indicó anteriormente, los contenedores ayudan a resolver el antiguo patrón *"Funciona en mi máquina"* entre los equipos de desarrollo y operaciones cuando se trata de desplegar aplicaciones para producción.

Como solución de empaquetado inteligente y sencilla para aplicaciones, satisfacen la necesidad de los desarrolladores de crear paquetes autoconsistentes con todos los binarios y configuraciones necesarios para ejecutar sus cargas de trabajo sin problemas. Como una forma autoconsistente de aislar procesos y garantizar la separación de namespaces y el uso de recursos, son apreciados por los equipos de operaciones, que ya no se ven obligados a mantener complejas restricciones de dependencia ni a segregar cada aplicación dentro de máquinas virtuales.

Desde este punto de vista, los contenedores pueden considerarse facilitadores de las mejores prácticas de **DevOps**, donde los desarrolladores y los operadores trabajan más estrechamente para desplegar y gestionar aplicaciones sin separaciones rígidas.

Se espera que los desarrolladores que quieran crear sus propias imágenes de contenedores sean más conscientes de la capa del sistema operativo integrada en la imagen y trabajen en estrecha colaboración con los equipos de operaciones para definir plantillas de construcción y automatizaciones.

#### Preparación para la nube (*Cloud readiness*)

Los contenedores están diseñados para la nube, pensados con un enfoque inmutable en mente. El patrón de inmutabilidad establece claramente que los cambios en la infraestructura (ya sea un solo contenedor o un clúster complejo) deben aplicarse volviendo a desplegar una versión modificada y no parcheando la actual. Esto ayuda a aumentar la previsibilidad y la confiabilidad del sistema.

Cuando se debe lanzar una nueva versión de una aplicación, se compila en una nueva imagen y se despliega un nuevo contenedor en lugar de la versión anterior. Se pueden implementar canalizaciones de construcción (*build pipelines*) para gestionar flujos de trabajo complejos, desde la compilación de la aplicación y la creación de imágenes, el envío al registro de imágenes y el etiquetado, hasta el despliegue en el host de destino. Este enfoque acorta drásticamente el tiempo de aprovisionamiento al tiempo que reduce las inconsistencias.

Veremos más adelante, en el Capítulo 14, que las soluciones dedicadas a la orquestación de contenedores, como Kubernetes, también proporcionan formas de automatizar los patrones de programación de grandes flotas de hosts y hacen que las cargas de trabajo en contenedores sean fáciles de desplegar, monitorizar y escalar.

#### Optimización de la infraestructura

En comparación con las máquinas virtuales, los contenedores tienen una huella ligera que impulsa una eficiencia mucho mayor en el consumo de recursos de cómputo y memoria. Al proporcionar una forma de simplificar la ejecución de la carga de trabajo, la adopción de contenedores genera un gran ahorro de costes.

La optimización de los recursos de TI se logra reduciendo el coste computacional de las aplicaciones; si un servidor de aplicaciones que se ejecutaba sobre una máquina virtual se puede contenerizar y ejecutar en un host junto con otros contenedores (con límites y solicitudes de recursos dedicados), los recursos informáticos se pueden ahorrar y reutilizar.

Se pueden remodular infraestructuras enteras teniendo en cuenta este nuevo paradigma; una máquina física configurada previamente como hipervisor se puede reasignar como un nodo de trabajo de un sistema de orquestación de contenedores que simplemente ejecuta aplicaciones contenedorizadas más granulares.

#### Microservicios

Las arquitecturas de microservicios dividen las aplicaciones en múltiples servicios que realizan funciones detalladas y forman parte de la aplicación en su conjunto.

Las aplicaciones tradicionales tienen un enfoque monolítico donde todas las funciones son parte de la misma instancia. El propósito de los microservicios es dividir el monolito en partes más pequeñas que interactúen de forma independiente.

Las aplicaciones monolíticas encajan bien en contenedores, pero las aplicaciones de microservicios tienen una coincidencia ideal con ellos.

Tener un contenedor para cada microservicio individual ayuda a lograr beneficios importantes, como los siguientes:

- Escalabilidad independiente de los microservicios.
- Responsabilidades más definidas para los equipos de desarrollo.
- Adopción potencial de diferentes pilas tecnológicas en los diferentes microservicios.
- Mayor control sobre los aspectos de seguridad (como servicios expuestos al público, conexiones mTLS, etc.).

Orquestar microservicios puede ser una tarea abrumadora cuando se trata de arquitecturas grandes y articuladas. La adopción de plataformas de orquestación como Kubernetes, soluciones de malla de servicios (*service mesh*) como Istio o Linkerd, y herramientas de rastreo como Jaeger y Kiali se vuelve crucial para lograr el control sobre la complejidad.

#### Inteligencia Artificial y Aprendizaje Automático (IA/ML)

Los contenedores también son una herramienta valiosa para desarrollar, entrenar y servir modelos de aprendizaje automático/aprendizaje profundo o modelos de lenguaje de gran tamaño (LLM). Los científicos e ingenieros de datos pueden usar contenedores para aislar sus cuadernos de Jupyter (una herramienta estándar *de facto* para implementar soluciones basadas en modelos de IA/ML) y ejecutarlos en entornos personalizados separados.

El entrenamiento es una parte muy importante del desarrollo de modelos. Desarrollar un modelo desde cero puede ser una tarea muy larga y costosa, especialmente para los LLM: por esta razón, muchos usuarios eligen modelos fundacionales genéricos y aplican técnicas de ajuste fino (*fine-tuning*) alimentando al modelo con conjuntos de datos más especializados. El proceso de ajuste fino se puede ejecutar dentro de contenedores, por ejemplo, en una canalización de entrenamiento que libera un nuevo modelo especializado a partir del fundacional.

Una vez que el modelo ha sido desarrollado y entrenado, los ingenieros de MLOps pueden ejecutar el modelo exponiendo API de inferencia que, nuevamente, se pueden ejecutar dentro de contenedores dedicados, proporcionando así una mejor capa de encapsulación de la carga de trabajo y una gestión óptima de los recursos.

Acabamos de presentar múltiples casos de uso para la tecnología de contenedores, pero ¿de dónde provienen los contenedores? La tecnología de contenedores no es un tema nuevo en la industria informática, como veremos en los próximos párrafos. Tiene raíces profundas en la historia de los sistemas operativos, ¡y descubriremos que podría ser incluso más antigua que nosotros!

---

### ¿De dónde provienen los contenedores?

Esta sección rebobina la cinta y resume los hitos más importantes de los contenedores en la historia de los sistemas operativos, desde Unix hasta las máquinas GNU/Linux: una mirada útil al pasado para comprender cómo evolucionó la idea subyacente a lo largo de los años.

#### Chroot y Unix v7

Si queremos crear una cronología de eventos para nuestro viaje en el tiempo por la historia de los contenedores, el primer y más antiguo destino es 1979: el año de Unix V7. En aquel momento, allá por 1979, se introdujo una importante llamada al sistema en el kernel de Unix: la llamada al sistema **chroot**.

> [!NOTE]
> Una **llamada al sistema (*system call* o *syscall*)** es un método utilizado por una aplicación para solicitar algo al kernel del sistema operativo.

Esta llamada al sistema permite que la aplicación cambie el directorio raíz de la copia en ejecución de sí misma y de sus hijos, eliminando cualquier capacidad del software en ejecución para escapar de esa jaula (*jail*). Esta característica permite prohibir el acceso de la aplicación en ejecución a cualquier tipo de archivo o directorio fuera del subárbol dado, lo que realmente cambió las reglas del juego para esa época.

Después de algunos años, allá por 1982, esta llamada al sistema también se introdujo en los sistemas BSD.

Desafortunadamente, esta característica no se creó pensando en la seguridad y, a lo largo de los años, la documentación del sistema operativo y la literatura de seguridad desaconsejaron enérgicamente el uso de jaulas chroot como mecanismo de seguridad para lograr el aislamiento.

Chroot fue solo el primer hito en el viaje hacia el aislamiento completo de procesos en sistemas *nix. El siguiente fue, desde un punto de vista histórico, la introducción de las jaulas de FreeBSD (*FreeBSD jails*).

#### FreeBSD jails

Dando algunos pasos hacia adelante en nuestro viaje histórico, saltamos hacia adelante hasta el año 2000, cuando el sistema operativo FreeBSD aprobó y lanzó un nuevo concepto que amplía la clásica llamada al sistema chroot: **FreeBSD jails**.

> [!NOTE]
> **FreeBSD** es un sistema operativo libre y de código abierto similar a Unix lanzado por primera vez en 1993, nacido de la Berkeley Software Distribution, que originalmente se basó en Research Unix.

Como informamos brevemente con anterioridad, chroot fue una gran característica en los años 80, pero la jaula que crea se puede evadir fácilmente y tiene muchas limitaciones, por lo que no era adecuada para escenarios complejos. Por esa razón, las jaulas de FreeBSD se construyeron sobre la syscall chroot, con el objetivo de ampliar y extender su conjunto de características.

En un entorno chroot estándar, un proceso en ejecución tiene limitaciones y aislamiento solo a nivel del sistema de archivos; todo lo demás, como los procesos en ejecución, los recursos del sistema, el subsistema de red y los usuarios del sistema, es compartido por los procesos dentro del chroot y los procesos del sistema host.

En cuanto a las jaulas de FreeBSD, su característica principal es la virtualización del subsistema de red, los usuarios del sistema y sus procesos; como puedes imaginar, esto mejora considerablemente la flexibilidad y la seguridad general de la solución.

Esquematicemos las cuatro características clave de una jaula de FreeBSD:

- **Subárbol de directorios**: Esto es lo que ya vimos para la jaula chroot. Básicamente, una vez definido como un subárbol, el proceso en ejecución se limita a él y no puede escapar.
- **Dirección IP**: Esta es una gran revolución; finalmente, podemos definir una dirección IP independiente para nuestra jaula y permitir que nuestro proceso en ejecución esté aislado incluso del sistema host.
- **Nombre de host (*Hostname*)**: Utilizado dentro de la jaula, este es, por supuesto, diferente del sistema host.
- **Comando**: Este es el ejecutable en ejecución y tiene la opción de ejecutarse dentro de la jaula del sistema. El ejecutable tiene una ruta relativa que es autónoma dentro de la jaula.

Una ventaja de este tipo de jaula es que cada instancia también tiene sus propios usuarios y cuenta de root, la cual no tiene ningún tipo de privilegios o permisos sobre las otras jaulas o el sistema host subyacente.

Otra característica interesante de las jaulas de FreeBSD es que tenemos dos formas de instalar/crear una jaula:

- **Desde binarios**: Reflejando los que podríamos instalar con el sistema operativo subyacente.
- **Desde el código fuente**: Construyendo desde cero lo que necesita la aplicación final.

#### Solaris Containers (también conocidos como Solaris Zones)

Volviendo a nuestra máquina del tiempo, debemos avanzar solo unos años, hasta 2004 para ser exactos, para encontrar finalmente la primera terminología que podemos reconocer: **Solaris Containers**.

> [!NOTE]
> **Solaris** es un sistema operativo Unix propietario nacido de SunOS en 1993, desarrollado originalmente por Sun Microsystems.

Solaris Containers fue en realidad solo un nombre transitorio de **Solaris Zones**, una tecnología de virtualización integrada en el sistema operativo Solaris, con la ayuda también de un sistema de archivos especial, ZFS, que permite instantáneas (*snapshots*) de almacenamiento y clonación.

Una zona es un entorno de aplicación virtualizado, construido a partir del sistema operativo subyacente, que permite un aislamiento completo entre el sistema host base y cualquier otra aplicación que se ejecute dentro de otras zonas.

La característica destacada que introdujo Solaris Zones es el concepto de **zona con marca (*branded zone*)**. Una branded zone es un entorno completamente diferente en comparación con el sistema operativo subyacente, y puede contener diferentes binarios, kits de herramientas o ¡incluso un sistema operativo diferente!

Finalmente, para garantizar el aislamiento, una zona de Solaris puede tener su propia red, sus propios usuarios e incluso su propia zona horaria.

#### Linux Containers (LXC)

Avancemos cuatro años más, donde nos encontraremos con **Linux Containers (LXC)**. Estamos en 2008, cuando se lanzó la primera solución completa de gestión de contenedores de Linux.

LXC no puede simplificarse simplemente como un gestor para una de las primeras implementaciones de contenedores de Linux, porque sus autores desarrollaron muchas de las características del kernel que ahora también se utilizan para otros runtimes de contenedores en Linux.

LXC tiene su propio runtime de contenedores de bajo nivel, y sus autores lo crearon con el objetivo de ofrecer un entorno aislado lo más cercano posible a las máquinas virtuales, pero sin la sobrecarga necesaria para simular el hardware y ejecutar una nueva instancia del kernel. Los contenedores de Linux logran dicho objetivo y aislamiento gracias a las siguientes funcionalidades del kernel:

- Namespaces
- Control de acceso obligatorio (*Mandatory Access Control*)
- Grupos de control (también conocidos como cgroups)

Recapitulemos las funcionalidades del kernel que vimos anteriormente en el capítulo:

- **Namespaces de Linux**: Un namespace [3] aísla procesos que abstraen un recurso global del sistema. Si un proceso realiza cambios en un recurso del sistema en un namespace, estos cambios solo son visibles para otros procesos dentro del mismo namespace. El uso común de la función de namespaces es implementar contenedores.
- **Control de acceso obligatorio**: En el ecosistema de Linux, existen varias implementaciones de MAC disponibles; el proyecto más conocido es SELinux, desarrollado por la Agencia de Seguridad Nacional (NSA) de EE. UU.

> [!NOTE]
> **SELinux** es una implementación de arquitectura de control de acceso obligatorio utilizada en los sistemas operativos Linux. Proporciona control de acceso basado en roles y seguridad multinivel mediante un mecanismo de etiquetado. Cada archivo, dispositivo y directorio tiene una etiqueta asociada (a menudo descrita como contexto de seguridad) que extiende los atributos comunes del sistema de archivos.

- **Grupos de control (*cgroups*)**: Los grupos de control son una característica integrada del kernel de Linux que puede ayudar a organizar, en grupos jerárquicos, varios tipos de recursos, incluidos los procesos. Estos recursos luego se pueden limitar y monitorizar. La interfaz común utilizada para interactuar con los cgroups es un pseudo-sistema de archivos llamado `cgroupfs`. Esta característica del kernel es realmente útil para rastrear y limitar los recursos de los procesos, como la memoria, la CPU, etc.

La principal y mayor característica de LXC que proviene de estas tres funcionalidades del kernel es, sin duda, los **contenedores sin privilegios (*unprivileged containers*)**.

Gracias a los namespaces, MAC y cgroups, de hecho, LXC puede aislar un cierto número de UID y GID, mapeándolos con el sistema operativo subyacente. Esto asegura que `UID=0` en el contenedor esté (en realidad) mapeado a un UID más alto en el host del sistema base.

Dependiendo de los privilegios y del conjunto de características que queramos asignar a nuestro contenedor, podemos elegir entre un amplio conjunto de tipos de namespaces preconstruidos, como los siguientes:

- **Red (*Network*)**: Ofrece acceso a dispositivos de red, pilas, puertos, etc.
- **Montaje (*Mount*)**: Ofrece acceso a puntos de montaje.
- **PID**: Ofrece acceso a los identificadores de procesos.

La siguiente evolución principal de LXC (y, sin duda, la que desencadenó el éxito de la adopción de contenedores) fue ciertamente Docker.

#### Docker

Tras solo 5 años, en 2013, Docker irrumpió en el panorama de los contenedores y rápidamente se volvió muy popular. Pero, ¿qué características se utilizaban en aquellos días? Bueno, ¡podemos descubrir fácilmente que uno de los primeros motores de contenedores de Docker fue LXC!

Tras solo un año de desarrollo, el equipo de Docker introdujo `libcontainer` y finalmente reemplazó el motor de contenedores LXC con su propia implementación. Docker, similar a su predecesor LXC, requiere un demonio que se ejecute en el sistema host base para mantener los contenedores en funcionamiento y funcionando correctamente.

Una de las características más notables (aparte del uso de namespaces, MAC y cgroups) fue, con certeza, **OverlayFS**, un sistema de archivos superpuesto que ayuda a combinar múltiples sistemas de archivos en un solo sistema de archivos.

> [!NOTE]
> **OverlayFS** es un sistema de archivos de unión de Linux. Puede combinar múltiples puntos de montaje en uno, creando una estructura de directorio única que contiene todos los archivos y subdirectorios subyacentes de los orígenes.

A un alto nivel, el equipo de Docker introdujo el concepto de **imágenes de contenedores** y **registros de contenedores**, que realmente cambiaron las reglas del juego en cuanto a funcionalidad. Los conceptos de registro e imagen permitieron la creación de un ecosistema completo en el que cada desarrollador, administrador de sistemas o entusiasta de la tecnología podía colaborar y contribuir con sus propias imágenes de contenedores personalizadas. También crearon un formato de archivo especial para crear nuevas imágenes de contenedores (`Dockerfile`) para automatizar fácilmente los pasos necesarios para construir las imágenes de contenedores desde cero.

Junto a Docker, surgió otro proyecto de motor/runtime que captó el interés de las comunidades: **rkt**.

#### rkt

Pocos años después del auge de Docker, entre 2014 y 2015, la empresa CoreOS (adquirida posteriormente por Red Hat) lanzó su propia implementación de un motor de contenedores que tenía una característica principal muy particular: **no tenía demonio (*daemon-less*)**.

Esta elección tuvo un impacto importante: en lugar de tener un demonio central que administrara un grupo de contenedores, cada contenedor funcionaba por su cuenta, como cualquier otro proceso estándar que podamos iniciar en nuestro sistema host base.

Pero el proyecto rkt (pronunciado *rocket*) se volvió muy popular en 2017 cuando la joven Cloud Native Computing Foundation (CNCF), cuyo objetivo es ayudar y coordinar proyectos relacionados con contenedores y la nube, decidió adoptar el proyecto bajo su tutela, junto con otro proyecto donado por el propio Docker: **containerd**.

En pocas palabras, el equipo de Docker extrajo el runtime central del proyecto de su demonio y lo donó a la CNCF, lo que supuso un gran paso que motivó y habilitó una gran comunidad en torno al tema de los contenedores, además de ayudar a desarrollar y mejorar las crecientes herramientas de orquestación de contenedores, como Kubernetes.

> [!NOTE]
> **Kubernetes** (del término griego *κυβερνήτης*, que significa "timonel"), también abreviado como K8s, es un sistema de orquestación de contenedores de código abierto para simplificar el despliegue y la gestión de aplicaciones en un entorno multihost. Fue publicado como un proyecto de código abierto por Google, pero ahora es mantenido por la CNCF.

Aunque el tema principal de este libro es Podman, debemos reconocer la creciente necesidad de orquestar proyectos complejos compuestos por muchos contenedores en entornos multimáquina; ahí es donde Kubernetes se alzó como líder del ecosistema.

Tras la adquisición de CoreOS por parte de Red Hat, el proyecto rkt fue discontinuado, pero su legado no se perdió e influyó en el desarrollo del proyecto Podman. Pero antes de presentar el tema principal de este libro, profundicemos en las especificaciones de OCI.

#### OCI y CRI-O

Como se mencionó anteriormente, la extracción de containerd de Docker y la consiguiente donación a la CNCF motivaron a la comunidad de código abierto a comenzar a trabajar seriamente en motores de contenedores que pudieran inyectarse bajo una capa de orquestación, como Kubernetes.

En la misma ola, en 2015, Docker, con la ayuda de muchas otras empresas (Red Hat, AWS, Google, Microsoft, IBM, etc.), inició un comité de gobernanza bajo el paraguas de la Fundación Linux: la **Open Container Initiative (OCI)**.

Bajo esta iniciativa, el equipo de trabajo desarrolló la especificación de runtime (*runtime spec*) [4] y la especificación de imagen (*image spec*) [5] para describir cómo se deberían crear la API y la arquitectura para los nuevos motores de contenedores en el futuro.

El mismo año, el equipo de OCI también lanzó la primera implementación de un runtime de contenedores que se adhería a las especificaciones de OCI; el proyecto se llamó **runc**.

La OCI definió no solo una especificación para ejecutar contenedores independientes, sino que también proporcionó la base para vincular la capa de Kubernetes con el motor de contenedores subyacente de manera más sencilla. Al mismo tiempo, la comunidad de Kubernetes lanzó la **Container Runtime Interface (CRI)** [6], una interfaz de plugins para permitir la adopción de una amplia variedad de runtimes de contenedores.

Ahí es donde CRI-O llega en 2017; publicado como un proyecto de código abierto por Red Hat, fue una de las primeras implementaciones de la CRI de Kubernetes, lo que permitió el uso de runtimes compatibles con OCI. CRI-O representa una alternativa ligera al uso de Docker, rkt o cualquier otro motor como runtime de Kubernetes.

A medida que el ecosistema continúa creciendo, los estándares y las especificaciones se adoptan cada vez más, lo que da lugar a un ecosistema de contenedores más amplio. Las especificaciones OCI mostradas anteriormente fueron cruciales para el desarrollo del runtime de contenedores runc, adoptado por el proyecto Podman.

#### Podman

Llegamos finalmente al final de nuestro viaje en el tiempo; alcanzamos el año 2017 en el párrafo anterior y, en ese mismo año, se realizó el primer commit del proyecto Podman en GitHub.

El nombre del proyecto revela mucho sobre su propósito: **PODMAN = POD MANager**. Ahora estamos listos para ver la definición básica de un pod en el mundo de los contenedores.

Un **pod** es la unidad de computación desplegable más pequeña que puede manejar Kubernetes; puede estar compuesto por uno o más contenedores. En el caso de múltiples contenedores en el mismo pod, se programan y ejecutan juntos en un contexto compartido.

Podman gestiona contenedores e imágenes de contenedores, sus volúmenes de almacenamiento y pods formados por uno o varios contenedores, y fue construido desde cero para adherirse a los estándares OCI.

Podman no tiene un demonio central que gestione los contenedores, sino que los inicia como procesos estándar del sistema. También define una interfaz de CLI compatible con Docker para facilitar la transición desde Docker.

Una de las grandes características introducidas por Podman son los **contenedores sin root (*rootless containers*)**. Por lo general, cuando pensamos en contenedores Linux, pensamos de inmediato en un administrador de sistemas que debe configurar algunos prerrequisitos a nivel del sistema operativo para preparar el entorno que permita que nuestro contenedor se ponga en funcionamiento.

Los contenedores rootless pueden ejecutarse fácilmente como un usuario normal, sin necesidad de root. Usar Podman con un usuario sin privilegios iniciará contenedores restringidos sin ningún privilegio, al igual que el usuario que lo ejecuta.

Sin lugar a dudas, Podman introdujo una mayor flexibilidad y es un proyecto muy activo cuya adopción crece constantemente. Cada versión principal trae muchas funciones nuevas; por ejemplo, la versión 3.0 introdujo soporte para Docker Compose, que era una característica muy solicitada. En la versión 4.0, se introdujo una nueva pila de red basada en los proyectos de código abierto Netavark y Aardvark para mejorar el rendimiento y la funcionalidad. Otra gran mejora llegó en la versión 5.0, con una reescritura importante de Podman Machine, esencial para ejecutar Podman en los sistemas operativos macOS o Windows, junto con una nueva pila de red para contenedores rootless basada en pasta. Red Hat anunció la intención de donar Podman, Podman Desktop y otras herramientas de contenedores relacionadas (Buildah, Skopeo, bootc, composefs) a la CNCF a finales de 2024, y los proyectos se unieron oficialmente al CNCF Sandbox en enero de 2025, consolidando su compromiso con un futuro abierto, neutral respecto a proveedores e impulsado por la comunidad. Esta es también una buena métrica de salud del apoyo comunitario.

Ahora que hemos cubierto los conceptos básicos, cerremos el capítulo con una descripción general de los casos de uso más comunes de adopción de contenedores.

---

### ¿Dónde se utilizan los contenedores hoy en día?

Esta es una sección abierta. La intención es explicar dónde y cómo se utilizan los contenedores hoy en día en un entorno de producción. Esta sección también introduce el concepto de orquestación de contenedores con Kubernetes, la solución de orquestador de código abierto más utilizada, adoptada por miles de empresas en todo el mundo.

La adopción de contenedores se está extendiendo a todas las empresas en todos los sectores comerciales. Pero si investigamos las historias de éxito de las empresas que ya utilizan contenedores o una distribución de Kubernetes, descubriremos que la contenerización y la orquestación de contenedores están acelerando el desarrollo y la entrega de proyectos, acelerando la creación de nuevos casos de uso en todo tipo de industrias, desde la automotriz hasta la atención médica. Independientemente de la economía, esto realmente tiene un gran impacto en la tecnología informática en general.

Las empresas están pasando del antiguo modelo de despliegue en máquinas virtuales a uno de contenedores para nuevas aplicaciones. Como presentamos brevemente en los párrafos anteriores, los contenedores podrían representarse fácilmente como una nueva forma de empaquetar aplicaciones.

Dando un paso atrás hacia las máquinas virtuales, ¿cuál era su propósito principal? Era crear un entorno aislado con un número reservado de recursos para que se ejecutara una aplicación de destino.

Con la introducción de los contenedores, las empresas se dieron cuenta de que podían optimizar mejor su infraestructura, acelerando el desarrollo y el despliegue de nuevos servicios e introduciendo cierto tipo de innovación.

Mirando hacia atrás (nuevamente) a la historia de la adopción de los contenedores y su uso, podemos ver que, al principio, se utilizaban como un método de empaquetado para los runtimes de aplicaciones monolíticas tradicionales, pero luego, una vez que surgió la ola nativa de la nube y conceptos como los microservicios se hicieron populares, los contenedores se convirtieron en el estándar *de facto* para empaquetar aplicaciones nativas de la nube de próxima generación.

> [!NOTE]
> La **computación nativa de la nube (*Cloud-native computing*)** es una práctica de desarrollo de software para crear y desplegar aplicaciones escalables en nubes públicas, privadas o híbridas.

Por otro lado, el formato de los contenedores y las herramientas de orquestación se vieron influenciados por el aumento del desarrollo y despliegue de microservicios; es por eso que hoy en día, en Kubernetes, encontramos muchos servicios y recursos adicionales, como mallas de servicios (*service meshes*) y computación sin servidor (*serverless*), que son útiles en una Arquitectura de Microservicios (MSA).

> [!NOTE]
> La **arquitectura de microservicios** es una práctica para crear aplicaciones basadas en servicios débilmente acoplados y de grano fino, utilizando protocolos ligeros.

A partir de nuestro trabajo diario con clientes que adoptan contenedores, podemos confirmar que los clientes comenzaron empaquetando solo aplicaciones estándar en contenedores y las orquestaron con un orquestador de contenedores, como Kubernetes, pero una vez que llegaron nuevos modelos de desarrollo a los equipos de desarrollo, los contenedores y sus orquestadores comenzaron a gestionar también este nuevo tipo de servicio cada vez más:

*Figura 1.6 – Arquitectura de microservicios en una aplicación real*

Solo para darnos un poco más de contexto en torno al tema de MSA, considera el diagrama anterior, donde encontramos una aplicación de tienda web simple construida con microservicios.

Como podemos ver, según el tipo de cliente que estemos utilizando (aplicación móvil o navegador web), podremos interactuar con los tres servicios subyacentes, todos desacoplados, comunicándose mediante una API REST. Una de las grandes novedades es también el desacoplamiento a nivel de datos; cada microservicio tiene su propia base de datos y estructura de datos, lo que los hace independientes en cada fase de desarrollo y despliegue.

Ahora, si asignamos un contenedor para cada microservicio que se muestra en la arquitectura y también agregamos un orquestador, como Kubernetes, ¡descubriremos que la solución está casi completa! Gracias a la tecnología de contenedores, cada servicio podría tener su propia imagen base de contenedor con solo los runtimes necesarios a bordo, lo que garantiza un paquete preconstruido ligero con todos los recursos que necesita el servicio una vez iniciado.

Por otro lado, al observar los diversos procesos automatizados en torno al desarrollo y mantenimiento de aplicaciones, una arquitectura basada en contenedores también podría adaptarse fácilmente a las herramientas de CI/CD para automatizar todos los pasos necesarios para desarrollar, probar y ejecutar una aplicación.

> [!NOTE]
> **CI/CD** significa integración continua y entrega/despliegue continuo (*continuous integration and continuous delivery/deployment*). Estas prácticas intentan cerrar la brecha entre las actividades de desarrollo y operaciones, aumentando la automatización en el proceso de construcción, prueba y despliegue de aplicaciones.

Podemos decir que la tecnología de contenedores nació para satisfacer las necesidades de los administradores de sistemas, ¡pero terminó siendo la herramienta favorita de los desarrolladores! Esta tecnología representa, en muchas empresas, el anillo de conjunción entre el equipo de desarrollo y el equipo de operaciones, lo que permitió y aceleró la adopción de prácticas de DevOps que antes estaban aisladas para aumentar la colaboración entre estos dos equipos.

> [!NOTE]
> **DevOps** es un conjunto de prácticas que vinculan el desarrollo de software (Dev) y las operaciones de TI (Ops). El objetivo de DevOps es acortar el ciclo de vida del desarrollo de una aplicación y aumentar la frecuencia de los lanzamientos de aplicaciones.

Aunque a los microservicios y a los contenedores les encanta vivir juntos, las empresas tienen una gran cantidad de aplicaciones, software y soluciones que no se basan en la arquitectura de microservicios sino en enfoques monolíticos anteriores, por ejemplo, ¡utilizando servidores de aplicaciones en clúster! Pero no tenemos que preocuparnos demasiado, ya que los contenedores y sus orquestadores evolucionaron al mismo tiempo para soportar también este tipo de carga de trabajo.

Las empresas se han encontrado cada vez más sobrecargadas por las limitaciones de las plataformas de virtualización propietarias y heredadas, con desafíos en términos de coste, escalabilidad y adaptabilidad. La adquisición de hardware especializado, las tarifas de licencia y la necesidad de experiencia dedicada contribuyen a aumentar los gastos. Además, la naturaleza intensiva en recursos de las máquinas virtuales, con sus instancias completas de sistema operativo, conduce a una utilización ineficiente de los recursos y dificulta la capacidad de escalar aplicaciones rápidamente.

Por el contrario, la tecnología de contenedores ofrece una solución atractiva a estos desafíos. Como hemos explorado, los contenedores proporcionan un medio ligero y portátil para empaquetar aplicaciones, incluidas sus dependencias, en unidades autónomas. Este enfoque simplificado no solo reduce la sobrecarga, sino que también permite un despliegue más rápido y una mayor densidad en los sistemas host, lo que se traduce en importantes ahorros de costes tanto en infraestructura como en operaciones. Además de esto, gestionar tanto máquinas virtuales como contenedores con la misma plataforma también puede ayudar a ahorrar en costes de formación y desarrollo de habilidades.

En conclusión, la convergencia de avances tecnológicos, presiones económicas y necesidades comerciales en evolución ha impulsado la contenerización a la vanguardia de las estrategias de TI empresariales. Al liberarse de las limitaciones de las plataformas de virtualización heredadas y adoptar los contenedores, las empresas pueden abrir un camino hacia una mayor agilidad, eficiencia de costes e innovación, posicionándose para el éxito en el futuro.

La tecnología de contenedores se puede considerar un formato de empaquetado de aplicaciones evolucionado que se puede optimizar para contener todas las bibliotecas y herramientas necesarias, incluso aplicaciones monolíticas complejas. Con el paso de los años, las imágenes base de contenedores han evolucionado para optimizar el tamaño y el contenido para crear runtimes más pequeños, capaces de mejorar la gestión general, incluso para aplicaciones monolíticas complejas.

Si nos fijamos en el tamaño de una imagen base de contenedor de Red Hat Enterprise Linux en su versión mínima, podemos ver que la imagen ronda los 30 MB durante la descarga y solo 84 MB una vez extraída (a través de Podman, por supuesto) en el sistema base de destino.

Incluso los orquestadores han adoptado características y recursos internos para manejar aplicaciones monolíticas, que están lejos de los conceptos nativos de la nube. Kubernetes, por ejemplo, introdujo en el núcleo de la plataforma algunas funciones para garantizar el estado (*statefulness*) de los contenedores, así como los conceptos de almacenamiento persistente para guardar datos en caché localmente o elementos importantes para la aplicación.

---

### Resumen

En este capítulo, descubrimos las funcionalidades subyacentes de la tecnología de contenedores, desde el aislamiento de procesos hasta los runtimes de contenedores. Luego, analizamos los principales propósitos y ventajas de los contenedores en comparación con las máquinas virtuales, y también las últimas novedades tecnológicas con respecto a las máquinas virtuales en contenedores para la optimización de recursos y el ahorro de costes. Después de eso, pusimos en marcha nuestra máquina del tiempo, examinando la historia de los contenedores desde 1979 hasta la actualidad. Finalmente, descubrimos las tendencias actuales del mercado y la adopción actual de contenedores en las empresas.

Este capítulo proporcionó una introducción a la tecnología de contenedores y su historia. Podman es muy similar a Docker en términos de usabilidad e interfaz de línea de comandos, y se desarrolla y mejora continuamente. El próximo capítulo cubrirá las diferencias entre los dos proyectos, desde el punto de vista arquitectónico y desde el punto de vista de la experiencia del usuario.

Después de presentar la arquitectura de alto nivel de Docker, se describirá en detalle la arquitectura sin demonio de Podman para comprender cómo este motor de contenedores puede gestionar contenedores sin necesidad de un demonio en ejecución.

---

### Lecturas adicionales

Para obtener más información sobre los temas tratados en este capítulo, consulta lo siguiente:

- [1] CNCF 2023 Annual Survey: [https://www.cncf.io/reports/cncf-annual-survey-2023/](https://www.cncf.io/reports/cncf-annual-survey-2023/)
- [2] *The Linux Programming Interface*, Michael Kerrisk (ISBN 978-1-59327-220-3)
- [3] Demystifying namespaces and containers in Linux: [https://opensource.com/article/19/10/namespaces-and-containers-linux](https://opensource.com/article/19/10/namespaces-and-containers-linux)
- [4] OCI Runtime Specs: [https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)
- [5] OCI Image Specs: [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)
- [6] Container Runtime Interface announcement: [https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)
