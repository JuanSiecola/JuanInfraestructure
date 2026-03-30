# Monitoreo con Grafana y Prometheus

![router-cisco](../assets/grafana/routercisco-img.png)

## 1. Resumen de Conectividad

El sistema de observabilidad opera en un modelo híbrido, en el cuál se ejecuta una instancia en Oracle Cloud, mientras que los dispositivos objetivos (Switches/Routers/Firewalls) se encuentran en una red local **(GNS3)**.

La comunicación entre la nube y el sitio local se asegura mediante **Tailscale (VPN)**.

---

## 2. Habilitar SNMP en dispositivos de red

Para habilitar el protocolo SNMP es los dispositivos ubicados en GNS3 se necesitan ejecutar ciertos comandos que se detallan debajo:

=== "Cisco"

    ```bash
    configure terminal
    snmp-server community public RO
    snmp-server location "Topologia red interna"
    exit
    write memory
    ```

=== "VyOS"

    ```bash
    configure
    set service snmp community public authorization ro
    set service snmp listen-address 10.255.255.1
    commit
    save
    exit
    ```

En caso de pfSense se puede hacer desde la webUI:

![snmp-pfsense](../assets/pfsense/snmp-pfsense.png)

---

## 3. Conectividad Segura (Tailscale)

Para alcanzar los objetivos SNMP que tienen IPs privadas, la VM de Oracle utiliza la red **Tailnet**, ya que
El firewall PfSense es quien comparte las redes, y la vm de Oracle acepta estas rutas para alcanzar estas redes. De esta manera tengo conectividad con los dispositivos de mi topología en GNS3.


![subnet-router-pfsense](../assets/pfsense/tailscale-routing-pfsense.png)

### 3.1 Flujo de Datos SNMP

1. **Origen:**  
   Prometheus inicia el scrape SNMP desde la VM en Oracle Cloud hacia un dispositivo de la red interna (por ejemplo `192.168.1.1`).

2. **Enrutamiento:**  
   La tabla de rutas del sistema operativo identifica que la red destino (`192.168.1.0/24`) está anunciada por Tailscale, por lo que el tráfico se envía a través de la interfaz `tailscale0`.

3. **Transporte:**  
   Los paquetes SNMP se encapsulan y cifran mediante Tailscale (WireGuard) y viajan por internet hasta el **Subnet Router**, que en este caso es el pfSense en GNS3, encargado de anunciar las rutas de la LAN.

4. **Destino:**  
   El pfSense recibe el tráfico desde Tailscale y lo enruta hacia el dispositivo de red correspondiente dentro de su red LAN, completando la solicitud SNMP.

!!! info
    Gracias a esta arquitectura, **no es necesario abrir el puerto UDP 161** en el firewall.

---

## 4. Reglas de Firewall (Security Lists OCI)

En las reglas de seguridad de la VCN, no es necesario abrir los puertos 3000 (Grafana), 9090 (Prometheus) ni (9116) como reglas de entrada. En cambio, para dejar público el servicio de Grafana en internet, como reglas de entrada se abren los puertos  80 y 443.

![firewall-rules](../assets/oracle/oracle-firewall-list.png)

### 4.1 Acceso a los Servicios

Para acceder al dashboard de grafana se utiliza la URL `https://grafana.js-lab-uy.duckdns.org`, pero para acceder a prometheus y snmp_exporter se necesita la dirección asignada a la VM por tailscale, indicando el puerto del contenedor correspondiente.

**Requisito:** El dispositivo desde donde se accede debe tener el cliente Tailscale conectado y estar dentro de la misma Tailnet.

| Servicio | URL de Acceso |
| :--- | :--- |
| **Grafana** | `https://grafana.js-lab-uy.duckdns.org` |
| **Prometheus** | `http://100.103.72.38:9090` |
| **SNMP Exporter** | `http://100.103.72.38:9116` |

![snmp-exporter](../assets/grafana/snmp_exporter.png)

![prometheus-snmp](../assets/grafana/prometheus_snmp.png)

![grafana-snmp](../assets/grafana/grafana-screenshot.png)

> *En el apartado de servicios -> grafana está todo más detallada su configuración, o haciendo click en el enlace* [Grafana](../servicios/grafana.md)