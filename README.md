# IBM Power Virtual Servers ☁️ - Conexión VPN On-Premise 🗄️

## 📃 Introducción
Power Systems Virtual Server ***PowerVS*** tiene servicio VPNaaS, pero existen algunas limitaciones, como se describe en la documentación ["Limitaciones VPNaaS de PowerVS"](https://cloud.ibm.com/docs/power-iaas?topic=power-iaas-VPN-connections). Es por lo que, en este artículo, vas a aprender a cómo acceder a PowerVS usando una VPN site-to-site que se puede usar en VPC, en lugar de la VPNaaS de PowerVS.

   *** Los puntos clave a tener en cuenta antes de empezar con la guía son: ***
   * PowerVS no conoce la información de enrutamiento al rango de direcciones IP de la infraestructura local, por lo que no es posible enviar paquetes desde PowerVS a la red local.
   * La clave de esta configuración es definir el rango de direcciones IP de la red local como el prefijo de la VPC. Esta definición permite que la información de enrutamiento para el rango de IP local se anuncie a PowerVS a través de conexiones en la nube (Direct Link 2.0) entre la VPC y PowerVS, lo que permite que PowerVS envíe paquetes para el rango de IP local a la VPC.
   * Defina una tabla de enrutamiento de entrada en la VPC para que los paquetes de PowerVS a la VPC (destinados a la IP local) se enruten a la puerta de enlace VPN.
   * La puerta de enlace VPN pasa los paquetes a las instalaciones a través del túnel VPN, lo que permite la comunicación de un extremo a otro.

A conitnuación se muestra la arquitectura de esta conexión, en esta también se muestra las distintas subnets involucradas tanto del lado de IBM como on-premise:
<p align="center"><img width="800" src="https://github.com/samirsoft-ux/Playbook_Power/blob/main/Imagenes/IS-arqui-power.png"></p>

<br />

## 📑 Índice  
1. [Pre-Requisitos](#pencil-Pre-Requisitos)
2. [Configuración de la VPN site-to-site](#Configuración-de-la-VPN-site-to-site)
3. [Configuración del Cloud Connection en PowerVS](#Configuración-del-Cloud-Connection-en-PowerVS)
4. [Configuración del Transit Gateway](#Configuración-del-Transit-Gateway)
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
* Tener una VPC ya creada la cual no tenga un prefijo de dirección predeterminado ya que en el transcurso de esta guía se le va agregar de forma manual.
* Tener un Workspace dentro del servicio de PowerVS con una instancia que solo tenga una subred privada.
<br />

## 1° Configuración de la VPN site-to-site
```Esta configuración permite la conexión entre la red local(on-premise) con la VPC.```

1. Ingresar al ***Navigation Menu*** dentro dirigirse a la sección ***VPC Infraestructure*** y seleccionar el apartado ***VPNs***.
   
2. Dar click en el botón "Create +".

   **Parámetros de creación**
   * El tipo de VPN debe ser ***Site-to-site gateways***.
   * La locación debe ser en ***Dallas*** ya que es donde menos latencia existe si se encuentra en Perú.
   * Escribir un nombre para el gateway que haga referencia al servicio y donde se encuentra.
   * Elegir el grupo de recursos de su preferencia.
   * Si desea ingrese una etiqueta esta te ayuda a identificar detalles del recurso.
   * Si desea ingrese una etiquetaa de administración de acceso esta te ayuda a aplicar políticas de acceso flexibles en recursos específicos.
   * Elige la VPC que ya está previamente creada.
   * La sección ***Subnet*** debe de estar vacía.
   * El modo debe ser ***Policy-based***.
   * Escribir un nombre para la conexión que haga referencia de donde a donde se está realizando la conexión.
   * Ingrese la ***Peer gateway address***, esta es la dirección IP pública del gateway de la red local.
   * Ingrese un ***Preshared key***, este es la clave configurada en la Peer gateway.
   * Los parámetros de la sección ***Dead peer detection*** dejarlos por defecto.
   * Crear un IKE policy.
   * Crear un IPsec policy.

3. Finalmente luego de haber creado la conexión asegurarse que el estado de la VPN sea ***Activa***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_1.gif>
</p>

   **Notas**
   * La conexión debe ser ***Policy Based***.
   * Esta es la <a href="https://cloud.ibm.com/docs/vpc?topic=vpc-using-vpn"> ***documentación oficial*** </a> en la cual puedes ver un overview de lo que es una Site-to-Site VPN.
   * En el enrutador VPN de la red local, también especifique la subred PowerVS, no la subred de la VPC, para los CIDR del mismo nivel.
<br />

## 2° Configuración del Cloud Connection en PowerVS
```Esta configuración es el primer paso para poder establecer la conexión del Power con la VPC ya que se establece que el power tiene que hacer uso de una conexión Direct Link 2.0.```

1. Ingresa a la sección de ***Lista de recursos*** y dentro ubicar el apartado ***Compute***.

2. Seleccionar el ***Workspace*** en donde se va a trabajar y dirigirse a la sección ***Cloud connections***.

3. Dentro darle click al botón "Create connection +".

   **Parámetros de creación**
   * Escribir un nombre para la conexión que haga referencia de donde a donde se está realizando la conexión.
   * Seleccionar una velocidad de 50 Mbps ya que con esta es suficiente para solo probar la conexión una vez terminada toda la guía.
   * Asegurarse que las opciones ***Enable global routing*** y ***Enable IBM Cloud Transit Gateway*** se encuentren habilitadas.
   * Seleccionar el botón "Done editing".
   * Habilitar la opción ***I understand virtual connections must be configured by creating a transit gateway in IBM interconnectivity***.
   * Seleccionar el botón "Continue".
   * En la seccion ***Subnet*** conectar la subnet privada de la instancia creada previamente.

4. Finalmente luego de haber creado el ***Cloud connection*** asegurarse que el estado sea ***Established***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_2.gif>
</p>

   **Notas**
   * La conexión debe ser de tipo ***Transit Gateway***.
<br />

## 3° Configuración del Transit Gateway
```Esta configuración es el segundo paso para poder establecer la conexión del Power con la VPC ya que se hace uso de la conexión Direct Link 2.0 ya establecida para que el Transit Gateway establezca la conexión Power-VPC.```

1. Ingresar al ***Navigation Menu*** y dentro dirigirse a la sección ***Interconnectivity***.

2. Dentro de esta sección dirigirse al apartado ***Transit Gateway***.

3. Seleccionar el botón "Create transit gateway".

   **Parámetros de creación**
   * Escribir un nombre para el Transit que haga referencia de donde a donde se está realizando la conexión.
   * Elegir el grupo de recursos de su preferencia.
   * Dentro de la sección ubicación la opción de routung debe de ser ***Local routing*** y la ubicación debe ser en Dallas la misma en donde se encuentra el Workspace de Powervs.
   * Establecer una conexión de tipo ***Direct Link*** y seleccionar la que hemos creado en la configuración anterior.
   * Dejar el nombre por defecto que aparece y seleccionar el botón ***Create***.

4. Finalmente luego de haber creado el ***Transit Gateway*** asegurarse que el estado de la conexión ***Direct Link*** creada sea ***Attached***.

<p align="center">
   <img src=https://github.com/samirsoft-ux/Playbook_Power/blob/main/GIFs/Part_3.gif>
</p>
<br />

## Configuración del prefijo de la VPC
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
