# Implementación de Infraestructura de Monitoreo Centralizado y Gestión Dinámica (Cisco IOS & Zabbix Server)

**Autor:** Manuel Alejandro Cruz Messón  
**Matrícula:** 2025-0689  
**Asignatura:** Seguridad de Redes  
**Enlace del Video Demostrativo:** https://youtu.be/[ID_DE_TU_VIDEO]  

---

## 1. Propósito del Laboratorio

El propósito fundamental de este laboratorio es diseñar, desplegar y auditar una plataforma de supervisión centralizada y gestión dinámica para dispositivos de red perimetrales y de conmutación (Router y Switch Cisco IOS), empleando el protocolo de gestión de red **SNMPv2c (Simple Network Management Protocol)** bajo directivas estrictas de seguridad de solo lectura, e integrándolo con un sistema de gestión de red (NMS) **Zabbix Appliance**.

A través de esta implementación se persiguen los siguientes fines específicos de seguridad y administración de redes:

* **Visibilidad Operativa y Detección Temprana de Anomalías:** Establecer telemetría constante sobre el estado de salud del hardware (tiempo de actividad o *uptime*, consumo de recursos) y el rendimiento de los enlaces de telecomunicación (tasa de transferencia, paquetes descartados, errores de interfaz), permitiendo mitigar proactivamente fallos de servicio y detectar posibles saturaciones o ataques de denegación de servicio (DoS).
* **Aplicación del Principio de Mínimo Privilegio (PoLP) en la Gestión SNMP:** Erradicar el uso de cadenas de comunidad universales por defecto (tales como `public` o `private`) e implementar comunidades descriptivas restringidas de manera inequívoca al modo de solo lectura (`ro`). Esto previene ataques de inyección y reconfiguración no autorizada de OIDs en la MIB del dispositivo que pudieran comprometer la topología o habilitar túneles maliciosos.
* **Automatización y Aprovisionamiento Dinámico de Red (DHCP):** Centralizar la entrega de parámetros IP en la red LAN a través del router de borde, implementando rangos de exclusión controlados para proteger las direcciones estáticas de la infraestructura crítica (gateways, switches y servidores) y facilitando el despliegue desatendido de sondas y clientes de monitoreo.
* **Aislamiento Perimetral y Salida Controlada a Redes Externas:** Proveer conectividad hacia el exterior (Internet/Management Cloud) mediante mecanismos de Traducción de Direcciones de Red con Sobrecarga (NAT/PAT) y redirección estática de puertos (Port Forwarding), manteniendo la topología interna oculta y accesible únicamente a través de puntos de entrada designados.
* **Trazabilidad y Normalización de Activos de Red:** Establecer registros descriptivos en cada dispositivo mediante objetos MIB estándar (`sysName`, `sysContact`, `sysLocation`), garantizando que la plataforma de monitoreo clasifique y geolocalice los incidentes de red de manera fehaciente.

---

## 2. Resumen y Alcance del Laboratorio

Este laboratorio integra una solución integral de supervisión de infraestructura y aprovisionamiento de red corporativa para dispositivos Cisco IOS, conectando un Router de borde, un Switch de acceso y un Servidor NMS Zabbix dentro del emulador PNETLab.

### Objetivos Clave Ejecutados:

1. **Segmentación y Direccionamiento IP Basado en Matrícula:** Cálculo e implementación del direccionamiento de red clase privada `10.25.68.0/24` derivado de los dígitos de la matrícula del estudiante (`2025-0689`).
2. **Despliegue del Servicio DHCP en Cisco IOS:** Configuración de un pool de asignación dinámica en el router perimetral con exclusión de las primeras 49 direcciones IP para preservar la infraestructura estática.
3. **Endurecimiento de Protocolo SNMPv2c:** Habilitación del agente SNMP en Router y Switch utilizando la comunidad personalizada `SNMP_ITLA_RO` en modo `read-only`, evitando cualquier alteración remota de la base de datos de administración (MIB).
4. **Traducción de Direcciones y Gestión Externa (Cloud0 & NAT):** Configuración de PAT en la interfaz WAN del router para dar conectividad de actualización al servidor Zabbix y redirección estática de puerto TCP 80 para acceder al frontend gráfico desde la estación de trabajo física.
5. **Aprovisionamiento y Operación de Zabbix Appliance:** Puesta a punto del host Zabbix como cliente DHCP en consola Linux, importación y asociación de plantillas de supervisión (`Template SNMP Device`), vinculación de macros comunitarias y verificación del polleo de métricas en tiempo real.

---

## 3. Topología de Red y Plan de Direccionamiento

La infraestructura se compone de un segmento WAN conectado a la red de gestión física (Cloud0) y un segmento LAN corporativo estructurado bajo el prefijo `10.25.68.0/24`.

[AQUI VA FOTO DE: Topología de red completa en PNETLab mostrando Cloud0, R1-Core, SW1-LAN y Zabbix Server]

### Matriz de Conectividad e Interfaces

| Dispositivo | Rol de Red | Interfaz | Dirección IP / Prefijo | Gateway Predeterminado | Tipo de Asignación |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R1-Core** (vIOS) | Router de Borde / Gateway LAN / Servidor DHCP | `GigabitEthernet0/0` | Dinámica (Rango Cloud0) | Proveedor WAN | DHCP (Outside) |
| **R1-Core** (vIOS) | Router de Borde / Gateway LAN / Servidor DHCP | `GigabitEthernet0/1` | `10.25.68.1/24` | N/A | Estática (Inside) |
| **SW1-LAN** (IOSvL2) | Switch de Acceso L2 / SVI de Administración | `Vlan 1` | `10.25.68.2/24` | `10.25.68.1` | Estática |
| **Zabbix-Server** (Linux QEMU) | Servidor NMS / Sonda de Monitoreo Centralizada | `eth0` | `10.25.68.50/24` | `10.25.68.1` | DHCP (Cliente) |

---

## 4. Matriz de Configuración de Servicios y Protocolos

### Parámetros del Agente SNMPv2c
* **Versión del Protocolo:** SNMPv2c
* **Cadena de Comunidad (Community String):** `SNMP_ITLA_RO`
* **Permisos:** Solo Lectura (`ro`)
* **Puerto de Transporte:** UDP 161
* **OIDs Principales Auditados:**
  * `sysDescr` (`1.3.6.1.2.1.1.1.0`): Descripción de versión de Cisco IOS.
  * `sysUpTime` (`1.3.6.1.2.1.1.3.0`): Tiempo de operación continua del sistema.
  * `sysContact` (`1.3.6.1.2.1.1.4.0`): Administrador responsable (`Admin_Redes`).
  * `sysName` (`1.3.6.1.2.1.1.5.0`): Hostname del equipo de red.
  * `sysLocation` (`1.3.6.1.2.1.1.6.0`): Ubicación física asignada al nodo.
  * `ifTable` / `ifXTable` (`1.3.6.1.2.1.2.2` / `1.3.6.1.2.1.31`): Estado operacional, octetos entrantes y salientes.

### Parámetros del Servicio DHCP (R1-Core)
* **Nombre del Pool:** `LAN_ZABBIX`
* **Red Servida:** `10.25.68.0 255.255.255.0`
* **Rango Excluido:** `10.25.68.1` al `10.25.68.49`
* **Rango Útil Dinámico:** `10.25.68.50` al `10.25.68.254`
* **Default Router:** `10.25.68.1`
* **Servidores DNS:** `8.8.8.8`, `1.1.1.1`

---

## 5. Implementación en Dispositivos Cisco IOS

### 5.1. Configuración de R1-Core
Se estructuró el enrutador con doble contexto de interfaces (WAN hacia Cloud0 y LAN hacia el switch local), aplicando reglas de NAT para la salida externa y levantando el proceso SNMP:

```cisco
hostname R1-Core

interface GigabitEthernet0/0
 description Conexión WAN - Management Cloud0
 ip address dhcp
 ip nat outside
 no shutdown
exit

interface GigabitEthernet0/1
 description Conexión LAN - Switch Local
 ip address 10.25.68.1 255.255.255.0
 ip nat inside
 no shutdown
exit

ip dhcp excluded-address 10.25.68.1 10.25.68.49
ip dhcp pool LAN_ZABBIX
 network 10.25.68.0 255.255.255.0
 default-router 10.25.68.1
 dns-server 8.8.8.8 1.1.1.1
exit

access-list 1 permit 10.25.68.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
ip nat inside source static tcp 10.25.68.50 80 interface GigabitEthernet0/0 80

snmp-server community SNMP_ITLA_RO ro
snmp-server location Laboratorio_ITLA
snmp-server contact Admin_Redes
```

### 5.2. Configuración de SW1-LAN
El conmutador de acceso se configuró con una interfaz virtual de conmutación (SVI) en la VLAN 1 para su gestión en Capa 3, gateway por defecto y los parámetros del agente SNMP:

```cisco
hostname SW1-LAN

interface vlan 1
 description Interfaz de Gestion LAN
 ip address 10.25.68.2 255.255.255.0
 no shutdown
exit

ip default-gateway 10.25.68.1

interface GigabitEthernet0/0
 description Enlace Troncal/Acceso a R1 Gi0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
exit

interface GigabitEthernet0/1
 description Enlace Acceso a Zabbix Server
 switchport mode access
 switchport access vlan 1
 no shutdown
exit

snmp-server community SNMP_ITLA_RO ro
snmp-server location Switch_Piso1
snmp-server contact Admin_Redes
```

---

## 6. Implementación y Configuración de Zabbix Server

### 6.1. Aprovisionamiento del Sistema Operativo Base
El servidor Zabbix (desplegado sobre imagen Linux Appliance) se integró a la topología configurando su interfaz `eth0` como cliente dinámico para recibir los parámetros emitidos por el router Cisco.

[AQUI VA FOTO DE: Consola de Zabbix Server ejecutando dhclient o reiniciando la interfaz de red]

### 6.2. Registro y Enlace de Plantillas en Zabbix Frontend
Dentro del entorno web (`http://<IP_ACCESO>/zabbix`):
1. **Creación de Hosts:** Se definieron los nodos de red correspondientes al router (`10.25.68.1:161`) y switch (`10.25.68.2:161`) utilizando interfaces exclusivas de tipo SNMP.
2. **Asociación de Plantillas:** Se vincularon los modelos predeterminados `Template SNMP Device` (conteniendo `Template SNMP Generic` y `Template SNMP Interfaces`), permitiendo la autogeneración de reglas de descubrimiento de interfaces de red y métricas globales.
3. **Inyección de Macros de Autenticación:** Se parametrizó la macro global/de host `{$SNMP_COMMUNITY}` con el valor unificado `SNMP_ITLA_RO`, permitiendo a los procesos poller autenticarse de forma transparente.

---

## 7. Pruebas y Evidencias de Validación

### 7.1. Verificación de Asignación Dinámica de Direcciones IP (DHCP Client)
* **Objetivo:** Comprobar que el servidor Zabbix adquiere automáticamente una dirección IP perteneciente al bloque dinámico configurado en Cisco IOS (`10.25.68.50/24`).
* **Comandos probados en consola Linux:** `ip a`, `ip route show`.

[AQUI VA FOTO DE: Salida de ip a en la terminal de Zabbix mostrando eth0 con la IP 10.25.68.50/24 y el gateway 10.25.68.1]

* **Resultado:** La interfaz de red `eth0` tomó de manera correcta la primera IP disponible del pool (`10.25.68.50`) con máscara de subred `/24`, gateway predeterminado hacia `10.25.68.1` y servidor DNS asignado.

### 7.2. Validación de Conectividad y Consulta SNMPv2 desde Consola
* **Objetivo:** Validar el canal bidireccional UDP 161 y confirmar que el agente Cisco responde favorablemente a consultas de objetos MIB con la comunidad configurada.
* **Comandos probados:** `ping -c 3 10.25.68.1`, `snmpwalk -v2c -c SNMP_ITLA_RO 10.25.68.1 sysName`.

[AQUI VA FOTO DE: Consola de Zabbix ejecutando snmpwalk hacia el Router y Switch obteniendo el nombre del sistema]

* **Resultado:** La solicitud SNMPv2 respondió satisfactoriamente devolviendo la variable `sysName.0 = STRING: R1-Core.localdomain` y `sysName.0 = STRING: SW1-LAN` sin registrar timeouts ni paquetes descartados.

### 7.3. Verificación de Disponibilidad SNMP de Hosts en Zabbix Frontend
* **Objetivo:** Demostrar que los dispositivos de infraestructura son reconocidos como disponibles y operativos bajo el protocolo SNMP en la interfaz web de monitoreo.
* **Ruta de verificación:** **Configuration** > **Hosts**.

[AQUI VA FOTO DE: Vista de Configuration -> Hosts mostrando Router R1 y SW1-LAN con el icono SNMP en verde]

* **Resultado:** En la columna *Availability*, ambos dispositivos presentan el icono **SNMP en verde brillante**, certificando que los ciclos periódicos de recolección se completan exitosamente hacia las IPs `10.25.68.1` y `10.25.68.2`.

### 7.4. Auditoría de Recolección de Métricas de Sistema e Interfaces (Latest Data)
* **Objetivo:** Validar la extracción continua de variables de rendimiento, estado físico de interfaces y metadatos de configuración en ambos dispositivos.
* **Ruta de verificación:** **Monitoring** > **Latest data** (Filtro: `Router R1`, `SW1-LAN`).

[AQUI VA FOTO DE: Pantalla de Latest Data en Zabbix desplegando Device contact, location, name, uptime y el estado operacional up de las interfaces]

* **Resultado:** El sistema recopiló correctamente:
  * **Device contact details:** `Admin_Redes`
  * **Device location:** `Laboratorio_ITLA` (Router) y `Switch_Piso1` (Switch)
  * **Device uptime:** Registro de tiempo operativo continuo (> 25 minutos).
  * **Admin status of interface:** Interfaces físicas activas (`GigabitEthernet0/0` y `0/1` en estado `up (1)`).

### 7.5. Análisis y Monitoreo de Tráfico de Red en Tiempo Real (Graphs)
* **Objetivo:** Demostrar la graficación histórica y en tiempo real del ancho de banda cursado por las interfaces de red de los equipos de telecomunicaciones.
* **Ruta de verificación:** **Monitoring** > **Graphs**.

#### A. Tráfico en Switch SW1-LAN (`GigabitEthernet0/0` - Enlace Router)
[AQUI VA FOTO DE: Gráfica de Zabbix mostrando el tráfico entrante y saliente en SW1-LAN interfaz Gi0/0 con picos de transferencia]

#### B. Tráfico en Switch SW1-LAN (`GigabitEthernet0/1` - Enlace Servidor Zabbix)
[AQUI VA FOTO DE: Gráfica de Zabbix mostrando el tráfico en SW1-LAN interfaz Gi0/1 correspondiente a la interacción web y polleo SNMP]

#### C. Tráfico en Router R1-Core (`GigabitEthernet0/1` - Enlace LAN)
[AQUI VA FOTO DE: Gráfica de Zabbix mostrando el comportamiento del tráfico de la red interna en R1 Gi0/1]

#### D. Tráfico en Router R1-Core (`GigabitEthernet0/0` - Enlace WAN Cloud0)
[AQUI VA FOTO DE: Gráfica de Zabbix reflejando el consumo de ancho de banda hacia la interfaz WAN Gi0/0]

* **Resultado:** Las cuatro gráficas exponen con nitidez la actividad de la red, registrando picos de tráfico de hasta 2 Mbps durante las transferencias iniciales de datos y manteniendo un suelo constante de telemetría SNMP.

---

## 8. Conclusiones

1. **Eficiencia y Seguridad en el Monitoreo Proactivo:** La adopción de SNMPv2 de solo lectura combinada con comunidades personalizadas robustece la visibilidad operativa de la organización sin exponer a los dispositivos a manipulaciones indebidas de configuración a través del plano de gestión.
2. **Resiliencia en el Control de Direccionamiento:** La delegación del servicio DHCP en el router perimetral con exclusión explícita de segmentos estáticos permite una administración ordenada del espacio de direccionamiento, asegurando que las sondas de monitoreo y estaciones operativas se desplieguen sin colisiones IP con los gateways y switches de la red.
3. **Telemetría y Capacidad de Planificación:** La recolección periódica y graficación en Zabbix proporciona a los administradores de seguridad y operaciones métricas cuantitativas sobre la utilización de ancho de banda y disponibilidad de enlaces, facilitando la detección de cuellos de botella, la investigación forense de fallos y el dimensionamiento oportuno de capacidad.

---

## Declaración de Uso de Inteligencia Artificial

* **Herramientas utilizadas:** Modelos de lenguaje e inteligencia artificial generativa asistieron en la consulta de sintaxis, estructuración de archivos de configuración y formateo de la documentación técnica.
* **Alcance del uso:** La herramienta sirvió de soporte técnico para la redacción y resolución de incompatibilidades de entorno durante el despliegue del appliance de monitoreo.
* **Autoría y validación:** El diseño topológico, la implementación en PNETLab, el direccionamiento calculado en base a la matrícula, la configuración de Cisco IOS (DHCP, NAT y SNMP) y las validaciones de telemetría en Zabbix fueron configuradas, ejecutadas y comprobadas íntegramente por el autor del proyecto.
