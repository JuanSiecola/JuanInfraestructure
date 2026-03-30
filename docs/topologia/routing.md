# Protocolos de Enrutamiento

## 1. Introducción

El enrutamiento es el proceso de dirigir el tráfico entre las diferentes subredes. En esta topología, el enrutamiento cumple tres funciones principales:

1. **Enrutamiento Inter-VLAN:** Permitir que las VLANs (Datacenter, Dev, Sysadmin) se comuniquen entre sí.
2. **Enrutamiento Core:** Interconectar las red VLANs con la red de servicios (Firewall y Management).
3. **Enrutamiento a Internet:** Proveer una ruta por defecto (Default Route) y Traducción de Direcciones de Red (NAT) para que los hosts internos accedan a Internet.

**Dispositivos de Enrutamiento:**

* **`JPROO2` (Router Cisco):** Se encarga del "Router-on-a-Stick" y es el gateway de todas las VLANs.
* **`JPROO1` (Router VyOS):** Actúa como router "core" o de tránsito, conectando el router Cisco con el firewall.
* **`JPFW01` (Firewall):** Es el firewall de borde. Se encarga de la seguridad y del NAT para salir a Internet.

---

## 2. Enrutamiento Inter-VLAN (Router-on-a-Stick)

El enrutamiento entre las VLANs de la red (1, 5, 10, 200) se maneja en el router Cisco `JPROO2` usando la técnica **Router-on-a-Stick**.

* **Dispositivo:** `JPROO2`
* **Interfaz Física:** `GigabitEthernet0/0`
* **Concepto:** Se crean sub-interfaces lógicas en `G0/0`, una por cada VLAN. Cada sub-interfaz se etiqueta con el ID de la VLAN (usando `encapsulation dot1q [vlan-id]`) y se le asigna la dirección IP del gateway para esa subred.

```
show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         10.10.1.2       YES NVRAM  up                    up
GigabitEthernet0/1         unassigned      YES NVRAM  administratively down down
GigabitEthernet0/2         unassigned      YES NVRAM  administratively down down
GigabitEthernet0/3         unassigned      YES NVRAM  up                    up
GigabitEthernet0/3.5       172.16.5.1      YES NVRAM  up                    up
GigabitEthernet0/3.10      172.16.10.1     YES NVRAM  up                    up
GigabitEthernet0/3.99      172.16.99.1     YES NVRAM  up                    up
GigabitEthernet0/3.200     172.16.200.1    YES NVRAM  up                    up
Loopback0                  10.255.255.2    YES NVRAM  up                    up
```

> **Nota:** La configuración detallada de las sub-interfaces se encuentra en la página de [Configuración de VLANs](vlans.md).

---

## 3. Enrutamiento Core (JPROO2 :octicons-arrow-both-24: JPROO1)

Esta es la conexión entre los dos routers principales sobre la red `10.10.1.0/30`.

Para intercambiar rutas automáticamente entre los routers `JPROO2` (Cisco) y `JPROO1` (VyOS), se utiliza el protocolo de enrutamiento dinámico **OSPF (Open Shortest Path First)**.

### 3.1 Configuración en `JPROO2` (Cisco)

`JPROO2` se configura para "anunciar" en OSPF todas sus redes conectadas directamente:

1.  Las redes de las VLANs (desde las sub-interfaces).
2.  La red del enlace vecino (hacia `JPROO1`).
3.  Tiene una **ruta estática por defecto** (`0.0.0.0/0`) que apunta al router vyos `JPRO01` (en `10.10.1.1`). 

```title="Configuración OSPF en JPROO2 (Cisco) y Static Route"
router ospf 1
 network 10.10.1.0 0.0.0.3 area 10
 network 10.255.255.2 0.0.0.0 area 10
 network 172.16.5.0 0.0.0.255 area 10
 network 172.16.10.0 0.0.0.255 area 10
 network 172.16.99.0 0.0.0.255 area 10
 network 172.16.200.0 0.0.0.255 area 10

 ! Ruta estática (hacia JPROO1)
 ip route 0.0.0.0 0.0.0.0 10.10.1.1
```

### 3.2 Configuración en `JPROO1` (VyOS)

1.  Publica su enlace con `JPROO2` (la red `10.10.1.0/30`).
2.  Publica su enlace con `JPFW01` (la red `10.10.0.0/30`)
3.  Tiene una **ruta estática por defecto** (`0.0.0.0/0`) que apunta al firewall `JPFW01` (en `10.10.0.1`).
4.  Tiene una **ruta estática** para redirigir el tráfico de la vlan1 para administrar y monitorear los switches por SSH y SNMP.

``` title="Configuración de JPROO1 (VyOS)"
protocols {
    ospf {
     area 10 {
         network 10.10.1.0/30
         network 10.10.0.0/30
         network 10.255.255.1/32
     }
 }
 static {
     route 0.0.0.0/0 {
         next-hop 10.10.0.1 {
         }
     }
     route 172.16.1.0/24 {
         next-hop 10.10.1.2 {
         }
     }
 }
```

---

## 4. Enrutamiento de Borde y NAT (JPFW01 - pfSense)

`JPFW01` (pfSense) actúa como el **firewall de borde** y el **gateway principal** hacia Internet. Su configuración de enrutamiento es fundamental para que la red interna funcione.

A diferencia de los routers internos que usan OSPF, el firewall se maneja principalmente con **rutas estáticas** y **NAT**.

### 4.1. Gateway por Defecto (Salida a Internet)

El primer paso en pfSense es definir los "gateways" o "próximos saltos" que el firewall conoce.

Se necesitan dos gateways:

1.  **Gateway por Defecto (WAN):** Para salir a Internet (marcado con el :material-web:).
2.  **Gateway Interno (HACIA_RED_INTERNA):** Para saber cómo "devolver" el tráfico a la red interna a través del router `JPROO1` (VyOS).

**Configuración de Gateways en JPFW01:**
![Gateway Pfsense WAN](../assets/pfsense/gatewaysPfsense.png)



### 4.2. Ruta Estática (Retorno al Interior)

Cuando un paquete vuelve de Internet, pfSense lo recibe. Pero, ¿cómo sabe pfSense dónde están las redes `172.16.5.0/24` o `172.16.200.0/24`?

**No lo sabe.** Debemos decírselo con **rutas estáticas**.

Creamos rutas para cada una de las redes internas (VLANs) que le dicen a pfSense: "Para alcanzar cualquier host en estas redes, envía el tráfico de vuelta a `JPROO1` (VyOS)" usando el gateway `HACIA_RED_INTERNA` que definimos en el paso anterior.

**Rutas Estáticas en JPFW01:**
![Static Routes Pfsense](../assets/pfsense/rutas-estaticas-pfsense.png)




### 4.3. NAT Saliente (Outbound NAT)

El Enrutamiento y el NAT son cosas distintas pero trabajan juntas.

1.  **Enrutamiento:** Decide *hacia dónde* va un paquete.
2.  **NAT:** *Cambia* la dirección IP de origen del paquete.

pfSense debe hacer **NAT Saliente** para "esconder" todas tus IPs privadas (172.16.x.x, 192.168.1.x, 10.10.x.x) detrás de su única IP pública (la de la interfaz `nat0`/`WAN`).

#### Modo Híbrido (Hybrid Mode)

Como se ve en la captura, se seleccionó el modo **"Hybrid Outbound NAT"**.
![NAT Pfsense 1](../assets/pfsense/nat1.png)

#### Esto permite en PfSense:
1.  **Reglas Manuales:** Permite crear reglas específicas (como la regla para `10.10.0.0/30` que se ve abajo en la imagen).
2.  **Reglas Automáticas:** pfSense sigue generando automáticamente las reglas para las redes internas (VLANs, etc.) que no coincidan con una regla manual.


#### Reglas Automáticas

La siguiente imagen muestra las reglas que pfSense genera automáticamente. La regla principal (la segunda en la lista) toma todas las redes internas (VLANs, Management, etc.) y las traduce (NAT) a la dirección de la `WAN` cuando salen a Internet.
![NAT Pfsense 2](../assets/pfsense/nat2.png)
