# Ataque DHCP Starvation — Documentación Técnica

Autor: Junior Javier Santos Perez

Matrícula: 2024-1599

Herramienta: dhcp_starvation.py

Plataforma de laboratorio: GNS3 + Kali Linux 2025.3

Repositorio GitHub:

https://github.com/juniorjaviersantosperez/DHCP-Starvation 

Video demostrativo:
https://www.youtube.com/watch?v=jfs6yb-D4jQ 


---

## Tabla de Contenidos

1. [Objetivo del Laboratorio](#objetivo-del-laboratorio)
2. [Objetivo del Script](#objetivo-del-script)
3. [Parámetros Utilizados](#parámetros-utilizados)
4. [Requisitos para el Uso de la Herramienta](#requisitos-para-el-uso-de-la-herramienta)
5. [Descripción del Funcionamiento del Script](#descripción-del-funcionamiento-del-script)
6. [Documentación de la Red](#documentación-de-la-red)
7. [Topología](#topología)
8. [Capturas de Pantalla](#capturas-de-pantalla)
9. [Medidas de Mitigación / Contramedidas](#medidas-de-mitigación--contramedidas)

---

## Objetivo del Laboratorio

Demostrar de forma práctica y controlada la ejecución de un ataque de **DHCP Starvation (Agotamiento de Pool DHCP)** sobre una red simulada en GNS3, con el fin de:

- Comprender cómo un atacante puede agotar el pool de direcciones IP de un servidor DHCP legítimo.
- Observar cómo los clientes legítimos quedan sin posibilidad de obtener una dirección IP.
- Verificar el agotamiento del pool mediante el error de configuración en la víctima.
- Proponer e implementar contramedidas efectivas para mitigar el ataque.

---

## Objetivo del Script

El script `dhcp_starvation.py` tiene como objetivo ejecutar un ataque de **Denegación de Servicio (DoS) a nivel DHCP** agotando completamente el pool de direcciones IP disponibles en el servidor DHCP legítimo (R1), con el fin de:

- Enviar masivamente solicitudes **DHCP DISCOVER** con **MACs de origen falsas y aleatorias** para que el servidor asigne una IP a cada solicitud.
- Consumir todas las IPs disponibles en el rango del pool DHCP (`10.0.99.0/24`).
- Impedir que cualquier cliente legítimo obtenga configuración de red mediante DHCP.
- Causar una denegación de servicio efectiva sin necesidad de interrumpir físicamente la red.

---

## Parámetros Utilizados

| Parámetro     | Valor            | Descripción                                                        |
|---------------|------------------|--------------------------------------------------------------------|
| Interfaz      | `eth0`           | Interfaz del atacante usada para enviar solicitudes DHCP           |
| Paquetes      | `1,000`          | Total de solicitudes DHCP DISCOVER enviadas con MACs falsas        |
| Red objetivo  | `10.0.99.0/24`   | Subred del pool DHCP del servidor legítimo                         |
| Pool DHCP     | `10.0.99.0/24`   | Red configurada en R1 con default-router `10.0.99.1`, DNS `8.8.8.8`|
| MACs falsas   | Aleatorias       | Cada solicitud usa una MAC diferente para simular un host distinto  |

---

## Requisitos para el Uso de la Herramienta

### Sistema Operativo
- Kali Linux 2025.3 (o cualquier distribución Linux con soporte a raw sockets)

### Privilegios
- Ejecución como `root` o con `sudo` (necesario para enviar paquetes DHCP crudos)

### Dependencias Python
```bash
# Requiere Scapy para construcción y envío de paquetes DHCP DISCOVER
pip install scapy

# Módulos de la librería estándar utilizados:
import random
import time
import threading
```

### Configuración Previa
```bash
# Reconectar la interfaz antes de ejecutar el ataque
# para asegurar conectividad al segmento DHCP
sudo nmcli device disconnect eth0
sudo nmcli device connect eth0
```

### Entorno de Laboratorio
- GNS3 con router Cisco R1 configurado como servidor DHCP
- Switch Cisco IOSv Layer 2 (Swich-1)
- Kali Linux como nodo atacante y Clonekali-1 como víctima
- Solar-PuTTY para monitoreo del router y switch

---

## 🔬 Descripción del Funcionamiento del Script

El script opera en las siguientes fases:

### Fase 1 — Preparación de la Interfaz
Antes de ejecutar el ataque se reconecta la interfaz `eth0` para garantizar acceso al segmento de red DHCP:
```bash
sudo nmcli device disconnect eth0
sudo nmcli device connect eth0
```

### Fase 2 — Generación de Solicitudes DHCP DISCOVER
El script genera **1,000 paquetes DHCP DISCOVER**, cada uno con una **dirección MAC de origen completamente aleatoria** de 6 bytes:
```python
mac_falsa = ':'.join(['{:02x}'.format(random.randint(0, 255)) for _ in range(6)])
```
Cada MAC única hace creer al servidor DHCP que se trata de un dispositivo diferente solicitando una IP.

### Fase 3 — Envío Masivo
El script envía los 1,000 paquetes DHCP DISCOVER al broadcast `255.255.255.255`, consumiendo progresivamente todas las IPs disponibles en el pool del servidor. El progreso se muestra en pantalla:
```
[+] Paquete   1/1000 | MAC: e7:17:bc:b5:0c:6f
[+] Paquete   2/1000 | MAC: d1:39:0b:d0:5f:eb
...
[+] Paquete 1000/1000 | MAC: ...
```

### Fase 4 — Agotamiento del Pool
Una vez consumidas todas las IPs del pool, el servidor DHCP ya no puede responder a nuevas solicitudes legítimas. La evidencia se verifica intentando obtener IP desde la víctima:
```bash
# En la víctima (Clonekali-1)
sudo nmcli device disconnect eth0
sudo nmcli device connect eth0
# Resultado:
Error: Connection activation failed: IP configuration could not be
reserved (no available address, timeout, etc.).
```

### Fase 5 — Denegación de Servicio Confirmada
La víctima queda sin dirección IP, sin gateway y sin DNS — efectivamente aislada de la red, sin que el atacante haya interrumpido ningún cable ni dispositivo físico.

---

## 🌐 Documentación de la Red

### Tabla de Direccionamiento IP

| Nodo                              | Rol             | Interfaz | Dirección IP    | Notas                                    |
|-----------------------------------|-----------------|----------|-----------------|------------------------------------------|
| kali-linux-2025.3-vmware-amd64-1  | Atacante        | eth0/e1  | 10.0.99.100     | Envía 1,000 DHCP DISCOVERs con MACs falsas|
| Clonekali-1                       | Víctima         | e0/e2    | Sin IP (DoS)    | No puede obtener IP — pool agotado       |
| R1                                | Router/DHCP     | f0/0     | 10.0.99.1       | Servidor DHCP legítimo — pool agotado    |
| Swich-1                           | Switch L2       | e0/e1/e2 | N/A (L2)        | Cisco IOS IOSv Switch                    |

### Configuración del Servidor DHCP Legítimo (R1)
```cisco
ip dhcp pool ITLA
 network 10.0.99.0 255.255.255.0
 default-router 10.0.99.1
 dns-server 8.8.8.8
```

### Protocolo Explotado

| Protocolo | Puerto    | Descripción                                                                   |
|-----------|-----------|-------------------------------------------------------------------------------|
| DHCP      | UDP 67/68 | Sin límite de solicitudes por MAC — pool consumible por cualquier cliente     |

### Evidencia del Agotamiento

| Estado      | Resultado en víctima                                                          |
|-------------|-------------------------------------------------------------------------------|
| Antes        | IP obtenida correctamente desde el pool `10.0.99.0/24`                       |
| Durante/Después | `Error: IP configuration could not be reserved (no available address)`   |

---

## 🗺️ Topología

La topología fue diseñada e implementada en **GNS3** con los siguientes componentes:

```
    [Clonekali-1 / Víctima]
    Sin IP — Pool agotado
          |
          | (e2 - Swich-1)
     [Swich-1]────────────────[R1 / Servidor DHCP]
          |  (e0)                  f0/0 — 10.0.99.1
          |                        Pool: 10.0.99.0/24
     (e1 - Swich-1)
          |
    [kali-linux-2025.3 / Atacante]
    10.0.99.100
    1,000 DHCP DISCOVERs con MACs falsas
```

> 📁 La imagen de la topología se encuentra en la carpeta `/images/` del repositorio.

---

## 📸 Capturas de Pantalla

Las capturas de pantalla se encuentran almacenadas en la carpeta **`/images/`** del repositorio.

| # | Archivo | Descripción |
|---|---------|-------------|
| 1 | `imagen_01_topologia.png` | Topología del laboratorio en GNS3 con nombre y matrícula del estudiante |
| 2 | `imagen_02_dhcp_pool_r1.png` | Configuración del pool DHCP en R1: red `10.0.99.0/24`, gateway `10.0.99.1`, DNS `8.8.8.8` |
| 3 | `imagen_03_reconexion_interfaz_victima.png` | Reconexión de interfaz `eth0` en la víctima previa al ataque — conexión exitosa |
| 4 | `imagen_04_ataque_iniciado.png` | Script `dhcp_starvation.py` en ejecución — 1,000 paquetes con MACs aleatorias enviándose |
| 5 | `imagen_05_pool_agotado_victima.png` | Víctima intentando obtener IP — error `no available address` — pool completamente agotado |
| 6 | `imagen_06_contramedida_port_security.png` | Contramedida aplicada en el switch: `port-security maximum 2`, `violation shutdown` en g0/2 |

---

## 🛡️ Medidas de Mitigación / Contramedidas

### 1. Port Security en el Switch (Contramedida Principal)
```cisco
Switch# configure terminal
Switch(config)# interface g0/2
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 2
Switch(config-if)# switchport port-security violation shutdown
Switch(config-if)# ^Z
Switch# write memory
```
> Limita el número de MACs por puerto. Al superar el límite, el puerto se deshabilita (`err-disabled`), bloqueando el flujo de solicitudes DHCP con MACs falsas. **Es la contramedida más directa contra DHCP Starvation.**

### 2. DHCP Snooping con Rate Limiting
```cisco
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1
Switch(config)# interface g0/2
Switch(config-if)# ip dhcp snooping limit rate 15
```
> Limita el número de paquetes DHCP por segundo en puertos de acceso. Solicitudes que excedan la tasa configurada son descartadas automáticamente.

### 3. Limitar el Tiempo de Lease en el Router
```cisco
R1(config)# ip dhcp pool ITLA
R1(dhcp-config)# lease 0 1
```
> Reducir el tiempo de arrendamiento hace que las IPs "robadas" expiren más rápido, liberando el pool con mayor frecuencia.

### 4. Exclusión de IPs Críticas
```cisco
R1(config)# ip dhcp excluded-address 10.0.99.1 10.0.99.10
```
> Excluir un rango de IPs garantiza que dispositivos críticos (routers, servidores, cámaras) siempre tengan IPs disponibles incluso durante un ataque.

### 5. Monitoreo y Detección
```cisco
! Verificar IPs asignadas y agotamiento del pool
R1# show ip dhcp binding
R1# show ip dhcp pool
R1# show ip dhcp conflict

! Verificar estado de Port Security en el switch
Switch# show port-security interface g0/2
Switch# show port-security address
```

---

## ⚠️ Aviso Legal / Disclaimer

> Este laboratorio fue realizado en un entorno **completamente controlado y simulado** con fines académicos y de investigación en seguridad informática. El uso de estas técnicas fuera de entornos autorizados es ilegal y contrario a la ética profesional. El autor no se hace responsable del uso indebido de este material.

