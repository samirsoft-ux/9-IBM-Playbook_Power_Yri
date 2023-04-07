# IBM Power Virtual Servers ☁️ - Conexión VPN On-Premise 🗄️

## 📃 Introducción
Power Systems Virtual Server ***PowerVS*** tiene servicio VPNaaS, pero existen algunas limitaciones, como se describe en la documentación ["Limitaciones VPNaaS de PowerVS"](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-VPN-connections). Es por lo que, en este artículo, me gustaría enseñarte a cómo acceder a PowerVS usando una VPN site-to-site que se puede usar en VPC, en lugar de la VPNaaS de PowerVS.

### Los puntos clave a tener en cuenta antes de empezar con la guía son:
- PowerVS no conoce la información de enrutamiento al rango de direcciones IP de la infraestructura local, por lo que no es posible enviar paquetes desde PowerVS a la red local.
- La clave de esta configuración es definir el rango de direcciones IP de la red local como el prefijo de la VPC. Esta definición permite que la información de enrutamiento para el rango de IP local se anuncie a PowerVS a través de conexiones en la nube (Direct Link 2.0) entre la VPC y PowerVS, lo que permite que PowerVS envíe paquetes para el rango de IP local a la VPC.
- Defina una tabla de enrutamiento de entrada en la VPC para que los paquetes de PowerVS a la VPC (destinados a la IP local) se enruten a la puerta de enlace VPN.
- La puerta de enlace VPN pasa los paquetes a las instalaciones a través del túnel VPN, lo que permite la comunicación de un extremo a otro.

Gaaa
<p align="center"><img width="800" src="https://github.com/samirsoft-ux/Playbook_Power/blob/main/Imagenes/IS-arqui-power.png"></p>



<br />

## 📑 Índice  
1. [Pre-Requisitos](#pencil-Pre-Requisitos)
2. [Datos de Configuración de las subredes](#cloud-Datos-de-Configuración-de-las-subredes)
3. [Creación del PowerVS location](#👷🏻Creación-del-PowerVS-location)
   * [Creación de las subredes privadas](#🕸️Creación-de-las-subredes-privadas)
4. [Aprovisionar IBM i o AIX VSI's en cada PowerVS location](#computer-Aprovisionar-IBM-i-o-AIX-VSI's-en-cada-PowerVS-location)
5. [Despliegue de Direct Link 2.0](#cloud-Despliegue-de-cloud-Direct-Link-2.0)
6. [Desplegar y configurar un Vyatta en cada datacenter](#wrench-Desplegar-y-configurar-un-Vyatta-en-cada-datacenter)
7. [Configurar los túneles GRE en cada PowerVS location](#gear-Configurar-los-túneles-GRE-en-cada-PowerVS-location)
8. [Configurar los túneles IPSec entre los dos Vyattas](#hammer_and_wrench-Configurar-los-túneles-IPSec-entre-los-dos-Vyattas)
9. [Configurar un servidor proxy en cada datacenter (Opcional)](#gear-Configurar-un-servidor-proxy-en-cada-datacenter-(Opcional))
10. [Referencias](#mag-Referencias)
11. [Autores](#black_nib-Autores)
<br />

## :pencil: Pre-Requisitos 
* Contar con una cuenta facturable en <a href="https://cloud.ibm.com/"> ***IBM Cloud®*** </a>.
<br />

## :cloud: Datos de Configuración de las subredes 

### SUBRED 1
| ***DATACENTER*** | ***CIDR*** |
|     :---:      |     :---:      |
| Montreal 01  | 192.168.3.0/24 |
<br />

### SUBRED 2
| ***DATACENTER*** | ***CIDR*** |
|     :---:      |     :---:      |
| Washington DC 06  | 192.168.4.0/24 |
<br />

## 👷🏻Creación-del-PowerVS-location
El primer paso consiste en crear el *PowerVS location* en cada uno de los datacenters. 
Para ello, realice los pasos que se muestran a continuación:

1. En el apartado de ```Catálogo``` buscar la opción de ```Power Systems Virtual Server```, dar click y aparecerá la ventana para la creación del *Power VS Location*, complete lo siguiente:
* ```Seleccione una ubicación```: Seleccionar la ubicación del datacenter donde queremos realizar el despliegue. (Para ejemplo del tutorial: Montreal 01 & Washington DC 01)
* ```Seleccione un plan de precios```: Seleccione uno de los planes disponibles que se adecué a sus requerimientos.

<br />
<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/conf%20power%20location.png"></p>

A continuación complete los campos para finalizar con la configuración del recurso:
<br />

**Configurar su recurso:**
* ```Nombre de Servicio```: Asigne un nombre exclusivo para su PowerVS location. (Preferible relacionado con la ubicación de su datacenter)
* ```Seleccione un grupo de recursos```: Escoja el grupo de recursos con el que trabajará la solución de Disaster Recovery.
* ```Etiquetas```: Asigne etiquetas como buena práctica para reconocer el recurso - Opcional (Ejemplo: owner:yvy).
* ```Etiquetas de gestión de acceso```: Asigne etiquetas como buena práctica para gestionar el recurso - Opcional (Ejemplo: proy:powervs).
<br />

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/conf%20recurso.png"></p>

Cuando ya tenga todos los campos configurados de click en el botón ```Crear```.
<br />

>NOTA: Repita el mismo procedimiento para implementar el PowerVS location del segundo datacenter.

Espere unos minutos mientras el *PowerVS location* es desplegado y asegúrese de tener seleccionada la región en la cual lo implementó.
<br />

>NOTA: Luego de crear el *PowerVS location* deberá completar los siguientes procedimientos:
   * [Creación de las subredes privadas](#🕸️Creación-de-las-subredes-privadas)
<br />

## 🕸️Creación de las subredes privadas
Una vez ha creado los *PowerVS location*, se deben configurar las subredes privadas, de acuerdo a los datos de configuración especificados en la tabla [Datos de Configuración de las subredes](#cloud-Datos-de-Configuración-de-las-subredes), para esto complete los siguientes pasos:
1. En la sección de ```Lista de recursos``` seleccione la opción ```Servicios y Software``` y ubique el recurso del PowerVS implementado, posteriormente ingrese a la sección ```Subredes``` y darle click en ```Crear```. Una vez le aparezca la ventana para la configuración y creación de la *subred privada*, complete lo siguiente:  
* ```Nombre```: Asigne un nombre exclusivo para la *subred privada*.

El siguiente parámetro se escoge de acuerdo a lo especificado en la tabla [Creación de las subredes privadas](#🕸️Creación-de-las-subredes-privadas):
* ```CIDR```: 172.16.0.0/24 (Para el caso de Montreal 01) 
Los parámetros a continuación se autocompletarán una vez ingresado el valor del CIDR:
* ```Pasarela```: 172.16.0.1
* ```Rangos de IP```: 172.16.0.2 - 172.16.0.254
* ```Servidor DNS```: 127.0.0.1
* ``` Conexión Cloud (opcional) ```: Seleccionar una conexión cloud existente si la hubiera.
>Nota: En el siguiente paso se creará una conexión cloud para cada PowerVS location.


Cuando ya tenga todos los campos configurados de click en el botón ```Crear subred```.
<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/conf%20subred.png"></p>

2. Espere unos minutos mientras la *subred* es desplegada y asegúrese de tener seleccionada la región en la cual la implementó.
<br />

>Nota: Implemente la subred privada para el PowerVS Location del otro datacenter ( Washington DC 01 para el caso de este tutorial).

<br />

## :computer: Aprovisionar IBM i o AIX VSI's en cada PowerVS location
En el paso 1 se desplegó los PowerVS location en cada datacenter, ahora se hará el despliegue de las instancias de PowerVS en cada location implementado.
Para ello se ubicará en la sección de ```Lista de recursos``` seleccione la opción ```Servicios y Software``` y ubique el recurso del PowerVS location implementado, posteriormente ingrese a la sección ```Instancias de Servidor Virtual``` y darle click en ```Crear Instancia```. 
Una vez le aparezca la ventana para la configuración y creación de la *instancia de servidor virtual*, complete lo siguiente:  
* ```Nombre de instancia```: Asigne un nombre exclusivo para su *instancia PowerVS*.
* ```Número de instancias```: Se pueden desplegar hasta 5 instancias en un mismo despliegue. Para fines del tutorial solo desplegaremos una instancia.
* ```Anclaje de máquinas virtuales```: Permite controlar el movimiento de las máquinas virtuales durante sucesos como reinicio. Para fines del tutorial usaremos: ```ninguno```
* ```Grupo de colocación```: Los grupos de colocación proporcionan control sobre el host en el que se coloca una VM. El host lo determina la política de colocación del grupo, un servidor distinto o el mismo servidor. Para fines del tutorial usaremos: ```ninguno```

**Clave SSH:**
Añadiremos una clave SSH pública para poder conectarnos de manera segura a nuestra máquina virtual:
* ```Elegir una clave SSH```: Elegir una clave SSH con la que nos conectaremos a nuestra instancia.
>Nota 1: Si no cuenta con una clave SSH, dar click en ```Crear una clave SSH```
>Asignar un nombre a nuestra nueva clave SSH y copiar la clave pública.
>Finalmente dar click en ```Añadir clave SSH```
>
<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/config%20power%20instancia.png"></p>

**Imagen de arranque:**
* ```Sistema Operativo```: Escoja el sistema operativo para su *instancia de Power VS*. Para el tutorial usaremos: IBM i
* ```Imagen```: Seleccionamos la imagen con la cual trabajaremos. (Ejemplo: IBMi74-01-001)
* ```Nivel```: Seleccionaremos el nivel de almacenamiento. (Ejemplo: Nivel 3)
* ```Agrupación de almacenamiento```: Seleccionaremos agrupación automáticamente.

**Perfil:**
* ```Tipo de Máquina```: Escoja el tipo de máquina según la capacidad de memoria que proporciona. (Ejemplo: e980).
* ```Tipo de núcleo```: Seleccionamos sin limitación compartida.
* ```Núcleos```: Seleccionaremos la capacidad de núcleos.
* ```Memoria```: Seleccionaremos la capacidad de memoria.

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/config%20perfil.png"></p>

**Interfaces de red:**
* ```Redes públicas```: Activarla en caso que la requiera.

*Redes privadas:*
Como contiuación del paso 2 donde se crearon las subredes privadas procederemos a conectarlas a nuestra instancia por desplegar, daremos click en ```Conectar existente``` y una vez le aparezca la ventana para la conexión, complete lo siguiente:
* ```Redes existentes```: Seleccionar la red previamente creada.
* ```Dirección IP```: Asignar automáticamente una dirección IP de un rango de IP.

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/config%20redes%20privadas.png"></p>

Una vez completado los campos finalizar dando click en ```Conectar```.

<br />

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/tree/main/Imagenes/arquitectura ref powervs.png"></p>

>Nota: Replicar el procedimiento para el despligue de la instancia en el otro datacenter

<br />


## :cloud: Desplegar y configurar cloud connections
Una vez desplegadas las instancias de PowerVS, procederemos a creación conexiones cloud, para ello se ubicará en la sección de ```Lista de recursos``` seleccione la opción ```Servicios y Software``` y ubique el recurso del PowerVS location implementado, posteriormente ingrese a la sección ```conexiones cloud``` y darle click en ```Crear una conexión```, una vez que aparezca la ventana de configuración complete lo siguiente:

**Detalles del recurso:**
* ```Nombre```: Asignar un nombre exclusivo para el cloud connection.
* ```Velocidad```: Seleccionar velocidad Mbps para nuestra conexión.
* ```Direccionamiento global```: Habilitar

* **Virtual Connections:**
* ```Infrastructura Clásica```: Habilitarla ya que trabajaremos con túneles GRE.

* **Subredes:**
* ```Conectar existente```: Seleccionar la subred creada previamente en el paso 2 y darle click en ```Conectar```.

Una vez completado los campos dar click en ```Finalizar``` y luego en ```Conectar```.
Esperar unos minutos para establecer la conexión.

<p align="center"><img width="400" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/conectared.png"></p>

<br />

## :wrench: Desplegar y configurar un Vyatta en cada datacenter

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/vyatta_config.png"></p>

<br />

## :gear: Configurar los túneles GRE en cada PowerVS location

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/Conexión virtual - GRE creation.png"></p>

<br />

## :hammer_and_wrench: Configurar los túneles IPSec entre los dos Vyattas

<p align="center"><img width="800" src="https://github.ibm.com/YrinaSuarez/IBM-PowerVS-Disaster-Recovery/blob/main/Imagenes/vyatta_config 100%.png"></p>

<br />

## :gear: Configurar un servidor proxy en cada datacenter (Opcional)

<br />

## :mag: Referencias 
* <a href="https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-creating-power-virtual-server#creating-power-virtual-server"> Creación de un Power Systems Virtual Server </a>.
* <a href="https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-configuring-subnet"> Configuración de una subnet privada </a>.
* <a href="https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-cloud-connections"> Configuración de cloud connections </a>.
* <a href="https://cloud.ibm.com/docs/virtual-router-appliance?topic=virtual-router-appliance-getting-started"> Virtual Router Appliance (VRA) </a>.
* <a href="https://cloud.ibm.com/docs/virtual-router-appliance?topic=solution-tutorials-configuring-IPSEC-VPN"> Configuración de túneles IPSec VPN </a>.
* <a href="https://cloud.ibm.com/media/docs/downloads/power-iaas-tutorials/PowerVS_IBMi_DR_Tutorial_v1.pdf"> PowerVS DR Tutorial </a>.
<br />

## Comentarios del autor a tener en cuenta
* The following Docs contains a connection configuration using the same concept as this article. https://cloud.ibm.com/docs/vpc topic=vpc-vpn-policy-based-ingress-routing-integration-example
* I used Transit Gateway, but from November 2022, Transit Gateway will charge a metered fee for data transfer even for local type.
 I think it is also a good idea to connect Cloud Connection (Direct Link 2.0) directly to VPC without using Transit Gateway.

## :black_nib: Autores 
Equipo ☁️ IBM Public Cloud Customer Success.
