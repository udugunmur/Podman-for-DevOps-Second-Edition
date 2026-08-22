# Parte 1: De la Teoría a la Práctica: Ejecutemos Nuestro Primer Contenedor con Podman

## Capítulo 3: Ejecución del Primer Contenedor

En los capítulos anteriores, analizamos la historia de los contenedores, su adopción y las diversas tecnologías que contribuyen a su difusión, al tiempo que examinamos las principales diferencias entre Docker y Podman.

Ahora es el momento de empezar a trabajar con ejemplos reales: en este capítulo, aprenderemos cómo poner en marcha Podman en tu sistema operativo Linux preferido para que podamos iniciar nuestro primer contenedor. Descubriremos los diversos métodos de instalación, todos los prerrequisitos y luego iniciaremos un contenedor.

En este capítulo, cubriremos los siguientes temas principales:

- Elección de un sistema operativo y método de instalación
- Preparación del entorno
- Ejecutar tu primer contenedor

---

### Requisitos técnicos

Tener una buena experiencia técnica en la administración de un sistema operativo Linux sería preferible para comprender los conceptos clave proporcionados en este capítulo.

Repasaremos los pasos principales para instalar software nuevo en varias distribuciones de Linux, por lo que tener algo de experiencia como administrador de sistemas Linux podría resultar útil para solucionar posibles problemas durante la instalación.

Además, algunos de los conceptos teóricos explicados en los capítulos anteriores podrían ayudarte a comprender los procedimientos descritos en este capítulo.

---

### Elección de un sistema operativo y método de instalación

Podman es compatible con diferentes distribuciones y sistemas operativos. Es muy fácil de instalar y las distintas distribuciones ahora proporcionan sus propios paquetes mantenidos que se pueden instalar con sus administradores de paquetes específicos.

En esta sección, cubriremos los diferentes pasos de instalación para las distribuciones de GNU/Linux más comunes, así como en macOS y Windows, aunque el enfoque de este libro está en entornos basados en Linux.

Como tema adicional, también aprenderemos cómo compilar Podman directamente desde el código fuente.

#### Elección entre distribuciones de Linux y otro sistema operativo

La elección entre las diferentes distribuciones del sistema operativo GNU/Linux es algo dictado por las preferencias y necesidades del usuario, que habitualmente están influenciadas por varios factores que quedan fuera del alcance de este libro.

Muchos usuarios avanzados hoy en día eligen distribuciones de Linux como sus sistemas operativos principales. Sin embargo, existe una gran proporción, especialmente entre los desarrolladores, que utiliza macOS como su sistema operativo estándar. Microsoft Windows todavía conserva la mayor cuota de mercado en estaciones de trabajo de escritorio y portátiles. En caso de que disfrutes trabajando con interfaces gráficas, no te preocupes, lo tenemos cubierto. Tenemos el [Capítulo 15](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/15), en el que repasaremos la interfaz gráfica de Podman Desktop.

Hoy en día, disponemos de un enorme ecosistema de distribuciones de Linux que han evolucionado a partir de un subconjunto más pequeño de distribuciones históricas y centrales como Debian, Fedora, Red Hat Enterprise Linux, Gentoo, Arch y openSUSE. Sitios web especializados como DistroWatch ([https://distrowatch.com](https://distrowatch.com/)) realizan un seguimiento de los numerosos lanzamientos de distribuciones basadas en Linux y BSD.

A pesar de ejecutar un kernel de Linux, las diversas distribuciones tienen diferentes enfoques arquitectónicos para el comportamiento en el espacio de usuario (*userspace*), como la estructura del sistema de archivos, las bibliotecas o los sistemas de empaquetado utilizados para entregar las versiones de software.

Otra diferencia significativa está relacionada con la seguridad y los subsistemas de control de acceso obligatorio: por ejemplo, Fedora, CentOS, Red Hat Enterprise Linux y todos sus derivados se apoyan en SELinux como su subsistema de control de acceso obligatorio. Por otro lado, Debian, Ubuntu y sus derivados se basan en una solución similar llamada AppArmor.

Podman interactúa tanto con SELinux como con AppArmor para proporcionar un mejor aislamiento del contenedor, pero las interfaces subyacentes son diferentes.

> [!NOTE]
> Todos los ejemplos y el código fuente del libro han sido escritos y probados utilizando Fedora Workstation 40 como sistema operativo de referencia.

Aquellos de ustedes que deseen reproducir un entorno lo más cercano posible al libro en su laboratorio tienen diferentes opciones:

- Utilizar el Vagrant Box de Fedora 40 ([https://app.vagrantup.com/fedora/boxes/40-cloud-base](https://app.vagrantup.com/fedora/boxes/40-cloud-base)). Vagrant es una solución de software desarrollada por HashiCorp para crear máquinas virtuales rápidas y ligeras, especialmente adecuadas para su uso en desarrollo. Consulta [https://www.vagrantup.com/](https://www.vagrantup.com/) para obtener más detalles sobre Vagrant y cómo utilizarlo en tu sistema operativo preferido.
- Descargar directamente la imagen en la nube ([https://alt.fedoraproject.org/cloud/](https://alt.fedoraproject.org/cloud/)) y crear instancias en la nube pública/privada o simplemente desplegarla en un hipervisor de tu elección.
- Instalar manualmente Fedora Workstation. En este caso, la guía de instalación oficial ([https://docs.fedoraproject.org/en-US/workstation-docs/](https://docs.fedoraproject.org/en-US/workstation-docs/)) proporciona instrucciones detalladas sobre el despliegue del sistema operativo.

Ejecutar instancias en nubes públicas es la mejor opción para los usuarios que no pueden ejecutar máquinas virtuales localmente. Proveedores como Amazon Web Services, Google Cloud Platform, Microsoft Azure y DigitalOcean también ofrecen instancias en la nube basadas en Fedora listas para usar con precios mensuales bajos para tamaños más pequeños. Los precios pueden variar con el tiempo y entre niveles, y realizar un seguimiento de ellos está más allá del propósito de este libro. Casi todos los proveedores ofrecen planes gratuitos para aprendizaje o uso básico, con niveles pequeños/micro a precios muy bajos.

Los contenedores están basados en Linux, y los diferentes motores y runtimes de contenedores interactúan con el kernel y las bibliotecas de Linux para operar. Windows también ha introducido soporte para contenedores nativos con un enfoque de aislamiento bastante cercano a los conceptos de namespaces de Linux descritos anteriormente. Sin embargo, solo las imágenes basadas en Windows pueden ejecutarse de forma nativa, y muy pocos motores de contenedores admiten la ejecución nativa.

Las mismas consideraciones son válidas para macOS: su arquitectura no está basada en Linux sino en un kernel híbrido Mach/BSD llamado XNU (acrónimo de *X is not Unix*). Por esta razón, no ofrece las características del kernel de Linux necesarias para ejecutar contenedores de forma nativa. Para cerrar esta brecha, han surgido herramientas como `container` de Apple. Escrito en Swift y optimizado específicamente para Apple Silicon, `container` permite a los usuarios ejecutar contenedores Linux como máquinas virtuales ligeras. Debido a que consume y produce imágenes compatibles con OCI, sigue siendo totalmente interoperable con registros estándar, lo que le permite descargar, compilar y enviar imágenes que funcionan en cualquier ecosistema compatible con OCI.

Tanto para Windows como para macOS, es necesaria una capa de virtualización que abstraiga la máquina Linux para ejecutar contenedores Linux nativos.

Podman ofrece funciones de cliente remoto para Windows y macOS, lo que permite a los usuarios conectarse a una máquina Linux local o remota.

Los usuarios de Windows también pueden beneficiarse de un enfoque alternativo basado en el Subsistema de Windows para Linux (WSL 2.0), una capa de compatibilidad que ejecuta una máquina virtual ligera para exponer interfaces del kernel de Linux junto con binarios del espacio de usuario de Linux, gracias al soporte de virtualización de Hyper-V.

Las siguientes secciones cubrirán los pasos necesarios para instalar Podman en las distribuciones de Linux más populares, así como en macOS y Windows.

#### Instalación de Podman en Fedora

Los paquetes de Fedora son mantenidos por su amplia comunidad y administrados con el gestor de paquetes DNF. Para instalar Podman, ejecuta el siguiente comando desde una terminal:

```bash
# dnf install -y podman
```

Este comando instala Podman y configura el entorno con archivos de configuración (cubiertos en la siguiente sección). También instala unidades de systemd para proporcionar funciones adicionales como servicios de API REST o actualizaciones automáticas de contenedores.

#### Instalación de Podman en CentOS Stream

Podman se puede instalar en CentOS Stream, con el paquete de Podman disponible en el repositorio AppStream que ya está habilitado.

Para instalar Podman, ejecuta el siguiente comando desde una terminal:

```bash
# dnf install -y podman
```

Al igual que en Fedora, este comando instala Podman y todas sus dependencias, incluidos los archivos de configuración y los archivos de unidad de systemd.

#### Instalación de Podman en RHEL

Para instalar Podman en Red Hat Enterprise Linux (RHEL) ([https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)), los usuarios deben seguir dos procedimientos diferentes en RHEL 8 y RHEL 9.

En RHEL 8, el paquete Podman está disponible como un paquete individual y también bajo un módulo dedicado llamado `container-tools`. Los módulos son conjuntos personalizados de paquetes RPM que se pueden organizar en flujos con ciclos de lanzamiento independientes:

```bash
# yum module enable -y container-tools:rhel8
# yum module install -y container-tools:rhel8
```

El módulo `container-tools` instala, junto con Podman y las bibliotecas requeridas, otras herramientas útiles que se cubrirán más adelante en este libro:

- **Skopeo**: Una herramienta para gestionar imágenes y registros OCI.
- **Buildah**: Una herramienta especializada para crear imágenes OCI personalizadas a partir de Dockerfiles y desde cero (*from scratch*).
- **CRIU**: Una utilidad para implementar la funcionalidad de punto de control/restauración (*checkpoint/restore*) para Linux.
- **Udica**: Una herramienta para generar perfiles de seguridad SELinux para contenedores.

Si no están interesados en el contenido completo del módulo, los usuarios pueden instalar únicamente el paquete Podman con el siguiente comando:

```bash
# yum install -y podman
```

En RHEL 9, no existe el módulo `container-tools`. En su lugar, los usuarios podrán instalar un metapaquete `container-tools` que incluye las mismas herramientas adicionales que el módulo. He aquí cómo instalar el metapaquete en RHEL 9 y RHEL 10:

```bash
# dnf install -y container-tools
```

Si no están interesados en el metapaquete completo, los usuarios pueden instalar el paquete individual de Podman con el siguiente comando:

```bash
# dnf install -y podman
```

#### Instalación de Podman en Rocky Linux

Rocky Linux es una distribución comunitaria que heredó de CentOS el enfoque de compatibilidad a nivel de corrección de errores (*bug-for-bug*) con Red Hat Enterprise Linux mediante la recompilación de sus fuentes. Al igual que en RHEL, Podman está disponible en esta distribución y se puede instalar con el siguiente comando:

```bash
# dnf install -y podman
```

#### (No) Instalación de Podman en Fedora CoreOS y Fedora Silverblue

El título de esta subsección es un poco una broma. La realidad es que Podman ya está instalado en ambas distribuciones y es una herramienta crucial para ejecutar cargas de trabajo en contenedores.

Las distribuciones Fedora CoreOS y Fedora Silverblue son sistemas operativos atómicos e inmutables diseñados para utilizarse en entornos de servidor/nube y de escritorio, respectivamente.

Fedora CoreOS ([https://getfedora.org/en/coreos/](https://getfedora.org/en/coreos/)) es el *upstream* de Red Hat CoreOS, el sistema operativo utilizado para ejecutar Red Hat OpenShift y el sistema operativo base de OpenShift Kubernetes Distribution (OKD), la distribución de Kubernetes basada en la comunidad utilizada como *upstream* de Red Hat OpenShift.

Fedora Silverblue ([https://silverblue.fedoraproject.org/](https://silverblue.fedoraproject.org/)) es un sistema operativo inmutable enfocado en el escritorio que tiene como objetivo proporcionar una experiencia de usuario de escritorio estable y cómoda, especialmente para desarrolladores que trabajan con contenedores.

Por lo tanto, tanto en Fedora CoreOS como en Fedora Silverblue, simplemente abre una terminal y ejecuta Podman.

#### Instalación de Podman en Debian

El paquete Podman está disponible en Debian ([https://www.debian.org/](https://www.debian.org/)) desde la versión 11, con nombre en código *Bullseye* (llamado así por el famoso caballo de juguete de las películas Toy Story 2 y 3).

Debian utiliza la utilidad de gestión de paquetes `apt-get` para instalar y actualizar paquetes del sistema.

Para instalar Podman en un sistema Debian, ejecuta el siguiente comando desde la terminal:

```bash
# apt-get -y install podman
```

El comando anterior instala el binario de Podman y sus dependencias, junto con sus archivos de configuración, unidades de systemd y páginas de manual (*man pages*).

#### Instalación de Podman en Ubuntu

Al estar construido sobre Debian, Ubuntu ([https://ubuntu.com/](https://ubuntu.com/)) se comporta de manera análoga en cuanto a la gestión de paquetes. Para instalar Podman en Ubuntu 20.10 o posterior, ejecuta los siguientes comandos:

```bash
# apt-get -y update
# apt-get -y install podman
```

Estos dos comandos actualizan los paquetes del sistema y luego instalan los binarios de Podman y las dependencias asociadas.

#### Instalación de Podman en openSUSE

La distribución openSUSE ([https://www.opensuse.org/](https://www.opensuse.org/)) está respaldada por SUSE y está disponible en dos versiones diferentes: la versión *rolling release* conocida como Tumbleweed y la distribución LTS conocida como Leap. Podman está disponible en los repositorios de openSUSE y se puede instalar con el siguiente comando:

```bash
# zypper install podman
```

El gestor de paquetes Zypper descargará e instalará todos los paquetes y dependencias necesarios.

#### Instalación de Podman en Gentoo

Gentoo ([https://www.gentoo.org/](https://www.gentoo.org/)) es una distribución ingeniosa que se caracteriza por compilar los paquetes instalados directamente en la máquina de destino con personalizaciones adicionales opcionales del usuario. Para lograr esto, utiliza el administrador de paquetes Portage, inspirado en los ports de FreeBSD.

Para instalar Podman en Gentoo, ejecuta el siguiente comando:

```bash
# emerge app-emulation/podman
```

La utilidad `emerge` descargará y compilará automáticamente las fuentes de Podman en el sistema.

#### Instalación de Podman en Arch Linux

Arch Linux ([https://archlinux.org/](https://archlinux.org/)) es una distribución de Linux *rolling release* que destaca por ser altamente personalizable. Utiliza el administrador de paquetes `pacman` para instalar y actualizar paquetes desde repositorios oficiales y personalizados por los usuarios.

Para instalar Podman en Arch Linux y distribuciones derivadas, ejecuta el siguiente comando desde la terminal:

```bash
# pacman -S podman
```

De forma predeterminada, la instalación de Podman en Arch Linux no permite contenedores sin root (*rootless*). Para habilitarlos, sigue las instrucciones oficiales de la wiki de Arch: [https://wiki.archlinux.org/title/Podman#Rootless_Podman](https://wiki.archlinux.org/title/Podman#Rootless_Podman).

#### Instalación de Podman en Raspberry Pi OS

La famosa computadora de placa única (*single-board computer*) Raspberry Pi ha alcanzado un enorme éxito entre desarrolladores, creadores y aficionados.

Ejecuta Raspberry Pi OS ([https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit](https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit)), que está basado en Debian.

La compilación `arm64` de Podman está disponible y se puede instalar siguiendo los mismos pasos descritos anteriormente para la distribución Debian.

#### Instalación de Podman en macOS

Los usuarios de Apple que desarrollan y ejecutan contenedores Linux pueden instalar y utilizar Podman como cliente remoto, mientras los contenedores se ejecutan en una máquina Linux remota. La máquina Linux también puede ser una máquina virtual ejecutada en macOS y administrada directamente por Podman.

El proyecto Podman ofrece un instalador para macOS que se puede descargar desde el sitio web [Podman.io](https://podman.io/).

Alternativamente, aunque el equipo de desarrollo de Podman no lo recomienda, los usuarios pueden instalar Podman utilizando el administrador de paquetes Homebrew ejecutando el siguiente comando desde la Terminal:

```bash
$ brew install podman
```

Una vez completada la instalación, para inicializar la máquina virtual que ejecuta el entorno Linux, ejecuta los siguientes comandos:

```bash
$ podman machine init
$ podman machine start
```

En el ejemplo anterior, el servicio Podman Machine se inicializa y se inicia. El servicio descarga y configura una máquina virtual Linux dedicada que ejecuta los contenedores sin problemas. Incluso puedes inicializarla e iniciarla conjuntamente con este comando:

```bash
$ podman machine init --now
```

Alternativamente, los usuarios pueden crear y conectarse a un host Linux externo utilizando Podman como cliente remoto.

Otro enfoque válido en macOS para crear máquinas virtuales rápidas y ligeras para uso en desarrollo es Vagrant. Cuando se crea la máquina Vagrant, los usuarios pueden aprovisionar software adicional de forma manual o automática, como Podman, y comenzar a usar la instancia personalizada mediante el cliente remoto.

#### Instalación de Podman en Windows

Ejecutar Podman en el Subsistema de Windows para Linux (también conocido como WSLv2) es una alternativa sencilla y conveniente para ejecutar contenedores en Windows. La distribución invitada se ejecuta en una máquina Podman que se puede crear automáticamente con el comando `podman machine init`. Esta solución requiere una versión reciente de Windows 10 o Windows 11 para admitir WSLv2 y un sistema capaz de admitir la virtualización (utilizada internamente por WSLv2). Como alternativa, incluso es posible utilizar Hyper-V como se documenta en la página de documentación de Podman.

Como primer paso, descarga la última versión del instalador (llamado `podman-x.y.z-setup.exe`) desde la página de versiones de GitHub ([https://github.com/containers/podman/releases/](https://github.com/containers/podman/releases/)) y ejecuta el asistente para completar el proceso de instalación.

Cuando se complete la instalación, abre una terminal y ejecuta el siguiente comando:

```powershell
PS C:\Users\User> podman machine init
```

Si WSLv2 no está instalado en el sistema, el comando también se encargará de ello y mostrará un mensaje para inicializar la instalación automatizada de la máquina Podman, que incluye la instalación de los componentes necesarios de WSLv2, reinicio(s) y la importación de la máquina Podman. Obviamente, los usuarios pueden optar por omitir la instalación automatizada de WSLv2 e instalarlo manualmente. Si WSLv2 ya está instalado, el comando simplemente descargará una distribución mínima de Fedora y la importará a WSL.

Cuando finalice la importación a WSLv2, inicia la máquina Podman:

```powershell
PS C:\Users\User> podman machine start
```

El sistema ya está listo para ejecutar contenedores en la máquina Podman ejecutada en WSLv2.

Alternativamente, para ejecutar Podman únicamente como cliente remoto, disponemos de un comando de Podman para gestionar las conexiones del sistema: `podman system connection add`, que automatiza la edición en la configuración de Podman. La forma manual (no recomendada) es descargar e instalar la última versión desde la página de lanzamientos de GitHub ([https://github.com/containers/podman/releases/](https://github.com/containers/podman/releases/)). Extrae el archivo `.zip` en una ubicación adecuada y edita el archivo `containers.conf` codificado en TOML para configurar un URI remoto para la máquina Linux o pasar opciones adicionales.

El siguiente fragmento de código muestra un ejemplo de configuración:

```toml
[engine]
remote_uri= " ssh://root@10.10.1.9:22/run/podman/podman.sock"
```

La máquina Linux remota expone Podman en un socket UNIX gestionado por una unidad de systemd. Cubriremos este tema con mayor detalle más adelante, en *Gestión de Cargas de Trabajo de Contenedores, Kubernetes e IA desde una Interfaz Gráfica*.

#### Compilación de Podman desde el código fuente (*Building Podman from source*)

Compilar una aplicación desde el código fuente tiene muchas ventajas: los usuarios pueden inspeccionar y personalizar el código antes de compilarlo, realizar compilación cruzada para diferentes arquitecturas o compilar selectivamente solo un subconjunto de binarios. También es una gran oportunidad de aprendizaje para conocer la estructura del proyecto y comprender su evolución. Por último, pero no por ello menos importante, compilar desde el código fuente permite a los usuarios obtener las últimas versiones de desarrollo con nuevas características interesantes, errores incluidos.

Los siguientes pasos asumen que la máquina de compilación es una distribución de Fedora.

Primero, debemos instalar las dependencias de compilación necesarias para compilar Podman:

```bash
# git clone https://github.com/containers/podman/ && \
cd podman
# dnf install -y builddep rpm/podman.spec
```

El comando `dnf builddep` instalará todas las dependencias de compilación necesarias declaradas en el archivo `rpm/podman.spec`. Llevará un tiempo instalar todos los paquetes y sus dependencias en cascada.

Antes de comenzar la compilación, instala las siguientes dependencias de tiempo de ejecución:

```bash
$ sudo dnf -y install \
conmon \
containers-common \
crun \
netavark \
nftables \
```

Recuerda que el formato RPM está asociado con las distribuciones Fedora/CentOS/RHEL y se gestiona mediante los administradores de paquetes `dnf` y `yum`.

Cambia al directorio del proyecto e inicia la compilación:

```bash
$ make BUILDTAGS="selinux seccomp" PREFIX=/usr
$ sudo make install PREFIX=/usr
```

El primer comando `make` compila el código fuente aplicando etiquetas de compilación específicas que habilitan la compatibilidad con SELinux y seccomp, aunque en la última versión, el proceso de compilación debería detectar automáticamente las etiquetas de compilación en función de las bibliotecas instaladas. El siguiente comando `sudo make install` instala los paquetes localmente bajo la ruta `/usr/bin`.

El proceso de compilación tardará unos minutos en completarse. Para comprobar la correcta instalación de los paquetes, simplemente ejecuta el siguiente comando:

```bash
$ podman version
Client:       Podman Engine
Version:      5.3.0-dev
API Version:  5.3.0-dev
Go Version:   go1.22.7
Git Commit:   af4b061f5383938a38910d81a0f637c478fc838f
Built:        Tue Sep 24 18:11:45 2024
OS/Arch:      linux/amd64
```

Para crear una versión binaria similar al archivo `.tar.gz` que está disponible en la página de versiones de GitHub, ejecuta el siguiente comando:

```bash
$ make podman-release
```

Consejo adicional: Compilar una versión diferente es muy fácil: simplemente cambia a la etiqueta de la versión de destino utilizando el comando `git`. Por ejemplo, para compilar `v5.3.0`, utiliza el siguiente comando:

```bash
$ git checkout v5.3.0
```

En esta sección, aprendimos cómo instalar versiones binarias de Podman en diferentes distribuciones utilizando sus respectivos administradores de paquetes. También aprendimos a instalar el cliente remoto de Podman en macOS y Windows, junto con el modo Windows WSLv2. Cerramos esta sección mostrándote cómo compilar desde el código fuente.

A continuación, aprenderemos a configurar Podman para la primera ejecución preparando el entorno del sistema.

---

### Preparación del entorno

Una vez que se han instalado los paquetes de Podman, Podman está listo para usarse directamente. Sin embargo, algunas personalizaciones menores pueden ser útiles para proporcionar una mejor interoperabilidad con registros externos o para personalizar los comportamientos en tiempo de ejecución.

#### Personalización de la lista de búsqueda de registros de contenedores

Podman busca y descarga imágenes de una lista de registros de contenedores confiables. El archivo `/etc/containers/registries.conf` es un archivo de configuración en formato TOML que se puede utilizar para personalizar los registros en lista blanca a los que se les permite buscar y utilizar como fuentes de imágenes, así como la duplicación de registros (*registry mirroring*) y registros no seguros sin terminación TLS. En este archivo de configuración, la clave `unqualified-search-registries` se completa con una matriz de registros no calificados sin especificaciones sobre repositorios de imágenes y etiquetas.

En un sistema Fedora, con una instalación nueva de Podman, esta clave tiene el siguiente contenido:

```toml
unqualified-search-registries = ["registry.fedoraproject.org", "registry.access.redhat.com", "docker.io", "quay.io"]
```

Los usuarios pueden agregar o eliminar registros de esta matriz para permitir que Podman busque y descargue desde ellos.

> [!IMPORTANT]
> Sé muy prudente al agregar registros y utiliza solo registros confiables para evitar descargar imágenes que contengan código malicioso.

La lista predeterminada es adecuada para buscar y ejecutar todos los ejemplos del libro. Aquellos de ustedes que ya estén ejecutando registros privados pueden intentar agregarlos a la matriz de registros de búsqueda no calificados.

Dado que los registros son tanto privados como públicos, ten en cuenta que los registros privados generalmente requieren autenticación adicional para poder acceder a ellos. Esto se puede lograr con el comando `podman login`, que se tratará en el [Capítulo 9](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/9), *Publicación de Imágenes en un Registro de Contenedores*.

Si el archivo `$HOME/.config/containers/registries.conf` se encuentra en el directorio de inicio del usuario, anula el archivo `/etc/containers/registries.conf`. De esta manera, diferentes usuarios en el mismo sistema podrán ejecutar Podman con sus listas blancas y espejos de registros personalizados.

#### Opcional – Habilitar servicios basados en sockets

Este es un paso opcional y, en ausencia de necesidades específicas, el contenido de esta sección se puede omitir de manera segura.

Como mencionamos anteriormente, Podman es un administrador de contenedores sin demonio que no necesita ningún servicio en segundo plano para ejecutar contenedores. Sin embargo, los usuarios pueden necesitar interactuar con las API de Libpod expuestas por Podman y, además de eso, Podman también tiene API compatibles con Docker que se pueden usar para la transición, especialmente útiles al migrar desde un entorno basado en Docker.

Podman puede exponer sus API mediante un socket UNIX (comportamiento predeterminado) o un socket TCP. Esta última opción es menos segura porque hace que Podman sea accesible desde el mundo exterior, pero es necesaria en algunos casos, como cuando se debe acceder a él mediante un cliente de Podman en una estación de trabajo Windows o macOS. Antes de hacer esto, evalúa también las posibilidades de mediar el servicio a través del acceso y protocolo SSH.

> [!WARNING]
> Ten cuidado al ejecutar el servicio de API utilizando un endpoint TCP en una máquina expuesta a Internet, ya que el servicio será accesible globalmente.

El siguiente comando expone las API de Podman en un socket UNIX:

```bash
$ sudo podman system service --time=0 \
unix:///run/podman/podman.sock
```

Después de ejecutar este comando, los usuarios pueden conectarse al servicio de API.

Tener que ejecutar este comando en una ventana de terminal no es un enfoque práctico. En su lugar, el mejor enfoque es utilizar un socket de systemd (ver `man systemd.socket`).

Las unidades de socket en systemd son tipos especiales de activadores de servicios: cuando una solicitud llega al endpoint predefinido del socket, systemd genera inmediatamente el servicio homónimo.

Cuando se instala Podman, se crean los archivos de unidad `podman.socket` y `podman.service`. `podman.socket` tiene el siguiente contenido:

```ini
# /usr/lib/systemd/system/podman.socket
[Unit]
Description=Podman API Socket
Documentation=man:podman-system-service(1)

[Socket]
ListenStream=%t/podman/podman.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
```

La clave `ListenStream` contiene la ruta relativa del socket, que se expande a `/run/podman/podman.sock`. El archivo de unidad de systemd `podman.service` completo tiene el siguiente contenido:

```ini
# /usr/lib/systemd/system/podman.service
[Unit]
Description=Podman API Service
Requires=podman.socket
After=podman.socket
Documentation=man:podman-system-service(1)
StartLimitIntervalSec=0

[Service]
Type=exec
KillMode=process
Environment=LOGGING="--log-level=info"
ExecStart=/usr/bin/podman $LOGGING system service

[Install]
WantedBy=multi-user.target
```

El campo `ExecStart=` indica el comando que lanzará el servicio, que es el mismo comando `podman system service` que mostramos anteriormente. El campo `Requires=` indica que la unidad `podman.service` necesita `podman.socket` para activarse.

Entonces, ¿qué sucede cuando habilitamos e iniciamos la unidad `podman.socket`? systemd maneja el socket y espera una conexión al endpoint del socket. Cuando ocurre este evento, inicia inmediatamente la unidad `podman.service`. Después de un período de inactividad, el servicio se detiene nuevamente.

Para habilitar e iniciar la unidad de socket, ejecuta el siguiente comando:

```bash
# systemctl enable --now podman.socket
```

Podemos probar los resultados con un comando `curl` simple:

```bash
# curl --unix-socket /run/podman/podman.sock \
http://d/v3.0.0/libpod/info
```

La salida impresa será una carga útil JSON que contiene la configuración del motor de contenedores.

¿Qué sucedió cuando accedimos a la URL? Bajo el capó, la unidad de servicio se inició inmediatamente e interactuó con el socket cuando se estableció la conexión. Algunos de ustedes habrán notado un ligero retraso (del orden de una décima de segundo) la primera vez que se ejecutó el comando.

Después de 5 segundos de inactividad, `podman.service` se desactiva nuevamente. Esto se debe al comportamiento predeterminado del comando `podman system service`, que se ejecuta solo durante 5 segundos de forma predeterminada, a menos que se pase la opción `--time` para proporcionar un tiempo de espera diferente (un valor de `0` significa para siempre).

#### Opcional – Personalizar el comportamiento de Podman

La configuración predeterminada de Podman funciona de forma predeterminada para la mayoría de los casos de uso, pero su configuración es muy flexible. Los siguientes archivos de configuración están disponibles para personalizar su comportamiento:

- `containers.conf`: Este archivo con formato TOML contiene las configuraciones de runtime de Podman, así como las rutas de búsqueda de los binarios de `conmon` y del runtime de contenedores. Se instala por defecto bajo la ruta `/usr/share/containers/` y puede ser anulado por los archivos `/etc/containers/containers.conf` y `$HOME/.config/containers/containers.conf` para configuraciones a nivel de todo el sistema y a nivel de usuario, respectivamente. Este archivo se puede utilizar para personalizar el comportamiento del motor. Los usuarios pueden influir en cómo se crea el contenedor y su ciclo de vida personalizando configuraciones como el registro (*logging*), la resolución DNS, las variables de entorno, el uso de memoria compartida, la administración de cgroups y muchas otras. Para obtener una lista completa de configuraciones, consulta la página de manual correspondiente, que se instaló junto con el paquete de Podman (`man containers.conf`).
- `storage.conf`: Este archivo con formato TOML se utiliza para personalizar los ajustes de almacenamiento que utiliza el motor de contenedores. En particular, este archivo te permite personalizar el controlador de almacenamiento predeterminado, así como el directorio de lectura/escritura del almacenamiento de contenedores (también conocido como *graph root*), que es una opción adicional de almacenamiento del controlador. De forma predeterminada, el controlador está configurado en `overlay`. La ruta predeterminada de este archivo es `/usr/share/containers/storage.conf` y las anulaciones se pueden encontrar o crear en `/etc/containers/storage.conf` para personalizaciones en todo el sistema. Las configuraciones a nivel de usuario que afectan a los contenedores sin root se pueden encontrar en `$XDG_CONFIG_HOME/containers/storage.conf` o `$HOME/.config/containers/storage.conf`.
- `mounts.conf`: Este archivo define los montajes de volumen que deben montarse automáticamente dentro de un contenedor cuando se inicia. Esto es útil, por ejemplo, para pasar automáticamente secretos como claves y certificados dentro de un contenedor. Se puede encontrar en `/usr/share/containers/mounts.conf` y anularse mediante un archivo ubicado en `/etc/containers/mounts.conf`. En el modo rootless, el archivo de anulación se puede colocar en `$HOME/.config/containers/mounts.conf`.
- `seccomp.json`: Este es un archivo JSON que permite a los usuarios personalizar las llamadas al sistema permitidas que puede realizar un proceso dentro de un contenedor y definir las bloqueadas al mismo tiempo. Este tema se tratará nuevamente en el [Capítulo 10](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/10), *Aseguramiento de Contenedores*, que proporcionará una comprensión más profunda de las restricciones de seguridad de los contenedores. La ruta predeterminada para este archivo es `/usr/share/containers/seccomp.json`. La página de manual de seccomp (`man seccomp`) proporciona una descripción general de cómo funciona seccomp en un sistema Linux.
- `policy.json`: Este es un archivo JSON que define cómo Podman realizará la verificación de firmas. La ruta predeterminada de este archivo es `/etc/containers/policy.json` y puede ser anulada por `$HOME/.config/containers/policy.json` a nivel de usuario. Este archivo de configuración acepta tres tipos de políticas:
  - `insecureAcceptAnything`: Acepta cualquier imagen del registro especificado.
  - `reject`: Rechaza cualquier imagen del registro especificado.
  - `signedBy`: Acepta solo imágenes firmadas por una entidad específica y conocida.

La configuración predeterminada es aceptar cada imagen (la política `insecureAcceptAnything`), pero se puede modificar para descargar solo imágenes confiables que puedan verificarse mediante una firma. Los usuarios pueden definir claves GPG personalizadas para verificar las firmas y la identidad que las firmó. Para obtener detalles adicionales sobre las posibles políticas y ejemplos de configuración, consulta la página de manual correspondiente (`man containers-policy.json`).

En esta sección, discutimos algunas configuraciones básicas de Podman que es útil conocer cuando se instala Podman por primera vez. En la siguiente sección, cubriremos nuestros primeros ejemplos de ejecución de contenedores.

---

### Running your first container (Ejecución de tu primer contenedor)

Ahora es el momento de ejecutar finalmente nuestro primer contenedor.

En la sección anterior, descubrimos cómo instalar Podman en nuestra distribución de Linux favorita, así como qué se incluye en los paquetes base una vez instalados. Ahora podemos empezar a utilizar nuestro motor de contenedores sin demonio.

La ejecución de contenedores en Podman se gestiona mediante el comando `podman run`, que acepta muchas opciones para controlar el comportamiento del contenedor recién ejecutado, su aislamiento, su comunicación, su almacenamiento, etc.

El comando de Podman más fácil y corto para ejecutar un contenedor completamente nuevo es el siguiente:

```bash
$ podman run <imageID>
```

Tenemos que reemplazar la cadena `<imageID>` con el nombre/ubicación/etiqueta de la imagen que queremos ejecutar. Si la imagen no está presente en la caché o no la hemos descargado antes, Podman descargará la imagen por nosotros desde el registro de contenedores respectivo.

#### Shell interactiva y pseudo-tty

Para presentar este comando y sus opciones, comencemos de forma sencilla y ejecutemos el siguiente comando:

```bash
$ podman run -i -t fedora /bin/bash
Resolved "fedora" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull registry.fedoraproject.org/fedora:latest...
Getting image source signatures
Copying blob dbe590460ba2 done   | 
Copying config 5eea52ff5b done   | 
Writing manifest to image destination
[root@a45628efe4b8 /]
```

Veamos qué hizo Podman una vez que ejecutamos el comando anterior:

1. Reconoció el nombre de la imagen, `fedora`, como un alias para la última imagen de contenedor de Fedora.
2. Luego se dio cuenta de que la imagen faltaba en la caché local porque era la primera vez que intentábamos ejecutarla.
3. Descargó la imagen del registro correcto. Eligió el registro del Proyecto Fedora porque coincidía con los alias contenidos en las configuraciones de los registros.
4. Finalmente, inició el contenedor y nos presentó una shell interactiva, ejecutando el programa de shell Bash que solicitamos.

El comando anterior generó una shell interactiva gracias a las dos opciones que podemos analizar a continuación:

- `--tty`, `-t`: Con esta opción, Podman asigna un pseudo-tty (ver `man pty`) y lo conecta a la entrada estándar (*standard input*) del contenedor.
- `--interactive`, `-i`: Con esta opción, Podman mantiene abierto `stdin` y listo para conectarse al pseudo-tty anterior.

Como se indicó en los capítulos anteriores, cuando se crea un contenedor, los procesos aislados dentro de él se ejecutarán en un sistema de archivos raíz editable, como resultado de una superposición por capas (*layered overlay*).

Esto permite que cualquier proceso escriba archivos, pero no olvides que durarán mientras el contenedor esté en funcionamiento, ya que los contenedores son efímeros de forma predeterminada.

Ahora puedes ejecutar cualquier comando y verificar su salida en la consola que acabamos de abrir:

```bash
[root@a45628efe4b8 /]# dnf install -y iputils iproute
...
Installing:
 iproute      x86_64     6.7.0-2.fc40     updates     813 k
 iputils      x86_64     20240117-4.fc4   fedora      194 k
...
[root@a45628efe4b8 /]# ip r
default via 192.168.124.1 dev eth0 proto dhcp metric 100 
192.168.124.0/24 dev eth0 proto kernel scope link metric 100 
[root@a45628efe4b8 /]# ping -c2 192.168.124.1
PING 192.168.124.1 (192.168.124.1) 56(84) bytes of data.
64 bytes from 192.168.124.1: icmp_seq=1 ttl=255 time=0.354 ms
64 bytes from 192.168.124.1: icmp_seq=2 ttl=255 time=0.516 ms
...
```

Como puedes ver en el ejemplo anterior, acabamos de instalar dos paquetes para inspeccionar la configuración de red del contenedor y luego ejecutar un ping al enrutador predeterminado asignado a la red virtual de nuestro contenedor en ejecución. Nuevamente, si detenemos este contenedor, cualquier cambio se perderá.

Para salir de esta shell interactiva, simplemente podemos presionar `Ctrl + D` o ejecutar el comando `exit`. Al hacer esto, el contenedor terminará porque el proceso principal en ejecución que solicitamos ejecutar (`/bin/bash`) se detendrá.

Ahora, veamos otras opciones interesantes y útiles que podemos usar con el comando `podman run`.

#### Desacoplarse de un contenedor en ejecución (*Detaching from a running container*)

Como aprendimos anteriormente, Podman nos brinda la oportunidad de conectar una shell interactiva a nuestro contenedor en ejecución. Sin embargo, pronto descubriremos que esta no es la forma preferida de ejecutar nuestros contenedores.

Una vez que se ha iniciado un contenedor, podemos desacoplarnos de él fácilmente, incluso si lo iniciamos con una tty interactiva conectada:

```bash
$ podman run -i -t quay.io/fedora/httpd-24
Trying to pull quay.io/fedora/httpd-24:latest...
Getting image source signatures
Copying blob 38cf9b2fea0d done   | 
Copying blob 94fb3cd520fe done   | 
Copying blob f0e7a10e7819 done   | 
Copying config 6a844fa80d done   | 
Writing manifest to image destination
=> sourcing 10-set-mpm.sh ...
=> sourcing 20-copy-config.sh ...
=> sourcing 40-ssl-certs.sh ...
---> Generating SSL key pair for httpd...
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.124.6. Set the 'ServerName' directive globally to suppress this message
...
[Tue Sep 24 11:02:20.626683 2024] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.62 (Fedora Linux) OpenSSL/3.2.2 configured -- resuming normal operations
[Tue Sep 24 11:02:20.626700 2024] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
```

¿Y ahora qué? Para desacoplarnos de nuestro contenedor en ejecución, solo tenemos que presionar estos atajos de teclado especiales: `Ctrl + P`, `Ctrl + Q`. Con esta secuencia, volveremos a nuestro prompt de la shell mientras el contenedor seguirá ejecutándose.

Para recuperar la tty de nuestro contenedor desacoplado, debemos obtener la lista de contenedores en ejecución:

```bash
$ podman ps
CONTAINER ID  IMAGE                                      COMMAND               CREATED        STATUS            PORTS       NAMES
685a339917e7  registry.fedoraproject.org/httpd:latest    /usr/bin/run-http...  3 minutes ago  Up 3 minutes ago              clever_zhukovsky
```

Exploraremos este comando con más detalle en el próximo capítulo, pero por el momento, solo toma nota del ID del contenedor y luego ejecuta el siguiente comando para volver a conectarte a la tty anterior:

```bash
$ podman attach 685a339917e7
```

Ten en cuenta que podemos iniciar fácilmente un contenedor en modo desacoplado simplemente agregando la opción `-d` a `podman run`, de la siguiente manera:

```bash
$ podman run -d registry.fedoraproject.org/httpd
```

En la siguiente sección, aprenderemos a utilizar la opción desacoplada para propósitos especiales.

#### Publicación de puertos de red (*Network port publishing*)

Como mencionamos en los capítulos anteriores, Podman, al igual que cualquier otro motor de contenedores, conecta una red virtual a un contenedor en estado de ejecución que ha sido aislado de la red del host original. Por esta razón, si queremos llegar fácilmente a nuestro contenedor o incluso exponerlo fuera de la red de nuestro host, debemos indicarle a Podman que realice el mapeo de puertos (*port mapping*).

La opción `-p` de Podman publica el puerto de un contenedor en el host:

```text
-p=ip:hostPort:containerPort/protocol
```

Tanto `hostPort` como `containerPort` pueden ser un rango de puertos, y si la IP del host no está configurada o está configurada en `0.0.0.0`, entonces el puerto se vinculará a todas las direcciones IP del host, y si se establece en `[::]`, nos vincularemos a todas las direcciones IPv6. Finalmente, el protocolo es un campo opcional que podría ser útil para restringir aún más el acceso a conexiones TCP o UDP.

Si retomamos el comando que usamos en la sección anterior, se convierte en el siguiente:

```bash
$ podman run -p 8080:8080 -d registry.fedoraproject.org/httpd
```

Ahora podemos tomar nota del ID de contenedor que se le ha asignado a nuestro contenedor en ejecución:

```bash
$ podman ps
CONTAINER ID  IMAGE                                      COMMAND               CREATED         STATUS             PORTS                   NAMES
fc9d97642801  registry.fedoraproject.org/httpd:latest    /usr/bin/run-http...  10 minutes ago  Up 10 minutes ago  0.0.0.0:8080->8080/tcp  confident_snyder
```

Luego, podemos observar el mapeo de puertos que acabamos de definir:

```bash
$ podman port fc9d97642801
8080/tcp -> 0.0.0.0:8080
```

A continuación, podemos probar si este mapeo de puertos funciona utilizando `curl`, un cliente web HTTP fácil de usar. Alternativamente, puedes apuntar tu navegador web favorito a la misma URL, de la siguiente manera:

```bash
$ curl -s 127.0.0.1:8080 | head
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                               Dload  Upload   Total   Spent    Left  Speed
100  4650  100  4650    0     0   100k      0 --:--:-- --:--:-- --:--:--  100k
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
<title>Test Page for the Apache HTTP Server on Fedora</title>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<style type="text/css">
/*<![CDATA[*/
body {
    background-color: #fff;
```

Antes de concluir este capítulo, echemos un vistazo a otras opciones interesantes que podrían resultar útiles para gestionar la configuración y el comportamiento del contenedor en tiempo de ejecución.

#### Configuración y variables de entorno

El comando `podman run` tiene una gran cantidad de opciones para permitirnos configurar el comportamiento del contenedor en tiempo de ejecución: estamos hablando de alrededor de 120 opciones al momento de escribir este libro.

Por ejemplo, tenemos una opción para cambiar la zona horaria de nuestros contenedores en ejecución: `--tz`:

```bash
$ date
Wed Sep 25 09:57:25 AM UTC 2024
$ podman run --tz=Asia/Shanghai fedora date
Wed Sep 25 17:57:33 CST 2024
```

Podemos cambiar el DNS de nuestro nuevo contenedor con la opción `--dns`:

```bash
$ podman run --dns=1.1.1.1 fedora cat /etc/resolv.conf
nameserver 1.1.1.1
```

También podemos agregar un host al archivo `/etc/hosts` para anular una dirección interna local:

```bash
$ podman run --add-host=my.server.local:192.168.1.10 \
fedora cat /etc/hosts
192.168.1.10	my.server.local
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.124.6	3497f23d8551	serene_shannon
```

Incluso podemos agregar un proxy HTTP para permitir que nuestro contenedor use un proxy para solicitudes HTTP. El comportamiento predeterminado de Podman es pasar muchas variables de entorno desde el host, algunas de las cuales son `http_proxy`, `https_proxy`, `ftp_proxy` y `no_proxy`.

Por otro lado, también podemos definir variables de entorno personalizadas que podemos pasar a nuestro contenedor gracias a la opción `--env`:

```bash
$ podman run --env MYENV=podman fedora printenv
container=oci
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MYENV=podman
HOME=/root
HOSTNAME=1db6008431c2
```

Agregar y usar variables de entorno con nuestros contenedores es una mejor práctica para pasar parámetros de configuración a la aplicación e influir en el comportamiento del servicio desde el host del sistema operativo. Como vimos en el [Capítulo 1](https://subscription.packtpub.com/book/cloud-and-networking/9781835886625/1), los contenedores son inmutables y efímeros de forma predeterminada. Por esta razón, debemos aprovechar las variables de entorno, como lo hicimos en el ejemplo anterior, para configurar un contenedor en tiempo de ejecución.

---

### Resumen

En este capítulo, comenzamos a interactuar con los comandos básicos de Podman, aprendimos a ejecutar un contenedor observando las opciones más interesantes disponibles y ahora estamos listos para pasar al siguiente nivel: la gestión de contenedores. Para trabajar como administrador de sistemas en el mundo de los contenedores, debemos comprender y aprender los comandos de gestión que nos permiten inspeccionar y verificar el estado de salud de nuestros servicios contenedorizados en ejecución; eso es lo que vimos en este capítulo.

En el próximo capítulo, que está profundamente enfocado en la gestión de contenedores, aprenderemos a gestionar el ciclo de vida de imágenes y contenedores con Podman. Aprenderemos a inspeccionar y extraer registros de contenedores en ejecución y también presentaremos los pods, cómo crearlos y cómo ejecutar contenedores dentro de ellos.

---

### Lecturas adicionales

Para obtener más información sobre los temas tratados en este capítulo, puedes consultar los siguientes recursos:

- Instalar Podman en macOS: [https://podman.io/blogs/2021/09/06/podman-on-macs.html](https://podman.io/blogs/2021/09/06/podman-on-macs.html)
- Instalar Podman en Windows: [https://www.redhat.com/sysadmin/podman-windows-wsl2](https://www.redhat.com/sysadmin/podman-windows-wsl2)
- Documentación de Podman para Windows: [https://podman-desktop.io/docs/installation/windows-install](https://podman-desktop.io/docs/installation/windows-install) y [https://github.com/containers/podman/blob/main/docs/tutorials/podman-for-windows.md](https://github.com/containers/podman/blob/main/docs/tutorials/podman-for-windows.md)
- Gestionar registros de contenedores: [https://www.redhat.com/sysadmin/manage-container-registries](https://www.redhat.com/sysadmin/manage-container-registries)
- Documentación de la API de Podman: [https://docs.podman.io/en/latest/_static/api.html](https://docs.podman.io/en/latest/_static/api.html)
- Manual de sockets de systemd: [https://www.freedesktop.org/software/systemd/man/systemd.socket.html](https://www.freedesktop.org/software/systemd/man/systemd.socket.html)
- Podman y perfiles de seccomp: [https://podman.io/blogs/2019/10/15/generate-seccomp-profiles.html](https://podman.io/blogs/2019/10/15/generate-seccomp-profiles.html)
