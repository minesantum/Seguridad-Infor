# Chuleta CCNA

## Orden de configuración recomendado

### **1. Switching**
1. **EtherChannel** → Agrupa puertos físicos antes de hacer trunking
2. **Trunking** → Configura los enlaces entre switches
3. **VLANs** → Crea las VLANs y asígnalas a los puertos
4. **PortFast** → En puertos de usuarios finales (PCs)
5. **Port-Security** → Restringe qué MACs pueden conectarse

### **2. Seguridad Avanzada (Protección de Capa 2)**

1. **DAI** - Proteja la red contra ataques de ARP Spoofing
2. **DHCP Snooping** - Evite que alguien conecte un router propio y empiece a dar IPs falsas

### **3. Enrutamiento**

1. **HSRP** - Redundancia (FHRP)
2. **Inter-VLAN Routing** 
3. **Enrutamiento Estático - Rutas Estáticas y por Defecto** - Defecto (`0.0.0.0`) y Estáticas (`ip route 192.168...`)
4. **Router-on-a-Stick** → Permite comunicación entre VLANs con un cable
5. **OSPF** → Enrutamiento dinámico entre routers

### **4. Servicios**
1. **DHCP** → Reparte IPs automáticamente - Hacer ip helper-address luego
2. **NTP** → Sincroniza el reloj de los equipos
3. **Syslog** → Monitorización y Gestión de Logs

### **5. Seguridad**
1. **ACL** → Filtra tráfico
2. **NAT** → Traduce IPs privadas a pública para salir a internet

### **6. Gestión remota**
1. **SSH** → Acceso remoto seguro al CLI

------

# EtherChannel - Switch

> Agrupa varios cables físicos entre switches en uno lógico para más ancho de banda y redundancia.

---

**"Configure los puertos fa0/1 al fa0/4 al mismo tiempo."**
```
interface range fa0/1-4
```
- `range` → selecciona múltiples puertos a la vez

---

**"Agrupe estas interfaces en el canal 1 usando el protocolo PAgP de forma activa."**
```
channel-group 1 mode desirable
```
- `desirable` → PAgP, inicia la negociación activamente

---

**"Agrupe estas interfaces en el canal 1 usando el protocolo PAgP, pero que espere al otro extremo."**
```
channel-group 1 mode auto
```
- `auto` → PAgP, espera que el otro extremo inicie

---

**"Agrupe estas interfaces en el canal 2 usando el protocolo LACP (estándar 802.3ad) de forma activa."**

```
channel-group 1 mode active
```
- `active` → LACP, inicia la negociación activamente

---

**"Agrupe estas interfaces en el canal 2 usando LACP, pero que espere al otro extremo."**
```
channel-group 1 mode passive
```
- `passive` → LACP, espera que el otro extremo inicie

---

**"Fuerce el EtherChannel sin usar ningún protocolo de negociación."**
```
channel-group 1 mode on
```
- `on` → sin PAgP ni LACP, ambos extremos deben estar en `on`

---

**"Configure el port-channel 1 como enlace troncal."**
```
interface port-channel 1
switchport mode trunk
```
- Entras a la interfaz lógica del canal para configurarla

---

**"Configure el balanceo de carga del EtherChannel basado en IP origen-destino."**
```
port-channel load-balance src-dst-ip
```
- Opciones: `src-ip`, `dst-ip`, `src-dst-ip`, `src-mac`, `dst-mac`, `src-dst-mac`

---

**"Verifique el estado del EtherChannel."**
```
show etherchannel summary
```
- Busca el estado `SU` → `S` = capa 2, `U` = en uso (correcto)

------

# Trunking - Switch

> Un enlace troncal transporta el tráfico de múltiples VLANs

---

**"Configure el puerto como enlace troncal."**
```
switchport mode trunk
```

---

**"El switch necesita encapsulación dot1q antes de poder configurar el troncal." (switches capa 3)**
```
switchport trunk encapsulation dot1q
```
- Solo necesario en switches multicapa o routers con módulo switch
- Si al escribir `switchport mode trunk` el switch te lanza un error diciendo *"Command rejected"*, significa que tienes que escribir primero el comando de `encapsulation`. Si no te da error, no lo necesitas

---

**"Permita únicamente las VLANs 10 y 20 en el enlace troncal."**

```
switchport trunk allowed vlan 10,20
```
- ⚠️ Esto **reemplaza** la lista completa. Si quieres añadir sin borrar, usa `add`

---

**"Añada la VLAN 30 a las VLANs permitidas en el troncal."**
```
switchport trunk allowed vlan add 30
```
- `add` → añade sin borrar las anteriores

---

**"Elimine la VLAN 20 de las VLANs permitidas."**
```
switchport trunk allowed vlan remove 20
```

---

**"Configure la VLAN 99 como VLAN nativa del enlace troncal."**
```
switchport trunk native vlan 99
```
- La VLAN nativa viaja sin etiqueta. Debe ser la misma en ambos extremos.

---

**"Desactive la negociación dinámica DTP en este puerto."**
```
switchport nonegotiate
```

---

**"Verifique qué puertos son troncales y qué VLANs permiten."**

```
show interfaces trunk
```

Debes configurar el puerto **Fa0/1** para que permita pasar las VLAN

```
interface fa0/1
switchport trunk allowed vlan 20
switchport trunk allowed vlan all
```

Abre el paso en los Port-Channels

```
interface port-channel 1
switchport trunk allowed vlan add 10,20,99
```

------

# VLANs - Switch o declarar int vlan en Router

> Las VLANs segmentan la red lógicamente, separando el tráfico aunque estén en el mismo switch
>
> - La IP que pongas en default-gateway la sacas de una de las VLAN, la que elijas para ello acabada en `1`, será la VLAN de Administración (`...99`)
> - En los switches int vlan con ip address .2 o más, la 99 solamente y poner `default-gateway` con int vlan y la ip correspondiente
> - Solo pones IP si el dispositivo necesita "ser visto" en esa red (para gestionar o para enrutar). Si solo debe pasar paquetes de un lado a otro, no lleva IP

---

**"Cree la VLAN 10 con el nombre Ventas"**

```
vlan 10
name Ventas
```

---

**"Asigne el puerto fa0/5 a la VLAN 10"**

```
interface fa0/5
switchport mode access
switchport access vlan 10
```
- `mode access` → puerto para dispositivo final (PC, impresora...)
- Al ejecutar el comando `switchport access vlan 10`, estás **sacando** por completo ese puerto de la VLAN 1 y metiéndolo exclusivamente en la VLAN 10, por eso es modo access y no trunk

---

**"Configure el puerto para que acepte tanto datos como voz, la voz en la VLAN 50"**

```
switchport access vlan 10
switchport voice vlan 50
```
- Se usa cuando hay un teléfono IP conectado al switch

---

**"Muestre todas las VLANs y los puertos asignados a cada una"**

```
show vlan brief
```

---

**"Muestre la configuración de switchport de una interfaz concreta"**

```
show interfaces fa0/1 switchport
```

**Poner IP a una VLAN en un Switch** - Algunas veces hay que poner la IP de la vlan en la pata del router que va hacia las VLAN y luego anunciar la red por OSPF

```
interface vlan 99
ip address 192.168.99.2 255.255.255.0
no shutdown
```

**Si la VLAN 10 llega por la interfaz g0/1 hazlo así, poniendo la IP siguiente a la que pusimos en los Swicthes**

En los Swicthes recuerda

```
interface vlan 10
ip address 192.168.10.2 255.255.255.0
no shutdown
```

En el Router

```
interface GigabitEthernet0/1
ip address 192.168.10.3 255.255.255.0
no shutdown
```

### El esquema queda así:

- **192.168.10.2:** Switch (Interfaz VLAN 10)
- **192.168.10.3:** Router VLAN 10 (Interfaz Gig0/1)

------

# PortFast - Switch el Spanning Tree

> PortFast hace que un puerto pase directamente a forwarding, sin esperar los 30 segundos de STP. Solo para puertos de dispositivos finales (PCs).

---

**"Active PortFast en el puerto fa0/5 (está conectado a un PC)."**
```
interface fa0/5
spanning-tree portfast
```

---

**"Active PortFast en todos los puertos de acceso del switch por defecto."**
```
spanning-tree portfast default
```
- Se configura en modo global, no en interfaz

---

**"Active BPDU Guard en el puerto fa0/5 para que se apague si recibe un BPDU."**
```
interface fa0/5
spanning-tree bpduguard enable
```
- Si alguien conecta un switch donde no debe, el puerto se apaga automáticamente (err-disable)

---

**"Active BPDU Guard globalmente en todos los puertos con PortFast."**
```
spanning-tree portfast bpduguard default
```

---

**"Verifique el estado de PortFast y BPDU Guard."**

```
show spanning-tree summary
show spanning-tree interface fa0/5 portfast
```

Port Channel 2 no funciona porque el protocolo Spanning Tree colocó algunos puertos en el modo de bloqueo. Desafortunadamente, esos puertos eran puertos Gigabit. En esta topología, puede restaurar estos puertos configurando **S1** para que sea la **raíz** principal de la VLAN 1. También puede establecer la prioridad en **24576**

```
spanning-tree vlan 1 root primary
spanning-tree vlan 1 priority 24576
spanning-tree vlan 1 priority
```

- Si los otros puertos, los que no van en el Etherchannel, siguen en naranja, se ponen en verde cuando vas creando las VLAN

------

# Port-Security - Switch

> Restringe qué direcciones MAC pueden conectarse a un puerto. Útil para evitar dispositivos no autorizados

---

**"Active la seguridad de puerto en fa0/5."**
```
interface fa0/5
switchport mode access
switchport port-security
```
- ⚠️ El puerto debe estar en modo `access` primero

---

**"Permita un máximo de 2 dispositivos en este puerto."**
```
switchport port-security maximum 2
```
- Por defecto el máximo es 1
- Se aprenden 2 MAX como máximo

---

**"Fije manualmente la MAC permitida en este puerto"**

```
switchport port-security mac-address 0050.1234.ABCD
```

---

**"Haga que el switch aprenda automáticamente la primera MAC que se conecte y la guarde"**

```
switchport port-security mac-address sticky
```
- "Sticky" guarda la MAC aprendida en la configuración (como si fuera estática)

---

**"Si hay una violación, que el puerto caiga completamente."** (más común en examen)
```
switchport port-security violation shutdown
```

---

**"Si hay una violación, que descarte el tráfico no permitido pero sin apagar el puerto."**
```
switchport port-security violation restrict
```

---

**"Si hay una violación, que descarte silenciosamente sin log ni notificación."**
```
switchport port-security violation protect
```

---

**"Verifique el estado de la seguridad de puerto."**
```
show port-security
show port-security interface fa0/5
show port-security address
```

------

Como ya tienes un flujo de trabajo muy bien organizado por bloques, aquí tienes la sección de **Inter-VLAN Routing (Capa 3)** diseñada específicamente para que la copies y pegues en tu documento

He incluido los comandos esenciales, la explicación directa y el "porqué" técnico que necesitas para ASIR

Para que el **HSRP** funcione correctamente en tu entorno de ASIR, debe configurarse exactamente en el punto donde reside la **Puerta de Enlace (Gateway)** de los usuarios

Aquí tienes la guía técnica para añadir a tu chuleta, dividida por los dos escenarios posibles que manejas:

------

# Seguridad Avanzada (Protección de Capa 2) - Switches normales y Capa 3

> - La regla de oro en seguridad es aplicarla lo más cerca posible del usuario
> - Se configura en los **switches** donde se conectan físicamente los PCs. Su función es distinguir entre puertos de confianza (donde está el servidor DHCP) y puertos no confiables (usuarios). Los puertos que van a los PCs se quedan siempre como `untrusted`

Protege la red contra ataques que intentan engañar a los equipos para robar datos o saturar el servidor DHCP

### **Dynamic ARP Inspection (DAI)**

"Proteja la red contra ataques de ARP Spoofing (suplantación de identidad)"

**1. Activar DHCP Snooping (Requisito previo):**

```
ip dhcp snooping
ip dhcp snooping vlan 10,20
```

**2. Confiar en la interfaz que va al Router/Core que es el servidor DHCP:**

> - ⚠ Si el Switch de Capa 3 tiene los PCs conectados directamente a él, el paso 2 no existe

```
interface fa0/1					#Interfaz que va al Servidor DHCP desde Switch
ip dhcp snooping trust			 #Solo se aplica en interfaces físicas como fa0/1
ip arp inspection trust
```

**3. Activar la inspección en las VLANs:**

```
ip arp inspection vlan 10,20
```

- **Regla de oro:** El puerto que va al router DHCP Servidor es `trust` (confianza); el de los PCs es `untrust` (por defecto)

### **DHCP Snooping (Protección DHCP)**

**"Evite que alguien conecte un router propio y empiece a dar IPs falsas"**

```
interface range fa0/10-20
interface range fa0/5, fa0/8, fa0/12
ip dhcp snooping limit rate 10
```

- `limit rate 10` Si un puerto envía más de 10 peticiones DHCP por segundo, se apaga (evita ataques de denegación de servicio)
- `interface range` Son los puertos **Untrusted** que van a los PC's
- No se puede hacer por `interface vlan`
- No se puede hacer por subinterfaz `int fa0/1.10`

⚠ Los ordenadores no cogen IP del Server DHCP - Tienes que hacer esto

Por defecto, cuando activas DHCP Snooping, el switch se vuelve "demasiado cotilla" y le mete un código (la Option 82) a las peticiones de los PCs. Al hacer esto en Packet Tracer, **el propio switch bloquea el tráfico** y los PCs se quedan sin IP

```
no ip dhcp snooping information option
```

"**Ver tabla de DHCP Snooping**"

```
show ip dhcp snooping
```

Para saber si el **DHCP Snooping** está haciendo su trabajo de "policía" en tu red de Packet Tracer, tienes varias formas de comprobarlo, desde comandos de estado hasta pruebas de ataque real

Aquí tienes los pasos de ingeniero para verificarlo:

### 1. La "Prueba de Fuego" (Show ip dhcp snooping binding)

Este es el comando más importante. Si el DHCP Snooping está funcionando y un PC ha recibido una IP legalmente, el switch debe haber anotado esa "matrícula" en su base de datos

```
show ip dhcp snooping binding
```

- **¿Qué debes ver?** Una tabla con la MAC del PC, la IP que le dio el servidor, en qué puerto está conectado y a qué VLAN pertenece
- **Si la tabla está vacía:** Algo falla. O el PC no ha pedido IP todavía, o no has configurado el `trust` en el puerto que va al servidor

### 2. Verificar el estado de los puertos (Trusted vs Untrusted)

Recuerda que por defecto todos los puertos son `untrusted` (no fiables). El puerto que va hacia tu servidor DHCP **DEBE** ser `trusted`

```
show ip dhcp snooping
```

- Busca la sección que dice **"Interface Trusted"**. Ahí debería aparecer el puerto que conecta con tu servidor o con el router que hace de DHCP
- Si el puerto del servidor aparece como `No`, el switch bloqueará todas las respuestas del servidor y los PCs nunca tendrán IP

### 3. Prueba de ataque (El router intruso)

Si quieres ver cómo actúa en tiempo real:

1. Conecta un router extra o un servidor DHCP falso a un puerto del switch que sea `untrusted` (cualquiera de los que van a los PCs)
2. Configúralo para que intente dar IPs
3. **Resultado esperado:** El switch detectará que una "oferta de IP" viene de un puerto no fiable y la tirará a la basura inmediatamente. El PC nunca recibirá la IP falsa

### 4. Ver los descartes (Estadísticas)

Si ha habido un intento de ataque o un error de configuración, el switch lleva la cuenta:

```
show ip dhcp snooping statistics
```

- Si ves números en **"Packets dropped"**, significa que el DHCP Snooping ha interceptado y bloqueado paquetes ilegales

> ⚠ Si activas DHCP Snooping y tus PCs dejan de recibir IP, el 99% de las veces es porque olvidaste poner el comando `ip dhcp snooping trust` en el puerto que conecta con el servidor DHCP o el Router

# HSRP (Hot Standby Router Protocol) - Router o Switch de Capa 3

El **HSRP** puede tener hasta **255 dispositivos** en un mismo grupo, pero en el mundo real (y en ASIR) lo normal es ver **dos**

> - Crea una "IP Virtual" compartida entre dos equipos. Si uno cae, el otro asume la IP automáticamente y los PCs no pierden la conexión
> - **HSRP no copia ni pega nada**. HSRP es muy "tonto": su único trabajo es mantener viva una **IP Virtual** y una **MAC Virtual**. Si el Router0 explota, el Router1 asume esa IP, pero **no hereda** tus pools de DHCP, ni tus ACLs, ni tu configuración de OSPF
> - No puedes hacer HSRP "solo". Necesitas una pareja de baile (otro router u otro switch de capa 3) para que cuando uno caiga, el otro asuma la IP virtual
> - Puede hacerse en subinterfaces de Router and Stick
> - Nunca usar una IP ya declarada a mano en otro dispositivo en el comando stanby 1
> - ⚠ Si tienes **HSRP mal:** El PC tiene su IP correcta que recibe por DHCP, pero el `ping` a su Gateway falla, pero puede dar ping al servidor final de fuera de internet gracias a NAT
> - **HSRP bien** Una vez cambiadas las IP's a 2 o 3 o 4 en los cores, una vez hecho el HSRP, ya va el ping

## Escenario 1 - Routers conectados a un Switch Router and Stick

**Router 0 (El Server DHCP)**

Entras en las dos subinterfaces que creaste para el Router-on-a-Stick

```
# HSRP para la VLAN 10

interface gig0/1.10

int vlan 10 --- ELEGIR PRIMERO
encapsulation dot1Q 10
ip address 192.168.10.2 255.255.255.0 # La .1 ya la tienes declarada como gateway

standby 1 ip 192.168.10.1           # Gateway Virtual VLAN 10
standby 1 priority 110              # Ponerlo siempre en el primer router
standby 1 preempt                   # Solo se pone en el Router de mayor prioridad

# HSRP para la VLAN 20
interface gig0/1.20
ip address 192.168.20.2 255.255.255.0 # La .1 ya la tienes declarada como gateway
encapsulation dot1Q 20

standby 2 ip 192.168.20.1           # Gateway Virtual VLAN 20
standby 2 priority 110              # Ponerlo siempre en el primer router
standby 2 preempt                   # Solo se pone en el Router de mayor prioridad
```

- Como ya declaraste la `.1` en la ip de la interface vlan 10 en el Switch - Ahora tienes que declarar la `.2` en el comando de ip address, que es la que queda libre y lo mismo con todas las VLAN

- La regla de la **prioridad**:

  - **En el Router Principal (DHCP Server):** Pones `standby 1 priority 110` para que saque una nota alta y sea el "delegado" (Active).
  - **En el Router de Respaldo:** **No pones nada**. Por defecto, todos los routers tienen una prioridad de **100**. Como 110 es mayor que 100, el Router DHCP siempre ganará.

- ¿Qué hace exactamente **preempt**?

  Permite que un router (Router0 en este caso) con mayor prioridad **le quite el puesto** al que está funcionando en ese momento cuando vuelve a estar online después de una desgracia.

  - **Sin `preempt`:** Si el Router 0 (el jefe) se reinicia, el Router 1 toma el control. Cuando el Router 0 vuelve a estar encendido, se queda mirando (en espera) aunque tenga más prioridad, porque el Router 1 ya "ha pillado el sitio" y el Router 0 es educado y no molesta.  
  - **Con `preempt`:** En cuanto el Router 0 vuelve a estar online, detecta que tiene más prioridad (110 vs 100) y le dice al Router 1: *"Quita de ahí, que el jefe ha vuelto"*. Automáticamente recupera la IP Virtual y vuelve a ser el **Active**. 

**Router 1 (El de Respaldo)**

```
# HSRP para la VLAN 10
interface gig0/0.10
ip address 192.168.10.3 255.255.255.0

standby 1 ip 192.168.10.1           # Mismo grupo y misma IP Virtual
standby 1 priority 109			   # Puedes ponerlo o no

# HSRP para la VLAN 20
interface gig0/0.20
ip address 192.168.20.3 255.255.255.0

standby 2 ip 192.168.20.1           # Mismo grupo y misma IP Virtual
standby 2 priority 109			   # Puedes ponerlo o no
```

- Declaras la 192.168.10.3, ya que la `.3` la declaraste en el Router principal
- ⚠ ip address - No puede haber dos iguales en ningún Router o Switch

## Al rato de acabar las configuraciones, la consola enviará mensajes como

```
%HSRP-6-STATECHANGE: GigabitEthernet0/0.10 Grp 1 state Speak -> Standby
%HSRP-6-STATECHANGE: GigabitEthernet0/0.20 Grp 2 state Speak -> Standby
```

## Para comprobar si funciona o no

Primero vete al **Router principal** y pon este comando `show standby brief` - El State en **Active**

```
do show standby brief
```

```
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         1    110 P Active   local           192.168.10.3    192.168.10.1   
Gig         2    110 P Active   local           192.168.20.3    192.168.20.1  
```

Ahora vete al **Router** secundario**, pon el mismo comando `show standby brief` - El State en **Stanby**

```
do show standby brief
```

```
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gig         1    109   Standby  192.168.10.2    local           192.168.10.1   
Gig         2    109   Standby  192.168.20.2    local           192.168.20.1
```

> ⚠ **HSRP mal:** El PC tiene su IP correcta que recibe por DHCP, pero el `ping` a su Gateway falla

## Escenario 2 - Swicth de Capa 3 conectado por puerto trunk a Switch normal

### Preparar el puerto Trunk en el Switch de Capa 3

Primero, asegúrate de que el puerto físico que va hacia el Switch0 esté en modo trunk y permita las VLANs necesarias

```
interface g0/1
switchport trunk encapsulation dot1q  # Solo si el switch lo pide
switchport mode trunk
switchport trunk allowed vlan 10,20
```

En el Switch de Capa 3 **Principal** -  En el servidor **DHCP**

En un switch de Capa 3, HSRP se configura en las interfaces virtuales de cada VLAN

```
ip routing                             # Activa el enrutamiento

interface vlan 10
 ip address 192.168.10.2 255.255.255.0 # IP real del Switch
 standby 1 ip 192.168.10.1             # Gateway Virtual (la que usan los PCs)
 standby 1 priority 110                # Prioridad alta para ser el principal
 standby 1 preempt                     # Recupera el control tras un fallo

interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 standby 2 ip 192.168.20.1
 standby 2 priority 110
 standby 2 preempt
```

En el Switch de Capa 3 de **Respaldo**

```
interface vlan 10
 ip address 192.168.10.3 255.255.255.0 # IP real distinta
 standby 1 ip 192.168.10.1             # Misma IP - Gateway (la que usan los PCs)
 standby 1 priority 109			   # Puedes ponerlo o no
 
interface vlan 20
 ip address 192.168.20.3 255.255.255.0
 standby 2 ip 192.168.20.1
 standby 2 priority 109			   # Puedes ponerlo o no
```

------

# Inter-VLAN Routing - Switch Capa 3 (Multicapa) o Router

> Permite que el switch CORE actúe como un router, comunicando las VLANs internamente sin necesidad de subir al router externo. Es más rápido y eficiente que el Router-on-a-Stick
>
> ⚠️ Si haces Inter-VLAN Routing, no haces Router and Stick en el mismo dispositivo

------

**"Active la capacidad de enrutamiento del switch (encienda el cerebro de router)."**

```
ip routing
```

- ⚠️ Sin este comando, el switch solo funcionará en Capa 2 y las VLANs no se comunicarán entre sí.

------

**"Cree la interfaz virtual (SVI) para la VLAN 10 y asígnele su puerta de enlace."**

```
interface vlan 10
ip address 192.168.10.2 255.255.255.0
no shutdown
```

- `interface vlan X` → crea una "pata virtual" del router dentro de esa VLAN.
- La IP asignada aquí **DEBE ser la Default Gateway** que configures en los **PCs** de esa VLAN

------

**"Configure una ruta por defecto hacia el Router ISP para tener salida a Internet - ⚠ Esto se hace en NAT"**

```
ip route 0.0.0.0 0.0.0.0 192.168.255.1
```

- `0.0.0.0 0.0.0.0` → significa "cualquier red con cualquier máscara" (Internet).
- `192.168.255.1` → es la IP del siguiente salto, la entrada del dispositivo siguiente del mismo cable

------

**"Convierta un puerto físico del switch en un puerto de enrutamiento (Capa 3) para conectar al Router."**

```
interface g0/1
no switchport
ip address 192.168.255.2 255.255.255.252
```

- `no switchport` → desactiva las funciones de switch (VLANs, STP) y permite poner una IP directamente en el puerto físico.
- Se suele usar una máscara `/30` (`255.255.255.252`) para estos enlaces directos.

------

**"Verifique la tabla de enrutamiento del switch."**

```
show ip route
```

- Debes ver las redes de tus VLANs marcadas con una **"C"** (Conectadas) y la ruta por defecto con una **"S\*"** (Estática).

------

### **¿Cuándo usar esto en lugar de Router-on-a-Stick?**

- **SVI (Capa 3):** Úsalo siempre que tengas un switch Multicapa (CORE). Es la forma profesional
- **Router-on-a-Stick:** Úsalo solo si tu switch es de Capa 2 (barato) y te ves obligado a que el router haga todo el trabajo

------

# Enrutamiento Estático - Routers a Switch de Capa 3 función de Router

> - **Si haces esto, no haces OSPF con default-originate**
> - Esto solo envía y trae el paquete, la NAT solo cambia la dirección de privada a pública
> - **Hacer ip route en todos los dispositivos del salto**
> - **En los Switches de Capa 3, el enrutamiento viene desactivado por defecto, **ejecutar **ip routing**
> - El **puerto** que va del Switch de Capa 3 al Router tiene que ser **no switchport**
> - No hace falta que estén los routers conectados directamente, puede haber un switch de por medio

## Escenario 1

⚠ Está internet -> router -> switch de capa 3 - ..... -> las VLAN

El router no conoce las redes `192.168.10.0` ni `192.168.20.0`. Debes decirle que para llegar a ellas, tiene que enviar el tráfico al Switch de Capa 3

1. internet -> **En el Router (Hacia las VLAN - VUELTA)** -> Switch de capa 3 - ..... -> las VLAN

```
ip route 192.168.10.0 255.255.255.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 10.0.0.2
```

- ip route es el comando
- 192.168.10.0 255.255.255.0 dirección acabada en .0 de la VLAN que quieres declarar
- 10.0.0.2 IP de la Interfaz del Switch de Capa 3 que va al Router

2. las VLAN -> .... -> **En el Switch de Capa 3 (Hacia Internet - IDA)** -> router -> internet 

```
ip routing
ip route 0.0.0.0 0.0.0.0 10.0.0.3
```

- ip route es el comando
- 0.0.0.0 0.0.0.0 **"cualquier destino que no conozcas localmente"** (es decir, Internet)
- 10.0.0.3 IP de la Interfaz del Router que va al Switch de Capa 3

## Escenario 2

Como en esta red **no hay Internet ni Switch de Capa 3**, la lógica de tu guía cambia ligeramente: ahora los protagonistas son los **dos Routers**. Para que un PC de la izquierda (ej. `192.168.1.3`) hable con uno de la derecha (ej. `192.168.20.2`), los routers necesitan saber llegar a las redes que no tienen conectadas físicamente

Aquí tienes cómo aplicar tu lógica paso a paso:

### 1. En el Router1 (Izquierda)

Este router conoce las redes rosa (`1.0`) y verde (`2.0`), pero **no sabe nada** de la azul (`30.0`) ni la naranja (`20.0`). Tienes que decirle que para llegar a la derecha, use al Router2 (`10.0.0.2`).

- **Ruta para red Azul:** `ip route 192.168.30.0 255.255.255.0 10.0.0.2`
- **Ruta para red Naranja:** `ip route 192.168.20.0 255.255.255.0 10.0.0.2`

*(En este caso, como solo hay un camino de salida, también podrías usar una sola ruta por defecto: `ip route 0.0.0.0 0.0.0.0 10.0.0.2`)*.

### 2. En el Router2 (Derecha)

A este le pasa lo contrario: conoce la azul y la naranja, pero **no conoce** la rosa ni la verde. Debe enviar ese tráfico al Router1 (`10.0.0.1`).

- **Ruta para red Rosa:** `ip route 192.168.1.0 255.255.255.0 10.0.0.1`
- **Ruta para red Verde:** `ip route 192.168.2.0 255.255.255.0 10.0.0.1`

### Resumen para tu red:

- **Router1:** "Para llegar a las redes de la derecha, envíalo a `10.0.0.2`"
- **Router2:** "Para llegar a las redes de la izquierda, envíalo a `10.0.0.1`"

------

# Router-on-a-Stick - Router (y Switch para el enlace troncal)

> - Técnica para enrutar entre VLANs usando un único enlace físico del router al switch, con subinterfaces lógicas
> - Poner el default-gateway de las VLAN en las subinterfaces .1
> - **Cable en modo Trunk (Router normal):** Sí requiere Router-on-a-Stick
>
>   **Interfaz con `no switchport` (Switch Capa 3):** No requiere Router-on-a-Stick; solo requiere una IP y una tabla de rutas

---

**"Active la interfaz física principal sin asignarle IP."**

```
interface g0/0
no shutdown
```
---

**"Cree la subinterfaz para la VLAN 10"**

```
interface g0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
```
- `dot1Q 10` → asocia la subinterfaz a la VLAN 10
- La IP es el default gateway de los equipos en esa VLAN

---

**"La VLAN 99 es la VLAN nativa, cree su subinterfaz"**

```
interface g0/0.99
encapsulation dot1Q 99 native
ip address 192.168.99.1 255.255.255.0
```
- Añade `native` si esa VLAN es la nativa del troncal

---

**"Configure el router para que retransmita las peticiones DHCP de la VLAN al servidor 10.0.0.5."**
```
interface g0/0.10
ip helper-address 10.0.0.5
```

---

**"Verifique que las subinterfaces están activas y tienen IP."**
```
show ip interface brief
```

------

# Enrutamiento dinámico - OSPF - Router - Varios routers conectados entre sí

> - ⚠ Si haces esto, no haces enrutamiento estático
> - Válido para Switches de Capa 3 con el puerto en `no switchport`
> - ⚠ No hace falta que estén los routers conectados directamente, **puede haber un switch de por medio**
> - **Tener en cuenta que publicar las IP de las redes que van al servidor de internet (las 200), es una mala práctica, tendrías suspenso**
> - Recuerda que si cambias el `router-id` en un proceso OSPF que ya está funcionando, los cambios no se aplican hasta que reinicies el proceso con `clear ip ospf process`
> - `passive-interface GigabitEthernet0/2`: Asegúrate de que esa es la interfaz que va a los PCs. Si pones como pasiva la interfaz que conecta con otro router, OSPF dejará de funcionar entre ellos, pones entonces `no passive-interface GigabitEthernet0/2`

Comprobar que están declaradas las interfaces

```
int g0/1
no switchport							#Para Switch de Capa 3
ip address 192.168.255.2 255.255.255.0
```

- **De Router a Router:** Creas una red nueva (como la `.50` o la `.60`) que sirve de "puente" solo para ellos
- **De Router a Switch:** Usas la red de la **VLAN** (la `.10`)

Comenzamos por poner

```
show ip route
```

## En un **Switch de Capa 3** al poner el comando si ves esto

```
Default gateway is not set

Host               Gateway           Last Use    Total Uses  Interface
ICMP redirect cache is empty
```

Tienes que activar el enrutamiento IP

```
ip routing
```

Entonces ya verás esto y podremos seguir el proceso

```
C       172.16.1.0/24 is directly connected, GigabitEthernet0/2
L       172.16.1.1/32 is directly connected, GigabitEthernet0/2
        192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.1.0/24 is directly connected, Loopback0
L       192.168.1.1/32 is directly connected, Loopback0
        192.168.255.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.255.0/30 is directly connected, GigabitEthernet0/1
L       192.168.255.2/32 is directly connected, GigabitEthernet0/1
```

Declaramos el ID process del OSPF

```
router ospf 10
```

Configure los router IDs en los routers de la siguiente manera

```
router-id 4.4.4.4
```

Ahora, una vez declaradas las interfaces, las añadimos al OSPF usando sus IP's + Máscara y añadimos las direcciones que tengan la letra C del comando

```
network 172.16.1.0 0.0.0.255 area 0
network 192.168.1.0 0.0.0.255 area 0
network 192.168.255.0 0.0.0.3 area 0
```

- ⚠En ocasiones, no olvidar declarar las IP's de las VLAN si no están en el listado, esto solo en caso que se haya hecho el inter vlan y puesto la IP en el **dispositivo en cuestión**
- Tener en cuenta no publicar la IP's 200+ por que hay que distinguir entre las redes privadas y las públicas
- 

Para finalizar, podremos ver con una O las rutas que han aprendido por OSPF

```
show ip route 
     	172.16.0.0/16 is variably subnetted, 5 subnets, 2 masks
O       172.16.1.0/24 [110/3] via 192.168.255.5, 00:01:38, GigabitEthernet0/0
C       172.16.2.0/24 is directly connected, GigabitEthernet0/1.20
L       172.16.2.1/32 is directly connected, GigabitEthernet0/1.20
C       172.16.255.0/24 is directly connected, GigabitEthernet0/1.99
L       172.16.255.1/32 is directly connected, GigabitEthernet0/1.99
     	192.168.1.0/32 is subnetted, 1 subnets
O       192.168.1.1/32 [110/3] via 192.168.255.5, 00:01:38, GigabitEthernet0/0
        192.168.255.0/24 is variably subnetted, 3 subnets, 2 masks
O       192.168.255.0/30 [110/2] via 192.168.255.5, 00:01:38, GigabitEthernet0/0
C       192.168.255.4/30 is directly connected, GigabitEthernet0/0
L       192.168.255.6/32 is directly connected, GigabitEthernet0/0
```

**Se pone **siempre en el último router que tiene **una de las patas outside la IP 200+** para que se propague lo local con el internet, y puedan salir a internet

```
router ospf 10
default-information originate [always]
```

- ⚠ Si esto no funciona, hay que usar enrutamiento estático o **acabar haciendo NAT primero**
- Va en el último router con OSPF, en donde no has puesto en OSPF las IP's 200+ y que tiene NAT donde se pone inside y outside

### Un pequeño detalle técnico: El modificador `always`

A veces verás que la gente usa: `default-information originate always`

- **Sin el `always`:** El router solo anunciará la ruta por defecto si él mismo ya tiene una configurada (la famosa `ip route 0.0.0.0 0.0.0.0 ...`). Si su enlace a internet se cae y la ruta desaparece, dejará de anunciarla a los demás.
- **Con el `always`:** El router anunciará que él es la salida a internet **siempre**, incluso si su propia ruta de salida no está activa en ese momento.

---

**"Active el proceso OSPF con ID de proceso 1."**
```
router ospf 1
```
- El ID de proceso (1-65535) es local, no tiene que coincidir entre routers

---

**"Anuncie la red 192.168.10.0/24 en el área 0."**
```
router ospf [ID]
network 192.168.10.0 0.0.0.255 area 0
```
- La máscara es **wildcard** (inversa): /24 → `0.0.0.255`, /28 → `0.0.0.15`
- `area 0` es el área backbone (la más común en examen)

---

**"Fije el Router-ID manualmente a 1.1.1.1."**
```
router-id 1.1.1.1
```
- Evita que OSPF elija el RID automáticamente

**"Evite que OSPF envíe hellos por el puerto g0/1 (donde hay PCs)."**

El ejercicio dice que no envíes actualizaciones donde no sean necesarias. Como en la Loopback no hay ningún router al otro lado escuchando, debes marcarla como **pasiva** dentro de OSPF

```
router ospf [ID]
passive-interface g0/1
```

---

**"Haga que todos los puertos sean pasivos por defecto, y active OSPF solo en g0/0."**
```
passive-interface default
no passive-interface g0/0
```

---

**"Ajuste el ancho de banda de referencia a 1000 Mbps para que OSPF calcule bien el coste en redes Gigabit."**
```
auto-cost reference-bandwidth 1000
```
- Por defecto es 100 Mbps, lo que da coste 1 a FastEthernet y también a Gigabit

---

**"Propague la ruta por defecto (0.0.0.0/0) a todos los vecinos OSPF."**

```
ip route 0.0.0.0 0.0.0.0 [IP_DEL_ISP o Interfaz_de_salida]
router ospf [ID_PROCESO]
default-information originate
```
- Debe existir una ruta por defecto en el router para que funcione

---

**"Verifique los vecinos OSPF y las rutas aprendidas."**
```
show ip ospf neighbor
show ip route ospf
show ip ospf interface g0/0
```

------

Aquí tienes la secuencia exacta para una configuración OSPF básica y funcional. He ordenado los comandos para que los copies y pegues directamente, asumiendo que tu **ID de proceso es 1** y tu **área es 0**.

### 1. Configurar la salida a Internet (Ruta estática)

Esto se hace en el router que tiene el cable que va hacia fuera, (el ISP) por ejemplo

```
conf t
ip route 0.0.0.0 0.0.0.0 [IP_DEL_ISP]
```

### 2. Configurar OSPF y las redes

Ahora entras al "cerebro" del protocolo para decirle qué redes quieres que los demás routers conozcan.

```
router ospf 1
router-id 1.1.1.1  (Opcional: es el nombre del router para OSPF)
network 192.168.10.0 0.0.0.255 area 0
```

- **network**: La red que quieres anunciar
- **0.0.0.255**: Es la **Wildcard** (lo contrario a la máscara). Si tu máscara es 255.255.255.0, la wildcard es 0.0.0.255
- **area 0**: El área principal (el "corazón" de OSPF)

### 3. Compartir la ruta de Internet con los demás

Estando todavía dentro de `router ospf 10`, lanzas el comando que ya conoces:

```
 router ospf 10
 default-information originate
 exit
```

------

### Resumen de comandos de verificación

Para saber si lo has hecho bien, usa estos dos comandos fuera del modo configuración (`exit` o `end`):

|       **Comando**       | **Para qué sirve**                                           |
| :---------------------: | ------------------------------------------------------------ |
| `show ip ospf neighbor` | Para ver si el router ha hecho "amigos" (vecinos) con los otros routers |
|     `show ip route`     | Para ver si aparecen rutas con una **O**. Si ves una **O\*E2**, es que la ruta de internet ha llegado |

**Nota importante sobre la Wildcard:**

Si tu red es `192.168.10.0` con máscara `/24` (255.255.255.0), la wildcard que debes poner es `0.0.0.255`. La que tú pusiste (`0.0.0.3`) solo sirve para enlaces muy pequeños de dos routers (máscara `/30`).

## Configure los interfaces punto a punto para que no se desencadene la elección DR/BDR

Debes aplicar esto en los enlaces que unen a los routers entre sí:

- **En el Router ISP:** interfaces `G1/0` y `G0/0`
- **En el Router A:** interfaz `G0/1`
- **En el Router B:** interfaz `G0/0`

**En el Router ISP:**

```
interface GigabitEthernet 1/0
 ip ospf network point-to-point
exit
interface GigabitEthernet 0/0
 ip ospf network point-to-point
```

**En el Router A:**

```
interface GigabitEthernet 0/1
 ip ospf network point-to-point
```

**En el Router B:**

```
interface GigabitEthernet 0/0
 ip ospf network point-to-point
```

### ¿Qué ganas con esto?

1. **Velocidad:** Los routers se vuelven vecinos instantáneamente sin esperar a la elección.
2. **Eficiencia:** No se generan paquetes innecesarios para decidir quién manda en el cable.
3. **Orden:** En la tabla de vecinos (`show ip ospf neighbor`), verás que el estado pasa a ser **FULL** a secas, en lugar de **FULL/DR** o **FULL/BDR**

**Meter un PC de Administración de VLAN 99 en la misma VLAN 99 - Hacer después Pool y Excluded DHCP**

Para que este PC pueda "hablar" con los switches que están abajo (VLAN 99 física), el **Router ISP** tiene que avisar a los demás que esa red existe con la IP de la red en cuestión, no la IP de la interfaz

```
router ospf 10
network 192.168.254.0 0.0.0.255 area 0
```

**Configure los LoopBack (Lo0) para que OSPF publique la red. Estos puertos simulan otras LAN**

Primero, asignas la dirección IP a la interfaz lógica **Lo0**. Esta interfaz nunca se cae, por lo que es ideal para simular redes o identificar routers

```
interface Loopback0
ip address 192.168.1.1 255.255.255.0
```

Para que el resto de la red conozca estas "LAN", debes incluirlas dentro del proceso OSPF en el router

```
router ospf [id]
network 192.168.1.0 0.0.0.255 area 0
```

**Configure los interfaces punto a punto para que no se desencadene la elección DR/BDR**

Este router tiene dos interfaces hacia los otros routers. Interfaz hacia Router A:

```
interface GigabitEthernet1/0
ip ospf network point-to-point
```

**Configure OSPF para que las actualizaciones de enrutamiento no se envíen a las redes donde no sean necesarias**

Para cumplir con este requisito, debes configurar **interfaces pasivas**. Esto evita que el router envíe paquetes "Hello" de OSPF por puertos donde solo hay dispositivos finales (como PCs o impresoras), ahorrando recursos y mejorando la seguridad

```
router ospf 10
passive-interface GigabitEthernet0/2 -- Interfaz donde hay un ordenador
passive-interface Loopback0
```

**Configure una ruta predeterminada a internet mediante el argumento interface de salida**

Para configurar la ruta predeterminada (o de último recurso) hacia Internet, debes realizar la configuración en el **Router ISP**, que es el dispositivo que tiene la conexión directa hacia el mundo exterior a través de su interfaz **G2/0**

```
ip route 0.0.0.0 0.0.0.0 GigabitEthernet2/0
```

**0.0.0.0 0.0.0.0** es la **ruta predeterminada**. Si el router busca en su tabla y **no encuentra** una coincidencia exacta para el destino, usa esta ruta como último recurso. Es como un cartel que dice: *"Si no sabes a dónde ir, tira por aquí (hacia Internet)"*

Para verla en el router, se usa el comando:

```
show ip route
```

Si lo ejecutas, verás una lista de redes. Las que tienen una **"C"** son conectadas, las **"O"** son aprendidas por OSPF y la **"S\*"** es la estática predeterminada que acabas de crear

**Distribuya automáticamente la ruta predeterminada a todos los routers de la red**

Para que el resto de routers (Router A y Router B) sepan cómo salir a Internet sin tener que configurar la ruta a mano en cada uno, debes usar OSPF para que el **Router ISP** les "avise".

Sigue estos pasos en el **Router ISP**:

```
conf t
router ospf 10
default-information originate
exit
```

### ¿Qué hace este comando?

El comando `default-information originate` le dice al Router ISP: *"Si tienes una ruta predeterminada (la que creamos antes hacia G2/0), anúnciala a todos tus vecinos OSPF"*.

### ¿Cómo compruebas que ha funcionado?

Ve a cualquier otro router (por ejemplo, al **Router A**) y escribe:

```
show ip route
```

Verás una nueva línea al final parecida a esta:

```
O*E2 0.0.0.0/0 [110/1] via 192.168.255.1, ...
```

- **O**: Aprendida por OSPF.
- *****: Es una ruta predeterminada (candidata a ruta por defecto).
- **E2**: Significa que viene de fuera del área OSPF (es una ruta externa).

De esta forma, cuando un PC de la VLAN 10 quiera ir a Google, el Router A mirará su tabla, verá esa ruta "O*" y enviará el paquete hacia el ISP automáticamente

Si te refieres a **"limpiar" o borrar la configuración de OSPF** para empezar de cero porque te está dando problemas (algo muy común cuando las rutas se quedan "atascadas"), tienes dos formas de hacerlo: la rápida y la total.

**El "Reinicio" (Resetear los procesos)**

Si los comandos están bien pero las rutas no aparecen en los otros routers, a veces el OSPF se queda "atontado". Este comando obliga a OSPF a volver a hablar con sus vecinos y recalcular todo el mapa sin borrar tus líneas de configuración.

```
clear ip ospf process
```

- Te preguntará: `Reset ALL OSPF processes?:`
- Escribe **yes** y pulsa Enter.

------

**El "Flashazo" total (Borrar la configuración)**

Si quieres eliminar por completo el proceso OSPF de un router para escribir los comandos `network` desde cero, usa este comando en el modo de configuración global:

```
conf t
no router ospf 1
```

*(Usa el número de proceso que hayas creado, normalmente es el 1).*

**¿Qué pasa cuando haces esto?**

- Se borran todos los `network`, el `router-id` y cualquier configuración de OSPF.
- El router deja de anunciar sus redes a los demás.
- Es la mejor opción si te has liado con las **Wildcards** o has metido redes que no querías

------

# DHCP - Router o (Switch de capa 3) - ip helper-addres luego

> - El router actúa como servidor DHCP y reparte IPs automáticamente
> - ⚠ Nunca hacer DHCP a un Servidor de Internet

---

**Si el router no responde a ninguna petición DHCP, puede que el servicio esté apagado. Se activa así**

```
service dhcp
ip routing
```

**1** **"Excluya las IPs del .1 al .10 para que DHCP no las reparta (para routers, impresoras, etc.)."**

```
ip dhcp excluded-address 192.168.10.1 192.168.10.10
```
- ⚠️ Hacer esto **antes** de crear el pool

---

**2 "Cree un pool DHCP LAN-VLAN10 para la red 192.168.10.0/24 - No olvidar declarar las IP's de las VLAN"**

```
ip dhcp pool LAN-VLAN10
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 8.8.8.8
```
- `network` → red definida
- `default-router` → IP interfaz del Router Server DHCP que va al dispositivo receptor

  - **Router-on-a-Stick**, declarar ip acabada en .1 de la subinterfaz de esa VLAN

    **HSRP**, si el servidor DHCP ha tenido HSRP, hay que declarar `192.168.10.1` de la `standby 1 ip` de esa VLAN

- `dns-server` → servidor DNS que recibirán los clientes
- **Si entre el Router DHCP y el dispositivo final, hay otro router de por medio, hay que hacer ip helper-address en ese router de por medio**

Si ves que un PC tiene la `.1`, usa `clear ip dhcp binding *` para resetearlo después de poner la exclusión

**Ver la configuración DHCP en el router**

```
show run | sc dhcp
```

---

**"Configure la duración del arrendamiento a 2 días"**

```
lease 2
```
- Formato: `lease días horas minutos`

---

**"Configure la interfaz del router para obtener su IP por DHCP." (router como cliente)**
```
interface g0/0
ip address dhcp
```

---

**"Verifique qué IPs ha repartido el servidor DHCP"**

```
show ip dhcp binding
```

---

**"Verifique si hay conflictos de IPs en el servidor DHCP"**

```
show ip dhcp conflict
```

**Configura DHCP relay donde sea necesario**

Lo que el ejercicio te quiere decir es que el **Router ISP** es el único que tiene los "pools" (Por que es el servidor DHCP) de direcciones (las cajas con IPs), pero entre el dispositivo que va a recibir la dirección DHCP automática y el router servidor dhcp, hay otro router, en ese router tienes que poner este comando

## IP Helper Address -  Último router  o Switch Capa 3 hacia el dispositivo que va a recibir la IP automática - DHCP Relay

Configuración en el Router normal

```
interface g0/1.20 - DISPOSITIVO ACTUAL - interfaz que va hacia dispositivo
interface g0/1 - DISPOSITIVO ACTUAL - interfaz que va hacia dispositivo
ip helper-address 192.168.60.1 - DISPOSITIVO ANTERIOR - IP de interfaz que va hacia dispositivo
```

- interface GigabitEthernet0/1.20 - En caso de que sea Trunk y venga de Router and Stick
- ip helper-address 192.168.60.1 es la IP del dispositivo anterior, pata outside
- ⚠ Si no funciona después de DHCP y el helper-address, revisar OSPF
- El dispositivo puede ser un router o un switch de capa 3
- activar ip routing en este último router
- Para que el DHCP funcione entre redes distintas, los paquetes deben poder ir y volver, para eso se necesita 

Configuración en el Switch Capa 3

```
ip routing
interface fa0/3
switchport trunk encapsulation dot1q
switchport mode trunk

interface vlan 10
ip address 192.168.10.1 255.255.255.0
ip helper-address 5.0.0.3

interface vlan 20
ip address 192.168.20.1 255.255.255.0
ip helper-address 5.0.0.3
```

- interface fa0/3 es la que va hacia los dispositivos, por eso se pone en modo trunk, primero encapsulation
- Declaramos las interfaces VLAN
- Luego hacemos ip helper address en Switch Capa 3 - Server DHCP > la IP de la interfaz que va hacia dispositivos
- El Switch de Capa 3 debe tener las `interface vlan .1`. Esas interfaces actúan como "orejas" que escuchan a los PCs y, gracias al `ip helper-address`, reenvían la petición
- ⚠ La petición de IP es un viaje de **ida y vuelta** (**Hacer ip route** en el dispositivo Server DHCP si hay un switch capa 3 de por medio)
- El `ip route` en el Switch-Servidor le enseña el camino de regreso: *"Para llegar a la 192.168.10.0, envíaselo al Switch central (IP 5.0.0.2)*

**Añadir PC de Administrador para que reciba su propia dirección IP** - **Hacer antes OSPF**

El PC3 está conectado a la interfaz **G3/0** del Router ISP. Para que reciba IP automáticamente, el Router ISP necesita saber qué números repartir en esa "boca" concreta

Crear el Pool de DHCP para el Administrador

```
ip dhcp pool admin_pool
network 192.168.254.0 255.255.255.0
default-router 192.168.254.1
dns-server 209.128.128.10
```

No queremos que el DHCP le asigne a un PC la misma IP que tiene la interfaz del router (`.1`), porque habría un conflicto

```
ip dhcp excluded-address 192.168.254.1
```

**Configura DHCP relay donde sea necesario**

Para que los equipos de las VLAN reciban su dirección IP del **Router ISP**, debes configurar el **DHCP Relay** (agente de escucha) en los routers que actúan como puerta de enlace de esas redes. Esto es necesario porque las peticiones DHCP son mensajes de difusión (broadcast) y los routers no los dejan pasar por defecto

Debes usar la dirección IP del servidor DHCP, que en este caso es la IP del **Router ISP** en sus interfaces hacia la red interna.

### 1. En Router A (Para VLAN 10)

El Router A recibe las peticiones de la VLAN 10 por su interfaz **G0/2**. Debes decirle que las reenvíe a la IP del ISP (192.168.255.1 )

```
interface GigabitEthernet0/2
 ip helper-address 192.168.255.1
 exit
```

### 2. En Router B (Para VLAN 20 y 99)

El Router B gestiona estas VLANs mediante subinterfaces en su puerto **G0/1**. Debes configurar el reenvío en cada subinterface apuntando a la IP del ISP (192.168.255.5 ).

- **Para VLAN 20:**

  Bash

  ```
  interface GigabitEthernet0/1.20
   ip helper-address 192.168.255.5
   exit
  ```

- **Para VLAN 99:**

  Bash

  ```
  interface GigabitEthernet0/1.99
   ip helper-address 192.168.255.5
   exit
  ```

------

### ¿Qué acabas de hacer?

Has configurado el comando `ip helper-address`. Ahora, cuando un PC pide una IP:

1. El router local (A o B) captura ese paquete
2. Lo convierte en un paquete "unicast" (dirigido)
3. Lo envía directamente al **Router ISP** para que este le asigne una dirección del pool correspondiente

## DHCP Snooping - Protege los ataques ARP - Switches

> El DHCP Snooping debe estar activo en todos los switches donde haya puertos de "acceso" (los que van a los PCs)

- Se ejecuta en los **Switches de acceso** (donde se conectan físicamente los PCs).
- La regla de oro de la seguridad en redes es: **La seguridad se aplica lo más cerca posible del usuario.**

Para que DAI funcione, primero necesitas activar **DHCP Snooping** (es el requisito previo):

1. **Activa DHCP Snooping:**

   ```
   ip dhcp snooping
   ip dhcp snooping vlan 100,110,130  # Tus VLANs
   ```

   - Si te olvidas de una VLAN, los PCs de esa VLAN no recibirán IP

2. **Marca los puertos de confianza (los que suben hacia el servidor DHCP)**

   ```
   interface [interfaz que va al servidor dhcp desde el switch]
   ip dhcp snooping trust
   ip arp inspection trust
   ```

3. **Activa la inspección ARP en las VLANs:**

   ```
   ip arp inspection vlan 100,110,130
   ```

El límite de tasa (Rate Limit) - Importante para examen

Es una práctica de ingeniería obligatoria limitar cuántos paquetes DHCP puede enviar un puerto de usuario. Si no lo pones, un atacante puede saturar el switch. 

**Regla de oro:** Limita los puertos `untrusted`

```
interface range fa0/1 - 20  # Puertos de PCs
ip dhcp snooping limit rate 10
```

------

# NTP - Router y Switch

> Network Time Protocol. Sincroniza el reloj de todos los equipos de la red.

---

**"Configure el router para sincronizarse con el servidor NTP 10.0.0.1."**
```
ntp server 209.128.128.10
```

---

**"Dé prioridad al servidor NTP 10.0.0.1 frente a otros."**
```
ntp server 10.0.0.1 prefer
```

---

**"Configure este router como servidor NTP maestro en estrato 2 (sin acceso a internet)."**
```
ntp master 2
```
- Se usa en labs sin internet. Estrato 2 es lo habitual. No uses estrato 1.

---

**"Configure autenticación NTP con la clave 1 y contraseña 'cisco'."**
```
ntp authenticate
ntp authentication-key 1 md5 cisco
ntp trusted-key 1
ntp server 10.0.0.1 key 1
```

---

**"Configure la zona horaria a UTC+2"**
```
clock timezone CET 2
```

---

**"Verifique el estado de la sincronización NTP."**

```
show ntp status
show ntp associations
show clock
```

------

# Syslog - Monitorización y Gestión de Logs

Configura el envío de mensajes de error y eventos a un servidor externo para saber qué pasa en la red

- Se hace en modo de configuración global en **todos** los equipos (**Routers y Switches**)

### **Syslog**

**"Envíe todos los mensajes de log al servidor Syslog 192.168.1.100."**

```
logging host 192.168.1.100
logging trap notifications
```

- `notifications` → Nivel de log 5 (avisa de cambios de estado en interfaces, etc.).

------

### **Service Timestamp (Marca de Tiempo)**

**"Añada la fecha y hora exacta a cada mensaje de log para auditorías"**

```
service timestamps log datetime msec
service timestamps debug datetime msec
```

- **¿Para qué sirve?** Para que el log no diga "hace 5 minutos", sino "May 05 23:45:01". Muy útil con NTP configurado.

------

### **Verificación de Seguridad y Logs**

```
show ip dhcp snooping
show ip arp inspection
show logging
```

------

# ACL - Router

Para configurar una ACL, siempre tienes que pensar en el **primer punto de contacto** entre el paquete y el router

> - **El "Implicit Deny":** Recuerda siempre que al final de cada ACL hay un `deny ip any any` invisible. Si creas una ACL para denegar una sola IP y no pones un `permit ip any any` al final, bloquearás **todo** el tráfico de la red

- **ACL Estándar:** Se coloca lo más cerca posible del **destino**
- **ACL Extendida:** Se coloca lo más cerca posible del **origen**

### El Orden de las Reglas (Estrategia de Ingeniero)

1. **Excepciones VIP:** (Permitir hosts específicos como el `Server-V`)
2. **Prohibiciones:** (Reglas `deny`)
3. **Permisos Generales:** (Reglas `permit` para servicios como Web/DNS)
4. **Cierre de Lista:** (La regla final que decide si el resto pasa o muere)

### Protocolos y Puertos

- **DNS:** Usa siempre **UDP** port 53 (casi nunca TCP)
- **HTTP y HTTPS:** Puerto 80 (HTTP) y 443 (HTTPS) son siempre **TCP**
- **FTP:** Siempre son dos puertos, **20 y 21**, y siempre **TCP** o poner `eq ftp` al final
- **SSH:** Puerto 22 **TCP**
- **Telnet**: Puerto 23 **TCP**
- **ICMP**: No tiene puertos, solo pone icmp
- **SMTP**: Puerto 25 **TCP**
- **POP3**: Puerto 100 **TCP**
- **IMAP** Seguro: Puerto 993 **TCP**
- **IMAP** No seguro: Puerto 143 **TCP**
- **TFTP**: Puerto 69 **UDP**

### Consejos:

- Si aplicas una ACL en la interfaz que recibe a los PCs, **tienes que permitir también el tráfico DNS (UDP 53)**

---

## ACL Estándar

⚠️ **1 al 99:** Son **ACL Estándar**. Solo pueden filtrar por "quién envía" (IP de origen)

**"Bloquee al host 192.168.1.5 y permita el resto"**

```
access-list 1 deny host 192.168.1.5
access-list 1 deny 192.168.1.5 0.0.0.255
access-list 1 permit any -- anula todo lo anterior en caso de que sea permit lo de antes
```
- ⚠️ Añade `permit any` si quieres dejar pasar el resto

---

**"Configure una instrucción para que la ACL 1 permita cualquier dirección que pertenezca a 172.16.0.0/16"**

```
access-list 1 permit 172.16.0.0 0.0.255.255
```

**¿Qué comandos usaría para permitir que todos los hosts en la red 192.168.10.0/24 accedan a la red?**

```
access-list 1 remark Allow R1 LANs Access
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 deny any
```

---

## ACL Extendida

- ⚠️ **100 al 199:** Son **ACL Extendidas**. Estas permiten filtrar por "quién envía", "a quién va" (destino) y "qué protocolo usa" (puerto 80 para web, puerto 443 para HTTPS, etc.)
- **No puedes usar la palabra `host` y una wildcard al mismo tiempo**

|    **Si el enunciado dice...**    | **La red es...** |         **Al final pones...**          |
| :-------------------------------: | :--------------: | :------------------------------------: |
|   "Se permite salir a Internet"   |   **Abierta**    |          `permit ip any any`           |
| "El resto de la red debe navegar" |   **Abierta**    |          `permit ip any any`           |
|  "Solo se permite el tráfico X"   |   **Cerrada**    | **NADA** (dejas que el router bloquee) |
| "Por seguridad, denegar lo demás" |   **Cerrada**    |     **NADA** (o `deny ip any any`)     |

**El PC Administrador (172.16.0.50) tiene acceso total (sin restricciones**

```
access-list 101 permit ip host 172.16.0.50 any
```

**Toda la red tiene prohibido el Telnet (puerto 23) al servidor 172.16.0.100**

```
access-list 101 deny tcp 172.16.0.0 0.0.0.255 host 172.16.0.100 eq 23
```

**"Bloquee el tráfico HTTP (puerto 80) desde cualquier origen al servidor 10.0.0.5"**

```
access-list 101 deny tcp any host 10.0.0.5 eq 80
access-list 101 permit ip any any
```
**"Bloquee el tráfico HTTP (puerto 80) desde ordenador origen 192.168.30.2 al servidor 10.0.0.5"**

```
access-list 101 deny tcp host 192.168.30.2 host 10.0.0.5 eq 80
access-list 101 permit ip any any
```

---

## **"Bloquee el ping desde 192.168.1.0/24 hacia cualquier destino"**

```
access-list 101 deny icmp 192.168.1.0 0.0.0.255 any echo
access-list 101 permit ip any any
```

| **Comando**              | **Resultado**                                                |
| :----------------------- | :----------------------------------------------------------- |
| `deny icmp ... any echo` | Solo bloquea que la red **inicie** un ping. Pueden responder si les preguntan |
| `deny icmp ... any`      | Bloquea **todo**. No pueden iniciar pings, ni responder pings, ni usar traceroute, ni recibir mensajes de error de red |

En un entorno real o de examen, siempre es mejor **ser específico**. Si solo quieres evitar el ping, añade `echo`. Si quieres que esa red sea "invisible" y no sepa nada del estado de la red, quita el `echo`

**Niega el tráfico de la Azul 192.168.1.0 con destino a la Verde 192.168.0.0**

```
access-list 101 deny ip 192.168.1.0 0.0.0.255 192.168.0.0 0.0.0.255
access-list 101 permit ip any any
```

**Sólo el equipo 192.168.0.10 puede administrar a través de SSH**

```
access-list 101 remark SSH
access-list 101 permit tcp host 192.168.0.10 any eq 22
access-list 101 deny ip any any
```

**Sólo el equipo 192.168.0.10 de la red 192.168.0.0 puede administrar a través de SSH los router R1 y ISP**

```
access-list 101 remark SSH

#Permitimos específicamente ese host
access-list 101 permit tcp host 192.168.0.10 any eq 22

#Bloqueamos acceso al resto de la misma red y otras redes
access-list 101 deny tcp 192.168.0.0 0.0.0.255 any eq 22
access-list 101 deny tcp 192.168.1.0 0.0.0.255 any eq 22
```

**Los equipos de la VLAN 10 sólo podrán salir a internet con HTTP y HTTPS**

Usamos un número entre 100 y 199. La red de la VLAN 10 es `172.16.1.0`

```
# 1. Definir la lista (ACL Extendida)
access-list 110 permit tcp 172.16.1.0 0.0.0.255 any eq 80
access-list 110 permit tcp 172.16.1.0 0.0.0.255 any eq 443

# (Opcional pero recomendado) Permite DNS para que carguen las webs por nombre
access-list 110 permit udp 172.16.1.0 0.0.0.255 any eq 53

# 2. El portero en la puerta de entrada
interface GigabitEthernet 0/2
ip access-group 110 in
```

**Los equipos de la VLAN 20 tienen permitido todo el tráfico a internet excepto FTP**

```
# 1. Prohibimos el protocolo FTP (puertos 20 y 21)
access-list 120 deny tcp 172.16.2.0 0.0.0.255 any eq 20
access-list 120 deny tcp 172.16.2.0 0.0.0.255 any eq 21

# 2. Permitimos todo lo demás
access-list 120 permit ip 172.16.2.0 0.0.0.255 any
```

---

## ACL Nombrada (la más recomendada)

**"Cree una ACL nombrada EL-NOMBRE que bloquee HTTP al servidor 10.0.0.5."**

```
ip access-list extended/standard EL-NOMBRE
deny/permit tcp any host 10.0.0.5 eq 80
permit ip any any
```
- Las nombradas permiten editar líneas individuales con número de secuencia

**Cree una ACL estándar para permitir que el tráfico de todos los hosts en la red 192.168.40.0/24 tenga acceso a todos los hosts en la red 192.168.10.0/24. Además, solo debe permitir el acceso del host PC-C a la red 192.168.10.0/24. El nombre de esta lista de acceso debe ser BRANCH-OFFICE-POLICY**

```
ip access-list standard BRANCH-OFFICE-POLICY
permit host 192.168.30.3
permit 192.168.40.0 0.0.0.255
```

---

## Aplicar la ACL - Permitir por ACL por UDP el DHCP con puerto 67

**"Aplique la ACL 1 o la nombrada como EL-NAME al tráfico entrante en la interfaz g0/0"**

```
interface g0/0
ip access-group 1 in
ip access-group EL-NAME out
```
- `in` → filtra el tráfico que entra al router por esa interfaz
- `out` → filtra el tráfico que sale del router por esa interfaz

---

**"Permita el acceso SSH/Telnet (VTY) solo desde la red 192.168.1.0/24 o el host 192.168.0.10"** - 

⚠ Configurar primero SSH para esto

Luego pones los comandos en los dispositivos en si que quieres gestionar por SSH y router que gestione el PC de origen

```
Permitir una red: 
access-list [99] permit 192.168.1.0 0.0.0.255

Permitir un host: 
access-list [99] permit host 192.168.0.10

line vty 0 15
access-class [99] in
transport input ssh
login local
```
- En VTY se usa `access-class`, no `access-group`
- **`transport input ssh`**: Bloqueas Telnet y otros protocolos inseguros

---

**"Verifique las ACLs y cuántos paquetes han coincidido con cada regla."**

```
show access-lists
show running-config | include access
```

------

### Verifique las ACLs y cuántos paquetes han coincidido con cada regla

El comando `remark` es un **comentario**. Es como una nota o etiqueta que el ingeniero deja dentro de la configuración para saber qué hace esa lista de acceso más tarde

- **No filtra tráfico:** No permite ni deniega nada
- **No afecta al rendimiento:** El router lo ignora al procesar paquetes
- **Solo para humanos:** Sirve para que, cuando hagas un `show run`, entiendas por qué creaste esa ACL

```
access-list 1 remark Bloquear acceso de Office 1 a Office 2
access-list 1 deny 192.168.10.0 0.0.0.255
access-list 1 permit any
```

## Para saber exactamente qué número de secuencia tiene cada regla, tienes que usar el comando de visualización. 

### Los routers Cisco asignan estos números automáticamente (normalmente de 10 en 10)

Desde el modo de **privilegio** (el que tiene el símbolo `#`), escribe:

`show access-lists`

O si quieres ser más específico con una sola lista:

`show access-lists EL-NOMBRE` 

---

### Cómo interpretar el resultado
Cuando ejecutes el comando, verás algo como esto en la pantalla:

```bash
Standard IP access list EL-NOMBRE
    10 permit host 192.168.30.3
    20 permit 192.168.40.0, wildcard bits 0.0.0.255
    30 permit 209.165.200.224, wildcard bits 0.0.0.31
    40 deny any <-- al final siempre
```

* **El número de la izquierda (10, 20, 30, 40):** Es el número de secuencia.
* **La regla:** Es lo que el router está permitiendo o denegando.

### ¿Por qué es útil mirar esto?
1.  **Ver los "Matches":** Si haces el comando después de que haya tráfico en la red, el router te dirá cuántas veces se ha usado cada línea (por ejemplo: `8 matches`). Esto te sirve para saber si la regla está funcionando de verdad.
2.  **Saber dónde "hueco":** Si ves que tienes la línea 10 y la 20, y necesitas meter algo en medio, ya sabes que puedes usar el número 15.
3.  **Confirmar el orden:** Te aseguras de que el `deny any` está al final (en la posición más alta, como la 40) y no bloqueando el tráfico antes de tiempo

**Truco rápido:** Si solo quieres ver si la ACL está aplicada en una interfaz concreta, puedes usar `show ip interface g0/0/0` y buscar la línea que dice `Outgoing access list is...`.

## Modificar o borrar una regla en específico

Primero mirar las reglas que hay

```
show access-lists
Standard IP access list BRANCH-OFFICE-POLICY
    10 permit 192.168.40.0 0.0.0.255
    20 permit host 192.168.30.3
    30 permit 209.165.200.224 0.0.0.31
    40 deny any
```

Entras en el modo de configuración de esa ACL específica  y usas el comando `no` seguido del número para borrarla:

```
ip access-list standard BRANCH-OFFICE-POLICY
no 20
```

Qudaría así con al regla 20 borrada:

```
show access-lists
Standard IP access list BRANCH-OFFICE-POLICY
    10 permit 192.168.40.0 0.0.0.255
    30 permit 209.165.200.224 0.0.0.31
    40 deny any
```

Creas la regla correcta usando el mismo número:

```
ip access-list standard BRANCH-OFFICE-POLICY
no 20
20 permit 192.168.30.0 0.0.0.255
```

Quedaría así ahora:

```
show access-lists
Standard IP access list BRANCH-OFFICE-POLICY
    10 permit 192.168.40.0 0.0.0.255
    20 permit 192.168.30.0 0.0.0.255
    30 permit 209.165.200.224 0.0.0.31
    40 deny any
```

# NAT - Router - Último pegado a la 200 de Internet

> - Traduce IPs privadas (internas) a una IP pública para poder comunicarse con internet

---

**1. "Marcar diciendo cual es la que está fuera y cual es la que está dentro"**

```
interface g0/0
ip nat inside

interface g0/1
ip nat outside
```
Si son interfaces de un anterior Router and Stick

```
interface g0/1.100
ip nat inside

interface g0/1.200
ip nat inside
```

- ⚠️ Obligatorio antes de cualquier regla NAT

**2. "Configure PAT (NAT con sobrecarga) para que toda la LAN 192.168.1.0/24 salga a internet por la interfaz g0/1."**

```
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
access-list 1 permit 192.168.99.0 0.0.0.255
```
**3. Aplicamos cambios**

```
ip nat inside/outside source list [1] interface [g0/1] overload
```

- `overload` = PAT. Toda la LAN comparte una sola IP pública usando puertos diferentes
- **interface - la de salida y se pone `ip nat inside...`**
- El `overload` al final es lo que permite que muchos PCs usen una sola IP pública (técnicamente se llama PAT)
- ⚠ Permitimos de dentro hacia afuera, si por error incluyes la red pública en esa ACL, el NAT hará cosas raras y podrías perder la conexión al router desde fuera

**4. La Ruta por Defecto - Si no haces OSPF a la pública pero si el default information-originate y también NAT, pon en el último router que tiene en una pata outside la 200 de internet esto para propagar **

```
ip route 0.0.0.0 0.0.0.0 [IP]
```

- [IP] DISPOSITIVO SIGUIENTE dirección internet - interfaz entrante/"inside para entendernos"
- Ponemos 0.0.0.0 para no tener que estar declarando todas las IP's que van desde las VLAN hasta el router del internet o el servidor
- Siempre va en el router de borde (el que sale a Internet) apuntando al siguiente salto fuera de tu red
- Para saber los que he puesto `show run | include ip route`

**Para que el ping de Internet 10.0.0.2 a un PC de la VLAN 10 192.168.10.10"** - **Port Forwarding o NAT Estático**

```
ip nat inside source static 192.168.10.10 10.0.0.2
```

- NAT estático → siempre la misma traducción (para servidores)
- 10.0.0.2 - Pata outside que va hacia el dispositivo del cual es el punto 1 del ping
- Si no va, poner la IP Gateway en las dos direcciones, acabando en .1

---

**"Configure NAT dinámico con un pool de IPs públicas 203.0.113.1 - 203.0.113.10."**
```
ip nat pool MI-POOL 203.0.113.1 203.0.113.10 netmask 255.255.255.0
access-list 1 permit 192.168.1.0 0.0.0.255
ip nat inside source list 1 pool MI-POOL
```

---

**"Verificar las traducciones NAT activas"**

```
show ip nat translations
show ip nat statistics
show run | include nat
```

---

**"Borre todas las traducciones NAT."**

```
clear ip nat translation *
```

**Configura NAT con sobrecarga en el router ISP**

## Aplicar cambios

```
ip nat inside source list 1 interface [INTERFAZ_DE_FUERA] overload
```

1. **list 1**: Define quiénes tienen permiso para salir (tus VLANs)
2. **interface G2/0**: Dice qué interfaz usaremos para salir a internet
3. **overload**: La palabra mágica que permite que cientos de equipos usen esa misma interfaz al mismo tiempo
4. **"inside"**: Pillas la interface outside y pones inside en el comando

**Sólo desde los equipos de la VLAN 99 se podrá acceder por SSH para configurar los dispositivos**

| Switch planta 0 (P0) | VLAN 99 | 172.16.255.10/24 |
| :------------------: | :-----: | :--------------: |
| Switch planta 1 (P1) | VLAN 99 | 172.16.255.11/24 |
| Switch planta 2 (P2) | VLAN 99 | 172.16.255.12/24 |

La regla de oro en administración de redes es: **La seguridad se aplica en el destino**

En cada router y switch, crea una ACL estándar. La red de la VLAN 99 es `172.16.255.0/24`

```
access-list 10 permit 172.16.255.0 0.0.0.255
```

⚠Recuerda que para que el acceso por SSH funcione, antes debes haber configurado un **nombre de dominio**, generado las **claves RSA** y tener creados los usuarios en SSH

```
line vty 0 15
access-class 10 in
transport input ssh
```

**Los equipos de la VLAN 10 sólo podrán salir a internet con HTTP y HTTPS**

Crear la ACL Extendida

```
ip access-list extended FILTRO_VLAN10
permit tcp 172.16.1.0 0.0.0.255 any eq 80
permit tcp 172.16.1.0 0.0.0.255 any eq 443
deny ip 172.16.1.0 0.0.0.255 any
```

```
interface GigabitEthernet0/2
ip access-group FILTRO_VLAN10 in
```

- **Permit tcp... eq 80**: Permite la navegación web normal (HTTP)
- **Permit tcp... eq 443**: Permite la navegación web segura (HTTPS)
- **Deny ip... any**: Bloquea cualquier otro tipo de tráfico (como ping, FTP o correo) que intente salir de esa red

**Nota importante:** Al poner `deny ip any any` al final (que es implícito), también estarás bloqueando las peticiones **DNS** y **DHCP**. Si los PCs dejan de recibir IP o no resuelven nombres, tendrías que añadir:

- `permit udp 172.16.1.0 0.0.0.255 any eq 53` (para DNS)
- `permit udp any any eq 67` (para DHCP)

**Los equipos de la VLAN 20 tienen permitido todo el tráfico a internet excepto FTP**

El FTP utiliza dos puertos: el **20** (datos) y el **21** (control). Debemos bloquear ambos. Esta configuración se realiza en el **Router B**, que es el encargado de la VLAN 20

```
ip access-list extended FILTRO_VLAN20
deny tcp 172.16.2.0 0.0.0.255 any eq 20
deny tcp 172.16.2.0 0.0.0.255 any eq 21
permit ip 172.16.2.0 0.0.0.255 any
```

- **deny tcp... eq 20 y 21**: Bloquea específicamente cualquier intento de conexión FTP desde la red 172.16.2.0/24 hacia cualquier destino
- **permit ip... any**: Permite el resto de protocolos (navegación, ping, correo, etc.)

```
interface GigabitEthernet0/1.20
ip access-group FILTRO_VLAN20 in
```

------

# SSH - Router y Switch

> Permite acceder remotamente al CLI del router/switch de forma segura (cifrada). Alternativa segura a Telnet.

---

**"Configure el hostname y el dominio (necesarios para generar las claves RSA)."**
```
hostname R1
ip domain-name cisco.com
```
- ⚠️ Sin `hostname` y `ip domain-name` no se puede generar la clave RSA

---

**"Genere las claves RSA de 1024 bits para habilitar SSH."**
```
crypto key generate rsa
```
- Te preguntará el tamaño → escribe `1024` (o `2048` para más seguridad)

---

**"Cree un usuario administrador con contraseña cifrada y nivel de acceso total."**

Elegir uno u otro comando

```
username admin privilege 15 secret admin 
username admin password admin
```
- `privilege 15` → acceso completo (nivel máximo)
- `secret` → contraseña cifrada con MD5 (mejor que `password`)

---

**"Configure las líneas VTY para que solo acepten SSH y usen cuentas locales."**
```
line vty 0 15
transport input ssh
login local
```
- `login local` → usa los usuarios creados con `username`
- `transport input ssh` → bloquea Telnet, solo permite SSH

---

**"Fuerce SSH versión 2."**
```
ip ssh version 2
```

---

**"Configure un timeout de 60 segundos y máximo 3 intentos de login."**
```
ip ssh time-out 60
ip ssh authentication-retries 3
```

---

**"Verifique la configuración SSH."**
```
show ip ssh
show ssh
show running-config | begin line vty
```

------

# TFTP

> Protocolo para transferir archivos de configuración entre el router y un servidor externo. Se usa para hacer backups.

---

**"Guarde la configuración activa (running-config) en un servidor TFTP."**
```
copy running-config tftp:
```
- Te pedirá: IP del servidor TFTP, y nombre del archivo destino

---

**"Guarde la configuración de arranque (startup-config) en un servidor TFTP."**
```
copy startup-config tftp:
```

---

**"Restaure una configuración desde el servidor TFTP."**
```
copy tftp: running-config
```
- Te pedirá: IP del servidor TFTP, y nombre del archivo a descargar

---

**"Haga un backup de la imagen IOS del router al servidor TFTP."**
```
copy flash: tftp:
```

---

**"Actualice la imagen IOS del router descargándola del servidor TFTP."**
```
copy tftp: flash:
```

---

**"Verifique el contenido de la memoria flash."**
```
show flash:
```

------

# Wildcards

| **Prefijo CIDR** | **Máscara de  Subred** | **Máscara  Wildcard (ACL)** |
| :--------------: | :--------------------: | :-------------------------: |
|     **/32**      |    255.255.255.255     |  **0.0.0.0** (Host único)   |
|     **/31**      |    255.255.255.254     |         **0.0.0.1**         |
|     **/30**      |    255.255.255.252     |         **0.0.0.3**         |
|     **/29**      |    255.255.255.248     |         **0.0.0.7**         |
|     **/28**      |    255.255.255.240     |        **0.0.0.15**         |
|     **/27**      |    255.255.255.224     |        **0.0.0.31**         |
|     **/26**      |    255.255.255.192     |        **0.0.0.63**         |
|     **/25**      |    255.255.255.128     |        **0.0.0.127**        |
|     **/24**      |     255.255.255.0      |        **0.0.0.255**        |
|     **/23**      |     255.255.254.0      |        **0.0.1.255**        |
|     **/22**      |     255.255.252.0      |        **0.0.3.255**        |
|     **/21**      |     255.255.248.0      |        **0.0.7.255**        |
|     **/20**      |     255.255.240.0      |       **0.0.15.255**        |
|     **/19**      |     255.255.224.0      |       **0.0.31.255**        |
|     **/18**      |     255.255.192.0      |       **0.0.63.255**        |
|     **/17**      |     255.255.128.0      |       **0.0.127.255**       |
|     **/16**      |      255.255.0.0       |       **0.0.255.255**       |
|      **/8**      |       255.0.0.0        |      **0.255.255.255**      |
|      **/0**      |        0.0.0.0         |     **255.255.255.255**     |

### Buscar en la tabla el cuántas menos líneas mejor , cubrir el rango sumando con la máscara wildcard, no priorizar la división

| **Tamaño del Bloque** | **Máscara de Subred** | **Prefijo**   **CIDR** | **Máscara Wildcard** |                   **¿Qué IP la admite?**                   |
| :-------------------: | :-------------------: | :--------------------: | :------------------: | :--------------------------------------------------------: |
|      **256 + 1**      |     255.255.255.0     |          /24           |    0.0.0.**255**     |                    Múltiplo de **256**                     |
|      **128 + 1**      |    255.255.255.128    |          /25           |    0.0.0.**127**     |                    Múltiplo de **128**                     |
|      **64 + 1**       |    255.255.255.192    |          /26           |     0.0.0.**63**     |                     Múltiplo de **64**                     |
|      **32 + 1**       |    255.255.255.224    |          /27           |     0.0.0.**31**     | Múltiplo de **32**  (32,64,96,128,160,192,224,256,288,320) |
|      **16 + 1**       |    255.255.255.240    |          /28           |     0.0.0.**15**     |  Múltiplo de **16**  (16,32,48,64,80,96,112,128,144,160)   |
|       **8 + 1**       |    255.255.255.248    |          /29           |     0.0.0.**7**      |     Múltiplo de **8**  (8,16,24,32,40,48,56,64,72,80)      |
|       **4 + 1**       |    255.255.255.252    |          /30           |     0.0.0.**3**      |      Múltiplo de **4**  (4,8,12,16,20,24,32,36,40,44)      |
|       **2 + 1**       |    255.255.255.254    |          /31           |     0.0.0.**1**      |         Solo si la IP es **par** (.2,  .4, .6...)          |
|       **1 + 1**       |    255.255.255.255    |          /32           |     **0.0.0.0**      |                     Múltiplo de **1**                      |
