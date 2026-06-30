# 🌐 Proyecto de Interconexión de Sedes y Redundancia de Core (Sants - Atocha - Triana)

Este repositorio contiene la topología oficial, los ficheros de configuración y la documentación técnica de la infraestructura de red corporativa interconectada que unifica la sede principal de **Sants (Barcelona)** con las delegaciones de **Atocha (Madrid)** y **Triana (Sevilla)**.

La arquitectura asegura alta disponibilidad en el nodo central, enrutamiento dinámico global y aislamiento estricto de los servicios alojados en el Data Center.

---

## 🗺️ 1. Arquitectura de Direccionamiento IP y VLANs

Toda la segmentación avanzada por VLANs se concentra en los switches Core de la sede central (Sants). Las delegaciones remotas bajan directamente a sus switches locales mediante direccionamiento nativo en la interfaz física del router, optimizando el rendimiento y simplificando el mapa de enrutamiento WAN.

### Tabla de Direccionamiento Oficial (Estado Estable Original)

| Ubicación / Sede | Interfaz Física/Virtual | VLAN | Función del Segmento | Rango de Red (IP) | Pasarela (Gateway) |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **SANTS (Barcelona)** | Interfaz Virtual (SVI) | **10** | DMZ / Data Center (Servidores) | `192.168.10.0 /24` | `192.168.10.1` |
| **SANTS (Barcelona)** | Interfaz Virtual (SVI) | **20** | LAN Usuarios Locales (NOC) | `192.168.20.0 /24` | `192.168.20.1` |
| **SANTS (Barcelona)** | Interfaz Virtual (SVI) | **99** | Tránsito Interno (Core ↔ Router) | `10.0.99.0 /24` | *— (Interconexión)* |
| **ATOCHA (Madrid)** | Interfaz Física (`Gi0/0/1`) | *—* | LAN Única de Usuarios (PCs) | `192.168.10.0 /24` | `192.168.10.1` |
| **TRIANA (Sevilla)** | Interfaz Física (`Gi0/0/1`) | *—* | LAN Única de Usuarios (PCs) | `192.168.2.0 /24` | `192.168.2.1` |

> ⚠️ **Nota de Diseño:** La LAN de Atocha y la DMZ de Sants comparten el rango `192.168.10.0/24`. Al ser dominios de difusión completamente independientes separados por enlaces WAN punto a punto, coexisten sin conflictos de direccionamiento en sus respectivos switches locales.

### Enlaces WAN (Tránsito Punto a Punto)
* **Enlace Sants ↔ Atocha (Madrid):** `10.0.0.0 /30` (Router Sants: `10.0.0.1` / Router Atocha: `10.0.0.2`)
* **Enlace Sants ↔ Triana (Sevilla):** `10.0.1.0 /30` (Router Sants: `10.0.1.1` / Router Triana: `10.0.1.2`)

---

## ⚙️ 2. Ficheros de Configuración de Dispositivos Críticos

### A. Capa de Core y Distribución L3 (`SWITCH-CORE-SANTS_1`)
Mapea las VLANs y levanta las interfaces virtuales que sirven de Gateway, distribuyendo las rutas mediante OSPF.

```text
! --- ACTIVACIÓN DEL ENRUTAMIENTO IP ---
ip routing

! --- DEFINICIÓN DE VLANS ---
vlan 10
 name DMZ_SERVERS
vlan 20
 name LAN_SANTS
vlan 99
 name TRANSITO_CORE_ROUTER
exit

! --- CONFIGURACIÓN DE SVIs (GATEWAYS) ---
interface Vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

interface Vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit

interface Vlan 99
 ip address 10.0.99.2 255.255.255.0
 no shutdown
exit

! --- ENRUTAMIENTO DINÁMICO OSPF ---
router ospf 1
 log-adjacency-changes
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 10.0.99.0 0.0.0.255 area 0
exit
B. Router de Sede Remota (ISR4321_TRIANA)
Provee asignación dinámica de direcciones a los terminales de Sevilla mediante DHCP local y se adhiere a la red OSPF.

Plaintext
! --- INTERFAZ LAN LOCAL ---
interface GigabitEthernet0/0/1
 ip address 192.168.2.1 255.255.255.0
 no shutdown
exit

! --- EXCLUSIONES Y POOL DHCP ---
ip dhcp excluded-address 192.168.2.1 192.168.2.9
!
ip dhcp pool POOL_LAN_TRIANA
 network 192.168.2.0 255.255.255.0
 default-router 192.168.2.1
 dns-server 192.168.10.11
exit

! --- ENRUTAMIENTO WAN OSPF ---
router ospf 1
 log-adjacency-changes
 network 192.168.2.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.3 area 0
exit
C. Switch de Acceso Local (ACCESO-03-SANTS)
Muestra el modo de acceso plano asignado al puerto del usuario (como el puerto de las Laptops de oficina).

Plaintext
interface FastEthernet 0/5
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
🛠️ 3. Bitácora de Ingeniería y Resolución de Problemas (Troubleshooting)
Durante el despliegue de las configuraciones base, se documentó un caso crítico de análisis de comandos que sirve como lección aprendida de administración de IOS:

❌ El Incidente del Comando Inválido en el Router de Atocha
Al intentar segmentar o declarar rangos lógicos directamente en el modo global del Router de Madrid (ROUTER_ATOCHA), el sistema arrojó el siguiente error en la consola:

Plaintext
ROUTER_ATOCHA(config)# vlan 110
                        ^
% Invalid input detected at '^' marker.
🔍 Análisis Técnico del Error
Los routers Cisco tradicionales de la serie ISR no poseen la base de datos de conmutación de almacenamiento local de VLANs que sí integran los switches (vlan.dat). Por lo tanto, el comando ejecutable vlan <id> no existe en el modo de configuración global de un router.

La Solución Aplicada
La gestión de sub-interfaces lógicas para tráficos segmentados en Routers se realiza mediante encapsulamiento estricto por puerto (Sub-interfaces). Para resolverlo, se omitió la creación del objeto global y se atacó directamente la subinterfaz física aplicando el estándar IEEE 802.1Q:

Plaintext
ROUTER_ATOCHA(config)# interface GigabitEthernet 0/0/1.110
ROUTER_ATOCHA(config-subif)# encapsulation dot1Q 110
ROUTER_ATOCHA(config-subif)# ip address 192.168.110.1 255.255.255.0
💻 4. Infraestructura de Servicios Generales (Zona DMZ Sants)
Los servidores de producción se conectan directamente al switch ACCESO-01-SANTS bajo direccionamiento estático en la VLAN 10:

Servidor Web (WEB_SERVER): 192.168.10.10 / GW: 192.168.10.1

Servidor DNS (DNS_SERVER): 192.168.10.11 / GW: 192.168.10.1

Servidor Base de Datos (DB_SERVER): 192.168.10.12 / GW: 192.168.10.1
