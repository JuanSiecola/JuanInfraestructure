# ¡Bienvenido a Juane’s Home Lab! :wave:

¡Hola! Este sitio es mi **documentación técnica y laboratorio híbrido**. Aquí documento configuraciones, diagramas y prácticas reales de los conocimientos que voy adquiriendo a lo largo de mi formación para poder aplicarlos y probarlos en distintos escenarios.

El laboratorio fue desarrollado en **GNS3**, utilizando dispositivos de red como routers (Cisco y VyOS), switches (Cisco), firewall (PfSense) y servidores Debian 12. Además se implementaron instancias de proveedores de Cloud como Amazon Web Services y Oracle Cloud Infrastructure, las cuales tienen conectividad mediante Tailscale (VPN de Cero configuración) con mi topología local, con el propósito de diseñar una arquitectura de red híbrida y desplegar servicios en ambos entornos.

---

## :octicons-pin-16: Qué encontrarás aquí

- **Topología de red:** Diagrama de la topología implementada con su correspondiente servicio en cada servidor.  
- **Direcciones IP y VLANs:** Planes de direccionamiento, tablas y rangos implementados IpV4.  
- **Enrutamiento y protocolos:** OSPF, NAT, VPN, DNS, rutas estáticas, redistribución entre routers y firewall.  
- **Dispositivos individuales:** Configuraciones de routers, switches, servidores y firewall.  
- **Servicios desplegados con Docker:** Despliegue de servicios utilizando la herramienta docker-compose en equipos de la topología y VMs en la nube.

---

## :material-navigation-variant-outline: Navegación rápida

| Sección | Qué encontrarás |
|---------|----------------|
| [Topología](topologia/diagrama.md) | Diagrama general y componentes de la red |
| [Subredes](topologia/direccionamiento.md) | Planes de direccionamiento y subredes |
| [VLANs](topologia/vlans.md) | Configuración de dominios de broadcast |
| [Routing](topologia/routing.md) | Protocolos de enrutamiento y tablas |
| [Direcciones IP](topologia/direcciones-ip.md) | Direccionamiento IP por interfaz |
| [Dispositivos](dispositivos/index.md) | Configuración de routers, switches y servidores |
| [Servicios](servicios/index.md) | Servicios desplegados con Docker en servers |
| [AWS](AWS/aws.md) | Uso de la VM de AWS como proxy reverso para el ingreso a los servicios |
| [OCI](OCI/oci.md) | Uso de la VM de Oracle Cloud para correr Grafana y Prometheus con Docker |

---

## :fontawesome-solid-question: Cómo usar este sitio

1. Explora las secciones desde la barra lateral o los enlaces de arriba.  
2. Cada sección incluye información sobre lo implementado.  
3. Se incluyen imágenes, tablas y snippets de configuración.  

!!! warning 
    Este sitio documenta un entorno de laboratorio en desarrollo activo. 
    Ten en cuenta que los enlaces a servicios en vivo (Grafana, Kuma, NetBox) dependen de un host físico que **no opera en régimen 24/7**, por lo que es probable que los encuentres inactivos fuera de horario de pruebas.


