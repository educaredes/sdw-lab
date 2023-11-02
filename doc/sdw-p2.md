<!-- omit from toc -->
RDSV/SDNV P4 - Plataforma de orquestación de servicios basados en NFV
=====================================================================


- [Resumen](#resumen)
- [Entorno](#entorno)
- [Desarrollo de la práctica](#desarrollo-de-la-práctica)
  - [1. Instalación del entorno](#1-instalación-del-entorno)
  - [2. Definición OSM del clúster k8s y configuración de red](#2-definición-osm-del-clúster-k8s-y-configuración-de-red)
  - [3. Familiarización con la GUI de OSM](#3-familiarización-con-la-gui-de-osm)
  - [4. Repositorio de helm charts y docker](#4-repositorio-de-helm-charts-y-docker)
  - [5. (P) Relación entre helm y docker](#5-p-relación-entre-helm-y-docker)
  - [6. Instalación de descriptores en OSM](#6-instalación-de-descriptores-en-osm)
  - [7. (P) Análisis de descriptores](#7-p-análisis-de-descriptores)
  - [8. Arranque de escenarios VNX](#8-arranque-de-escenarios-vnx)
  - [9. Creación de instancias del servicio](#9-creación-de-instancias-del-servicio)
  - [10. Comprobación de los pods arrancados](#10-comprobación-de-los-pods-arrancados)
  - [11. (P) Acceso a los pods ya arrancados](#11-p-acceso-a-los-pods-ya-arrancados)
  - [12. (P) Scripts de configuración del servicio](#12-p-scripts-de-configuración-del-servicio)
  - [13. (P) Configuración del servicio](#13-p-configuración-del-servicio)
  - [14. (P) Servicio desde la red de acceso](#14-p-servicio-desde-la-red-de-acceso)
  - [15. (P) Análisis de tráfico en AccessNet1](#15-p-análisis-de-tráfico-en-accessnet1)
  - [16. (P) Análisis de tráfico en ExtNet1](#16-p-análisis-de-tráfico-en-extnet1)
  - [17. (P) Servicio para la segunda red residencial](#17-p-servicio-para-la-segunda-red-residencial)

## Resumen

En esta práctica, se va a profundizar en las funciones de red virtualizadas
(VNF) y su orquestación a través de la plataforma de código abierto [Open Source
MANO (OSM)](https://osm.etsi.org). Se va a ver cómo se despliegan funciones de
red virtualizadas mediante OSM a través de ejemplos que permitan entender
distintos procesos, como la creación de VNFs, la preparación de las plantillas
para VNFs y servicios de red y la carga (onboarding_) de los servicios de red y
sus VNFs en este tipo de plataformas NFV.

## Entorno

Para facilitar la tarea del estudiante, se utilizará la misma máquina virtual
VirtualBox de la anterior actividad práctica, en la que ya está instalado y
configurado todo el software necesario, por lo que no es necesario volver a
descargarla. La máquina ya tiene instaladas todas las herramientas necesarias,
principalmente:

El escenario explicado se va a implementar para la práctica en una máquina Linux
en VirtualBox, **RDSV-K8S**, que permite emular las distintas redes y hosts del
escenario, y el cluster de Kubernetes (K8s) de la central local. Tiene
instaladas las herramientas:
- la infraestructura de NFV (NFVI) que va a ser controlada por OSM, implementada
  mediante el paquete _microk8s_, que proporcionará la funcionalidad de un
  clúster de Kubernetes (k8s),
- scripts en la carpeta ~/bin para facilitar la gestión del entorno

Esta máquina tendrá conectividad con un servidor OSM instalado en la
infraestructura de laboratorios del DIT, a través de una red privada virtual
creada mediante la herramienta _tinc_. 

## Desarrollo de la práctica
### 1. Instalación del entorno

Para realizar la práctica debe utilizar la misma máquina virtual de la práctica 
anterior. Antes de arrancar la máquina, utilice la opción de configuración de
_Carpetas Compartidas_ para compartir una carpeta de su equipo con la máquina
virtual permanentemente, con punto de montaje `/home/upm/shared`. Asegúrese
además de configurar 4096 MB de memoria y 2 CPUs. 

A continuación, arranque la máquina,  abra un terminal y descargue en el
directorio compartido el repositorio de la práctica: 

```
cd ~/shared
git clone https://github.com/educaredes/nfv-lab.git
cd nfv-lab
```

>**Nota:**
>Si ya lo ha descargado antes puede actualizarlo con:
>
>```
>cd ~/shared/nfv-lab
>git pull
>```

Instale la red privada virtual con el servidor OSM mediante:

```
cd ~/shared/nfv-lab/bin
./install-tun <letra>
```

>**Nota:**
>El profesor asignará una \<letra\> a cada alumno de forma
>que cada clúster de k8s gestionado por el OSM central tenga una dirección IP
>distinta.

Compruebe que se ha establecido el túnel haciendo ping al servidor OSM:

```
ping 10.11.12.1
```

### 2. Definición OSM del clúster k8s y configuración de red 

Configure el entorno para acceder a OSM con su nombre de usuario y para
registrar su cluster k8s mediante:

```
cd ~/shared/nfv-lab/bin
./prepare-osmlab <letter> 
```

A continuación, **cierre el terminal**y abra uno nuevo.

En el nuevo terminal, obtenga los valores asignados a las diferentes variables
configuradas para acceder a OSM (OSM_*) y el identificador del _namespace_ de
K8S creado para OSM (OSMNS):

```
echo "-- OSM_USER=$OSM_USER"
echo "-- OSM_PASSWORD=$OSM_PASSWORD"
echo "-- OSM_PROJECT=$OSM_PROJECT"
echo "-- OSM_HOSTNAME=$OSM_HOSTNAME"
echo "-- OSMNS=$OSMNS"
```

### 3. Familiarización con la GUI de OSM

Acceda a OSM desde la máquina virtual mediante:

```
# Acceso desde la máquina virtual
firefox 10.11.12.1 &
```

Familiarícese con las distintas opciones del menú, especialmente:
- _Packages_: gestión de las plantillas de servicios de red (NS Packages)
y VNFs. 
- _Instances_: gestión de la instancias de los servicios desplegados
- _K8s_: gestión del registro de clústeres y repositorios k8s

### 4. Repositorio de helm charts y docker

Para implementar las funciones de red virtualizadas se usarán _helm charts_, que
empaquetan todos los recursos necesarios para el despliegue de una aplicación en
un clúster de K8S. A través de la GUI registraremos el repositorio de _helm
charts_ que utilizaremos en la práctica, alojado en Github Pages.

Acceda a la opción de menú _K8s Repos_, haga clic sobre el botón
 _Add K8s Repository_ y rellene los campos con los valores:
- id: `helmchartrepo`
- type: "Helm Chart" 
- URL: `https://educaredes.github.io/nfv-lab` (NO DEBE TERMINAR EN "/")
- description: _una descripción textual del repositorio_

![new-k8s-repository-details](img/new-k8s-repository.png)

En la carpeta compartida `$HOME/shared/nfv-lab/helm` puede encontrar las
definiciones de los helm charts `accesschart` y `cpechart`, mientras que en
`$HOME/shared/nfv-lab/img` está la definición del contenedor docker único que se
va a utilizar, `educaredes/vnf-img`. Este contenedor está alojado en DockerHub,
compruébelo accediendo a [este enlace](https://hub.docker.com/u/educaredes).

### 5. (P) Relación entre helm y docker

Busque en la carpeta `helm` en qué ficheros se hace referencia al contenedor
docker. Anote el resultado para incluirlo como parte de la entrega. Puede 
utilizar:

```
grep -R "educaredes/vnf-img"
```

### 6. Instalación de descriptores en OSM

Desde el _PC anfitrión_, acceda gráficamente al directorio 
`$HOME/shared/nfv-lab/pck`. Realice el proceso de instalación de los 
descriptores de las KNFs y del servicio de red (onboarding):
- Acceda al menu de OSM Packages->VNF packages y arrastre los ficheros 
`accessknf_vnfd.tar.gz` y `cpeknf_vnfd.tar.gz`   
- Acceda al menu de OSM Packages->NS packages y arrastre el fichero 
`renes_ns.tar.gz`

### 7. (P) Análisis de descriptores

Acceda a la descripción de las VNFs/KNFs y del servicio. Para entregar como 
resultado de la práctica:
1.	En la descripción de las VNFs, identifique y copie la información referente
al helm chart que se utiliza para desplegar el pod correspondiente en el clúster
de Kubernetes.
2.	En la descripción del servicio, identifique y copie la información 
referente a las dos VNFs.

### 8. Arranque de escenarios VNX 

Acceda a _RDSV-K8S_ y compruebe que están creados los switches `AccessNet1` y
`ExtNet1` tecleando en un terminal:

```
sudo ovs-vsctl show
```

A continuación arranque el escenario de la red residencial:

```
cd /home/upm/shared/nfv-lab
sudo vnx -f vnx/nfv3_home_lxc_ubuntu64.xml -t
```

El escenario contiene dos redes residenciales, nos centraremos inicialmente en
la primera de ellas (sistemas finales h11 y h12). Compruebe en los terminales
de los hosts h11 y h12 que no tienen asignada dirección IP en la interfaz 
`eth1` mediante:

```
ifconfig eth1
```

> **Nota:**
> Los hosts tienen configurada la red de gestión VNX en la interfaz `eth0`.

Compruebe también que el cliente DHCP no les permite obtener dirección IP y que
no tienen acceso a Internet:

```
dhclient eth1
ifconfig
ping 8.8.8.8
```

Arranque también el escenario "server"

```
sudo vnx -f vnx/nfv3_server_lxc_ubuntu64.xml -t
```

Finalmente, para permitir el acceso a aplicaciones con entorno gráfico desde las
máquinas arrancadas con VNX ejecute:

```
xhost +
```

> **Nota:**
> Mostrará como salida:
>
>```
>access control disabled, clients can connect from any host
>```

### 9. Creación de instancias del servicio

Desde el terminal lanzamos los siguientes comandos:

```
export NSID1=$(osm ns-create --ns_name renes1 --nsd_name renes --vim_account dummy_vim)
echo $NSID1
```

Mediante el comando `watch` visualizaremos el estado de la instancia del 
servicio, que hemos denominado `renes1`. 

```
watch osm ns-list
```

Espere a que alcance el estado _READY_ y salga con `Ctrl+C`.

Si se produce algún error, puede borrar la instancia del servicio con el 
comando:

```
osm ns-delete $NSID1
```

Y a continuación lanzar de nuevo la creación de una nueva instancia.

Acceda a la GUI de OSM, opción NS Instances, para ver cómo también es posible
gestionar el servicio gráficamente.

### 10. Comprobación de los pods arrancados

Usaremos kubectl para obtener los pods que han arrancado en el clúster:

```
kubectl -n $OSMNS get pods
```

A continuación, defina dos variables:

```
ACCPOD=<nombre del pod de la KNF:access>
CPEPOD=<nombre del pod de la KNF:cpe>
```

### 11. (P) Acceso a los pods ya arrancados

Haga una captura del texto o captura de pantalla del resultado de los 
siguientes comandos y explique dicho resultado. ¿Qué red están utilizando 
los pods para esa comunicación?

```
kubectl -n $OSMNS exec -it $ACCPOD -- ifconfig eth0
# anote la dirección IP

kubectl -n $OSMNS exec -it $CPEPOD -- /bin/bash
# Y a continuación haga un ping a la dirección IP anotada
# Salga con exit
```

### 12. (P) Scripts de configuración del servicio

Desde el _PC anfitrión_ acceda (mediante vi, nano, gedit, ...) al contenido 
del fichero `osm_renes1.sh` utilizado para configurar la instancia renes1 del
servicio. Compare los valores utilizados con los de la figura detallada del 
escenario. Indique cuál es la dirección IP "pública" (en realidad es de un 
rango privado), que deberá usar la función NAT del CPE para dar salida al 
tráfico de la red residencial hacia Internet. 

> **Parte opcional, para hacer tras la sesión del laboratorio:** 
> Analice y describa para qué se utilizan los scripts `osm_renes_start.sh` y 
`renes_start.sh`.

### 13. (P) Configuración del servicio

Desde _RDSV-K8S_, configure el servicio renes1 mediante `osm_renes1.sh`:

```
./osm_renes1.sh
```

Analice a continuación el detalle del escenario en la Fig. 4 e indique 
qué comando(s) puede utilizar **desde _RDSV-K8S_** para comprobar si hay 
conectividad entre el servicio desplegado y el dispositivo `brg1` de la 
red residencial. Verifique que haya conectividad. 

### 14. (P) Servicio desde la red de acceso

Compruebe la configuración de red de h11 y h12 y, si no han obtenido dirección 
IP, fuerce el acceso al servidor DHCP mediante el comando:

```
dhclient eth1
```

Indique qué direcciones IP obtienen h11 y h12 en la red residencial “privada”, 
así como la dirección IP del router.

Relacione el resultado con los ficheros de configuración del contenedor docker 
`educaredes/vnf-img` incluidos en el directorio 
`$HOME/shared/nfv-lab/img/vnf-img`.

### 15. (P) Análisis de tráfico en AccessNet1

Desde _RDSV-K8S_, arranque wireshark y póngalo a capturar el tráfico en
`AccessNet1`, usando:

```
wireshark -ki brg1-e2 &
```

Desde h11 realice un ping de 5 paquetes a la dirección IP de su router, 
comprobando que funciona correctamente.

```
ping -c 5 <dir_IP_router>
```

Detenga wireshark, y guarde la captura con nombre “access1.pcapng”. Analice 
el tráfico capturado, justificando las direcciones IP que aparecen en los 
paquetes capturados.

### 16. (P) Análisis de tráfico en ExtNet1

Arranque wireshark y póngalo a capturar el tráfico en ExtNet, por ejemplo 
haciendo:

```
wireshark -ki isp1-e1 &
```

Desde h11 realice un ping de 5 paquetes a la dirección IP de s1 (10.100.3.2), 
comprobando que funciona correctamente.

```
ping -c 5 10.100.3.2
```

Detenga wireshark, y guarde la captura con nombre “ext1.pcapng”. Analice el 
tráfico capturado, justificando las direcciones IP que aparecen en 
los paquetes capturados.

Desde la consola de h11, compruebe que tiene acceso a Internet. Además de usar 
ping, puede arrancar un navegador.

```
ping -c 5 8.8.8.8
firefox www.dit.upm.es &
```

### 17. (P) Servicio para la segunda red residencial

Indique y realice los pasos necesarios para dar acceso a Internet a la 
segunda red residencial (h21, h22). Para la configuración del servicio, 
tome como punto de partida `osm_renes1.sh` y cree un nuevo script
`osm_renes2.sh`. 

Compruebe que los pasos dados funcionan correctamente:
- compruebe que h21 y h22 obtienen acceso a Internet
- compruebe que a su vez sigue funcionando la primera red residencial (h11, h12)
- indique qué direcciones IP han obtenido h21 y h22

Incluya además el contenido de `osm_renes2.sh` en la memoria de respuestas.








