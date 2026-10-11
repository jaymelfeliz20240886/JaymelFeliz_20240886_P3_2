# 🛡️ Laboratorio de Seguridad Perimetral y Segmentación con FortiGate

▶️ [**Video Demostrativo del Laboratorio**]([INSERTE_ENLACE_AQUI]) *(Demostración de conectividad, pruebas de RDP y acceso al portal RDWeb)*

## 📑 Índice

 1. [Propósito del Laboratorio](#1-propósito-del-laboratorio)
 2. [Topología de Red y Arquitectura](#2-topología-de-red-y-arquitectura)
 3. [Tabla de Direccionamiento IP](#3-tabla-de-direccionamiento-ip)
 4. [Procedimiento de Implementación Paso a Paso](#4-procedimiento-de-implementación-paso-a-paso)
 5. [Políticas de Seguridad (Firewall Policies)](#5-políticas-de-seguridad-firewall-policies)
 6. [Validación y Pruebas Analíticas](#6-validación-y-pruebas-analíticas)
 7. [Diagramas y Evidencias Visuales](#7-diagramas-y-evidencias-visuales)
 8. [Running Configurations Completas](#8-running-configurations-completas)
 9. [Scripts y Archivos Utilizados](#9-scripts-y-archivos-utilizados)
10. [Conclusión](#10-conclusión)

## 1. Propósito del Laboratorio

El objetivo principal de esta práctica es diseñar, configurar y asegurar una infraestructura de red corporativa simulada utilizando un firewall FortiGate. Esto incluye la segmentación estricta de tráfico entre una red local (LAN) con usuarios estructurados por privilegios, y una Zona Desmilitarizada (DMZ) que aloja servidores críticos (Jump Server y Web Server con IIS/RDWeb).

Se busca aplicar políticas de acceso basadas en roles, aislar a los usuarios sin privilegios (bloqueando SSH) y habilitar enrutamiento inter-VLAN segmentado implementando reglas NAT y segmentación con submáscaras `/29` para garantizar la inspección profunda de todo el tráfico entre nodos.

## 2. Topología de Red y Arquitectura

El entorno está desplegado en GNS3 y se divide en tres zonas lógicas principales, rigurosamente mediadas por el FortiGate:

* **Zona WAN/Management:** Conectada a la nube para acceso administrativo.
* **Zona LAN (Port 4):** Red de usuarios locales segmentada físicamente en el puerto 4. Aquí se encuentran:
  * `Webterm Privilegiado` (CONPRIV)
  * `Webterm Sin Privilegios` (SINPRIV)
* **Zona DMZ Segmentada (Ports 2 y 3):** Zona de servidores. Por diseño estricto de seguridad, los servidores están en puertos físicos distintos del FortiGate para forzar la inspección del tráfico inter-servidor:
  * `Jump Server` (Bastion Host) en el `port3`.
  * `Web Server` (IIS / RemoteApp) en el `port2`.

## 3. Tabla de Direccionamiento IP

| Dispositivo / Interfaz | Rol | Dirección IP | Máscara de Subred | Puerta de Enlace (Gateway) |
| :--- | :--- | :--- | :--- | :--- |
| **FortiGate - Port 4** | Gateway LAN | `10.24.86.1` | `255.255.255.128` (/25) | N/A |
| **FortiGate - Port 3** | Gateway Jump | `10.24.8.9` | `255.255.255.248` (/29) | N/A |
| **FortiGate - Port 2** | Gateway Web | `10.24.8.1` | `255.255.255.248` (/29) | N/A |
| **Webterm (CONPRIV)** | Cliente Priv. | `10.24.86.20` | `255.255.255.128` (/25) | `10.24.86.1` |
| **Jump Server** | Bastion Host | `10.24.8.10` | `255.255.255.248` (/29) | `10.24.8.9` |
| **Web Server** | RD Web Access | `10.24.8.2` | `255.255.255.248` (/29) | `10.24.8.1` |

## 4. Procedimiento de Implementación Paso a Paso

La implementación se llevó a cabo de manera estructurada, aplicando las configuraciones necesarias en Capa 2, 3 y 7 de forma preventiva para garantizar la conectividad y seguridad desde el primer despliegue:

### Fase 1: Acondicionamiento de la Topología y Segmentación (Capa 2 y 3)
* **Conectividad Física:** Se separó de forma estricta la LAN (conectando el switch de usuarios al `port4`) de la WAN y la DMZ, asegurando que todo el tráfico transite obligatoriamente por el FortiGate.
* **Subnetting Estratégico:** Se aplicaron máscaras `/29` (`255.255.255.248`) a los servidores en la DMZ. Esta técnica obliga al sistema operativo Windows a enviar el tráfico inter-servidor (del Jump Server al Web Server) hacia su Default Gateway en el FortiGate, permitiendo la inspección del tráfico que de otro modo viajaría en capa 2.

### Fase 2: Preparación de Servidores Windows (Capa 7)
Para garantizar la gestión centralizada de la seguridad a través del FortiGate, se adaptaron las políticas locales de Windows Server:
* **Firewall y Acceso:** Se desactivó Windows Defender Firewall y se deshabilitó la Autenticación a Nivel de Red (NLA) en las propiedades de Escritorio Remoto, delegando el control de acceso al FortiGate.
* **Configuración de IIS:** Se generó un **Certificado Autofirmado (Self-Signed)** en IIS y se vinculó explícitamente al puerto 443 (Site Bindings) para habilitar las conexiones HTTPS requeridas por el portal RD Web.

### Fase 3: Configuración de Políticas en FortiGate
* Se crearon objetos de red (`Addresses`) para cada IP y subred involucrada.
* Se configuró la política de acceso privilegiado definiendo múltiples interfaces de salida (`port2` y `port3`) para permitir el alcance simultáneo a ambos servidores.
* **Implementación de NAT:** Se habilitó NAT (*Use Outgoing Interface Address*) en la política de acceso LAN a DMZ. Esto enmascara la IP de origen del Webterm, garantizando que el Web Server procese la petición web de forma nativa sin bloqueos por salto de subred.

## 5. Políticas de Seguridad (Firewall Policies)

| ID | Nombre de Política | Interfaz Entrada | Interfaz Salida | Origen | Destino | Servicio | Acción | NAT |
| :-: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :-: |
| 1 | `Acceso_Privilegiado_Jump` | port4 | port3, port2 | Webterm_Priv (10.24.86.20) | JumpServer, WebServer | HTTPS, HTTP, RDP | ACCEPT | **ON** |
| 2 | `Jump_to_Web` | port3 | port2 | JumpServer (10.24.8.10) | WebServer (10.24.8.2) | HTTPS, HTTP, RDP | ACCEPT | OFF |
| 3 | `Deny_SSH_SinPrivilegios` | port4 | port3, port2 | Webterm_SinPriv | Any | SSH | DENY | OFF |
| 4 | `Implicit_Deny` | Any | Any | Any | Any | All | DENY | OFF |

## 6. Validación y Pruebas Analíticas

* **Diagnóstico de Rutas:** Ping exitoso desde el Webterm hacia su Gateway LAN (`10.24.86.1`).
* **Inspección de Tráfico (Policy Lookup):** Verificación exitosa en FortiGate para tráfico TCP/443 y TCP/3389 desde `10.24.86.20` hacia la DMZ, confirmando el *match* con la política ID 1.
* **Prueba RDP Inter-VLAN:** Conexión de Escritorio Remoto directa y exitosa desde el Jump Server hacia el Web Server.
* **Prueba de Servicio Web (RD Web):** Acceso satisfactorio al portal de aplicaciones utilizando la ruta completa `https://10.24.8.2/rdweb` tanto desde el Jump Server como desde el Webterm.

## 7. Diagramas y Evidencias Visuales

*Nota: Reemplazar estos textos por las capturas de pantalla de la práctica.*

* `[IMAGEN 1: Topología de GNS3 con el cableado correcto al port 4]`
* `[IMAGEN 2: Tabla de Políticas IPv4 en el GUI de FortiGate]`
* `[IMAGEN 3: Resultados de la herramienta Policy Lookup mostrando el tráfico permitido]`
* `[IMAGEN 4: Configuración IPv4 con submáscaras /29 en el entorno Windows]`
* `[IMAGEN 5: Asignación del certificado SSL en los Bindings de IIS]`
* `[IMAGEN 6: Acceso exitoso al Portal RD Web en el navegador del Webterm]`

## 8. Running Configurations Completas

**Extracto de Configuración de Interfaces y Políticas (FortiOS CLI):**

```text
config system interface
    edit "port2"
        set vdom "root"
        set ip 10.24.8.1 255.255.255.248
        set allowaccess ping https ssh
        set type physical
    next
    edit "port3"
        set vdom "root"
        set ip 10.24.8.9 255.255.255.248
        set allowaccess ping https ssh
        set type physical
    next
    edit "port4"
        set vdom "root"
        set ip 10.24.86.1 255.255.255.128
        set allowaccess ping
        set type physical
    next
end

config firewall policy
    edit 1
        set name "Acceso_Privilegiado_Jump"
        set srcintf "port4"
        set dstintf "port3" "port2"
        set srcaddr "Webterm_Privilegiado"
        set dstaddr "JumpServer_IP" "WebServer_IP"
        set action accept
        set schedule "always"
        set service "HTTPS" "HTTP" "RDP"
        set nat enable
    next
    edit 2
        set name "Jump_to_Web"
        set srcintf "port3"
        set dstintf "port2"
        set srcaddr "JumpServer_IP"
        set dstaddr "WebServer_IP"
        set action accept
        set schedule "always"
        set service "HTTPS" "HTTP" "RDP"
    next
    edit 3
        set name "Deny_SSH_SinPrivilegios"
        set srcintf "port4"
        set dstintf "port3" "port2"
        set srcaddr "Webterm_SinPriv"
        set dstaddr "all"
        set action deny
        set schedule "always"
        set service "SSH"
    next
end
```

## 9. Scripts y Archivos Utilizados

**Comandos de inicialización en Webterm (Terminal Linux):**

```bash
# Revisar direccionamiento IP asignado
ip addr show

# Asignar la ruta por defecto hacia el FortiGate (Gateway LAN)
ip route add default via 10.24.86.1

# Verificación de alcance al Gateway
ping 10.24.86.1
```

**Comandos operativos en Windows Servers (CMD):**

```cmd
:: Verificación de parámetros IPv4 y submáscaras /29
ipconfig /all

:: Reinicio del motor web para la aplicación correcta del certificado SSL
iisreset
```

## 10. Conclusión

El laboratorio fue ejecutado con éxito, logrando una implementación robusta que demuestra las mejores prácticas en segmentación de redes y seguridad perimetral. Al aplicar un diseño premeditado con submáscaras `/29`, se garantizó que todo el tráfico inter-servidor cruzara el FortiGate para su debida inspección. 

La integración adecuada de NAT en el firewall, combinada con la preparación correcta de los servicios en Windows (IIS y Firewall local), permitió establecer comunicaciones fluidas y seguras, cumpliendo a cabalidad con los requerimientos de acceso estructurado para los distintos roles de usuario definidos en la rúbrica del proyecto.
