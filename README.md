# 🔐 FortiGate — Práctica GUI de FortiGate

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![FortiGate](https://img.shields.io/badge/Plataforma-FortiGate-red?style=for-the-badge&logo=fortinet)
![Platform](https://img.shields.io/badge/Entorno-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📋 Contenido

1. [Objetivo](#1-objetivo)
2. [Topología y direccionamiento](#2-topología-y-direccionamiento)
3. [Configuración paso a paso (GUI)](#3-configuración-paso-a-paso-gui)
   - [3.0 Habilitación inicial de la GUI (CLI en la WAN)](#30-habilitación-inicial-de-la-gui-cli-en-la-wan)
   - [3.1 Interfaces](#31-interfaces)
   - [3.2 DHCP para el segmento de usuarios](#32-dhcp-para-el-segmento-de-usuarios)
   - [3.3 Ruta hacia Internet](#33-ruta-hacia-internet)
   - [3.4 Salida a Internet con NAT](#34-salida-a-internet-con-nat)
   - [3.5 Restricción de tráfico hacia LAN-SERVIDORES (solo HTTP)](#35-restricción-de-tráfico-hacia-lan-servidores-solo-http)
   - [3.6 Application Control — redes sociales](#36-application-control--redes-sociales)
   - [3.7 Application Control — llamadas de WhatsApp](#37-application-control--llamadas-de-whatsapp)
   - [3.8 DNS/Web Filter — dominio institucional](#38-dnsweb-filter--dominio-institucional)
   - [3.9 DoS Policy — anti escaneo de puertos](#39-dos-policy--anti-escaneo-de-puertos)
   - [3.10 WAF para el servicio expuesto en LAN-SERVIDORES](#310-waf-para-el-servicio-expuesto-en-lan-servidores)
4. [Evidencias](#4-evidencias)
5. [Video](#5-video)
6. [Referencias](#6-referencias)

---

## 1. Objetivo

Endurecer un FortiGate configurado íntegramente desde la **GUI** (salvo el arranque inicial, obligatoriamente por CLI) para segmentar una red de laboratorio compuesta por un host que actúa como origen de pruebas (**LAN-USERS**) y un servidor en **LAN-SERVIDORES** que expone un servicio web.

Metas puntuales:

* Levantar acceso administrativo desde el host físico configurando primero la interfaz WAN por consola.
* Segmentar el tráfico entre LAN-USERS y LAN-SERVIDORES, dejando pasar únicamente HTTP entre ambos.
* Salida a Internet con NAT y asignación dinámica de direcciones para el segmento de usuarios.
* Filtrar capa de aplicación: bloquear redes sociales y las llamadas de voz/video de WhatsApp.
* Bloquear el acceso a un dominio institucional específico vía DNS/Web Filter.
* Identificar y frenar escaneos de puertos (TCP/UDP) contra la red mediante **DoS Policy**, ya que la licencia del equipo no trae el catálogo completo de firmas IPS para cubrir este tipo de reconocimiento.
* Proteger el servicio web publicado por LAN-SERVIDORES con un perfil **WAF**.

---

## 2. Topología y direccionamiento

```
                         [ Net ]
                            │
                          port2 (WAN)
                            │
                      ┌─────┴─────┐
                      │ Fortinet  │
                      │           │
             port3 ───┤           ├─── port1
                      └───────────┘
                  │                     │
                  e0                  eth1
                  │                     │
             ┌────┴─────┐        ┌──────┴──────┐
             │  usuario │        │   servidor  │
             │(LAN-USERS)│       │(LAN-SERVIDORES)│
             └──────────┘        └─────────────┘
```

**Interfaces del firewall:**

| Interfaz | Segmento | Conecta con | Se configura por |
|---|---|---|---|
| port2 | WAN | Nube *Net* | CLI (arranque) + GUI |
| port3 | LAN-USERS | Host usuario (e0) | GUI |
| port1 | LAN-SERVIDORES | Servidor (eth1) | GUI |

**Plan de direccionamiento** *(ajustar la IP de port2 si tu nube de PNETLab usa otro rango de salida)*:

| Interfaz | Red | IP asignada al Fortinet | Rango disponible para hosts |
|---|---|---|---|
| port2 (WAN) | 192.168.19.0/24 | 192.168.19.10/24 | .2 – .254 (uplink hacia *Net*) |
| port3 (LAN-USERS) | 202.50.73.0/25 | 202.50.73.1/25 | .2 – .126 (DHCP) |
| port1 (LAN-SERVIDORES) | 202.50.73.128/25 | 202.50.73.129/25 | .130 – .254 (IP fija) |

| Equipo | Interfaz | IP | Asignación | Rol |
|---|---|---|---|---|
| usuario | e0 | dentro de 202.50.73.2–.126 | DHCP | Origen de pruebas: navegación, escaneo de puertos |
| servidor | eth1 | 202.50.73.130 (fija) | Estática | Publica un servicio web a proteger con WAF |

---

## 3. Configuración paso a paso (GUI)

> Salvo el punto 3.0, toda la configuración se hace desde la interfaz web del FortiGate. Cada apartado indica el menú exacto usado y trae el enlace a la captura correspondiente dentro de [`evidencias/`](evidencias/).

---

### 3.0 Habilitación inicial de la GUI (CLI en la WAN)

Recién desplegado, el Fortinet no tiene forma de recibir conexiones HTTPS/HTTP porque ninguna interfaz tiene IP. Para poder entrar a la GUI desde mi host, entro por la consola (terminal de PNETLab) y le doy dirección y acceso administrativo a **port2**, que es la interfaz por la que me voy a conectar:

```bash
config system interface
    edit "port2"
        set mode static
        set ip 192.168.19.10 255.255.255.0
        set allowaccess ping https http ssh
        set role wan
    next
end
```

Confirmo que quedó aplicado:

```bash
get system interface physical
```

A partir de aquí abro `https://192.168.19.10` desde el navegador y continúo todo lo demás sin volver a tocar la CLI.

📸 [`00-cli-port2.png`](evidencias/00-cli-port2.png)

---

### 3.1 Interfaces

**Menú:** `Network > Interfaces`

Doy de alta port3 y port1 con la IP fija que le corresponde a cada una según la tabla de direccionamiento (sección 2):

**port3 (LAN-USERS)**

| Campo | Valor |
|---|---|
| Alias | LAN-USERS |
| Role | LAN |
| Modo | Manual |
| IP/Máscara | 202.50.73.1/25 |
| Acceso administrativo | PING, HTTPS |

**port1 (LAN-SERVIDORES)**

| Campo | Valor |
|---|---|
| Alias | LAN-SERVIDORES |
| Role | LAN |
| Modo | Manual |
| IP/Máscara | 202.50.73.129/25 |
| Acceso administrativo | PING |

📸 [`01-interface-port3.png`](evidencias/01-interface-port3.png)
📸 [`02-interface-port1.png`](evidencias/02-interface-port1.png)

---

### 3.2 DHCP para el segmento de usuarios

**Menú:** `Network > Interfaces > port3 > Edit > DHCP Server > Create New`

| Campo | Valor |
|---|---|
| Estado | Enable |
| Rango de direcciones | 202.50.73.2 – 202.50.73.126 |
| Máscara | 255.255.255.128 |
| Gateway entregado | 202.50.73.1 |
| DNS entregado | 8.8.8.8 / 8.8.4.4 |
| Tiempo de lease | 1 día |

📸 [`03-dhcp-usuarios.png`](evidencias/03-dhcp-usuarios.png)

---

### 3.3 Ruta hacia Internet

**Menú:** `Network > Static Routes > Create New`

| Campo | Valor |
|---|---|
| Destino | 0.0.0.0/0.0.0.0 |
| Gateway | 192.168.19.2 |
| Interfaz de salida | port2 |
| Distancia | 10 |

📸 [`04-ruta-default.png`](evidencias/04-ruta-default.png)

---

### 3.4 Salida a Internet con NAT

**Menú:** `Policy & Objects > Firewall Policy > Create New`

| Campo | Valor |
|---|---|
| Nombre | USERS-a-Internet |
| Entrada | port3 (LAN-USERS) |
| Salida | port2 (WAN) |
| Origen | all |
| Destino | all |
| Servicio | ALL |
| Acción | ACCEPT |
| NAT | Habilitado — Use Outgoing Interface Address |

📸 [`05-policy-nat.png`](evidencias/05-policy-nat.png)

---

### 3.5 Restricción de tráfico hacia LAN-SERVIDORES (solo HTTP)

Entre LAN-USERS y LAN-SERVIDORES solo debe pasar HTTP; todo lo demás queda bloqueado por la regla implícita de FortiGate al no existir otra política que lo permita.

**Menú:** `Policy & Objects > Firewall Policy > Create New`

| Campo | Valor |
|---|---|
| Nombre | USERS-a-SERVIDORES-HTTP |
| Entrada | port3 (LAN-USERS) |
| Salida | port1 (LAN-SERVIDORES) |
| Origen | all |
| Destino | all |
| Servicio | HTTP |
| Acción | ACCEPT |
| NAT | Deshabilitado |
| Log | All Sessions |

> Esta regla debe quedar por encima de cualquier política más amplia entre estos dos segmentos, ya que FortiGate evalúa las políticas en orden y aplica la primera coincidencia.

📸 [`06-policy-http-only.png`](evidencias/06-policy-http-only.png)

---

### 3.6 Application Control — redes sociales

**Paso 1 — Crear el perfil:**

**Menú:** `Security Profiles > Application Control > Create New`

| Campo | Valor |
|---|---|
| Nombre | APPCTRL-USERS |
| Categoría Social.Media | Block |
| Aplicaciones desconocidas | Monitor |

**Paso 2 — Asignarlo a la política de salida a Internet:**

En `USERS-a-Internet > Security Profiles`, activar `Application Control: APPCTRL-USERS`.

📸 [`07-appctrl-social.png`](evidencias/07-appctrl-social.png)

---

### 3.7 Application Control — llamadas de WhatsApp

Dentro del mismo perfil `APPCTRL-USERS`, agrego una excepción puntual para bloquear solo la parte de llamadas de voz/video, dejando la mensajería de texto intacta.

**Menú:** `Security Profiles > Application Control > APPCTRL-USERS > Application Overrides > Add Signatures`

| Campo | Valor |
|---|---|
| Firma | WhatsApp_VoIP.Call |
| Acción | Block |

📸 [`08-whatsapp-call-block.png`](evidencias/08-whatsapp-call-block.png)

---

### 3.8 DNS/Web Filter — dominio institucional

Para bloquear un dominio completo (incluyendo subdominios) combino DNS Filter y Web Filter.

**DNS Filter — Menú:** `Security Profiles > DNS Filter > Create New`

| Campo | Valor |
|---|---|
| Nombre | DNSFILTER-USERS |
| Dominio | itla.edu.do |
| Tipo | Wildcard |
| Acción | Block |

**Web Filter — Menú:** `Security Profiles > Web Filter > Create New`

| Campo | Valor |
|---|---|
| Nombre | WEBFILTER-USERS |
| URL | *.itla.edu.do |
| Tipo | Wildcard |
| Acción | Block |

Ambos perfiles se asignan también en `USERS-a-Internet > Security Profiles`.

📸 [`09-dns-web-filter.png`](evidencias/09-dns-web-filter.png)

---

### 3.9 IPv4 DoS Policy — anti escaneo de puertos

Como la licencia instalada no trae el paquete completo de firmas IPS, no cuento con firmas dedicadas para detectar escaneos tipo Nmap. FortiOS resuelve esto de forma nativa con una **IPv4 DoS Policy**, que analiza comportamiento (tasa de paquetes/conexiones) en lugar de depender de una firma puntual, así que la uso para cubrir esta parte del laboratorio.

La coloco sobre **port2 (WAN)**, ya que es el punto de entrada donde quiero detectar intentos de reconocimiento hacia la red, sean estos originados desde la nube *Net* o simulados desde el propio host de LAN-USERS enrutando hacia afuera.

**Menú:** `Policy & Objects > IPv4 DoS Policy > Create New`

| Campo | Valor |
|---|---|
| Nombre | DOS-ANTISCAN-WAN |
| Interfaz de entrada | port2 (WAN) |
| Origen | all |
| Destino | all |
| Servicio | ALL |

**Anomalías a activar dentro de la misma política:**

| Anomalía | Logging | Acción | Umbral usado en el lab |
|---|---|---|---|
| tcp_port_scan | Enable | Block | 5 |
| udp_scan | Enable | Block | 5 |

> El umbral por defecto de estas anomalías está pensado para tráfico real de producción (cientos/miles de paquetes por segundo), así que lo bajé a 5 para poder ver el bloqueo disparándose en una prueba corta con Nmap.

**Comprobación:**

Desde el host de LAN-USERS:

```bash
nmap -sS -sU 192.168.19.10
```

Reviso el resultado en `Log & Report > Anomaly`, donde debe aparecer la anomalía disparada, el origen y la acción `Blocked`.

📸 [`10-dos-policy.png`](evidencias/10-dos-policy.png)
📸 [`11-dos-policy-anomalias.png`](evidencias/11-dos-policy-anomalias.png)
📸 [`12-log-anomaly-scan.png`](evidencias/12-log-anomaly-scan.png)

---

### 3.10 WAF para el servicio expuesto en LAN-SERVIDORES

El servidor de LAN-SERVIDORES publica un servicio web que quiero blindar contra SQL Injection, XSS y otros ataques de capa 7 antes de que el tráfico llegue a la aplicación.

**Paso 1 — Crear el perfil WAF:**

**Menú:** `Security Profiles > Web Application Firewall > Create New`

| Campo | Valor |
|---|---|
| Nombre | WAF-SERVIDORES |
| Signature groups (SQLi, XSS, Generic Attacks, Known Exploits) | Block |
| Constraint | Block |
| Métodos permitidos | GET, POST, HEAD |

**Paso 2 — Política que aplica el perfil sobre el tráfico entrante hacia LAN-SERVIDORES:**

**Menú:** `Policy & Objects > Firewall Policy > Create New`

| Campo | Valor |
|---|---|
| Nombre | USERS-a-SERVIDORES-WAF |
| Entrada | port3 (LAN-USERS) |
| Salida | port1 (LAN-SERVIDORES) |
| Servicio | HTTP, HTTPS |
| Acción | ACCEPT |
| Web Application Firewall | WAF-SERVIDORES |
| Log | All Sessions |

> Esta política reemplaza o complementa a la de la sección 3.5: si el objetivo es que todo el HTTP hacia LAN-SERVIDORES pase también por el WAF, el perfil se puede asignar directamente sobre `USERS-a-SERVIDORES-HTTP` en vez de crear una regla aparte.

📸 [`13-waf-servidores.png`](evidencias/13-waf-servidores.png)
📸 [`14-waf-sqli-bloqueado.png`](evidencias/14-waf-sqli-bloqueado.png)

---

## 4. Evidencias

Todas las capturas referenciadas a lo largo de este documento se encuentran en la carpeta [`evidencias/`](evidencias/).

| # | Captura | Qué muestra |
|---|---|---|
| 00 | [`00-cli-port2.png`](evidencias/00-cli-port2.png) | Consola del Fortinet configurando la IP y accesos de port2, y primer login exitoso a la GUI desde el navegador. |
| 01–02 | [`01-interface-port3.png`](evidencias/01-interface-port3.png), [`02-interface-port1.png`](evidencias/02-interface-port1.png) | Interfaces port3 (LAN-USERS) y port1 (LAN-SERVIDORES) con IP fija y rol LAN asignados. |
| 03 | [`03-dhcp-usuarios.png`](evidencias/03-dhcp-usuarios.png) | Servidor DHCP activo en port3 con el rango y gateway configurados. |
| 04 | [`04-ruta-default.png`](evidencias/04-ruta-default.png) | Ruta estática 0.0.0.0/0 apuntando hacia la salida por port2. |
| 05 | [`05-policy-nat.png`](evidencias/05-policy-nat.png) | Política de salida a Internet con NAT habilitado. |
| 06 | [`06-policy-http-only.png`](evidencias/06-policy-http-only.png) | Política que limita el tráfico LAN-USERS→LAN-SERVIDORES a solo HTTP. |
| 07 | [`07-appctrl-social.png`](evidencias/07-appctrl-social.png) | Perfil de Application Control bloqueando la categoría de redes sociales. |
| 08 | [`08-whatsapp-call-block.png`](evidencias/08-whatsapp-call-block.png) | Override de firma bloqueando específicamente las llamadas de WhatsApp. |
| 09 | [`09-dns-web-filter.png`](evidencias/09-dns-web-filter.png) | DNS/Web Filter bloqueando el dominio institucional y sus subdominios. |
| 10–12 | [`10-dos-policy.png`](evidencias/10-dos-policy.png), [`11-dos-policy-anomalias.png`](evidencias/11-dos-policy-anomalias.png), [`12-log-anomaly-scan.png`](evidencias/12-log-anomaly-scan.png) | DoS Policy en port2 con tcp_port_scan y udp_scan activos, y el log mostrando un escaneo bloqueado. |
| 13–14 | [`13-waf-servidores.png`](evidencias/13-waf-servidores.png), [`14-waf-sqli-bloqueado.png`](evidencias/14-waf-sqli-bloqueado.png) | Perfil WAF aplicado y un intento de SQL Injection bloqueado contra el servicio en LAN-SERVIDORES. |
| 15 | [15_bloqueo_rrss.png](evidencias/15_bloqueo_rrss.png) | Intento fallido de acceso a una red social (ej. facebook.com) desde la LAN de usuarios — página de bloqueo de FortiGate. |
| 16 | [16_bloqueo_itla.png](evidencias/16_bloqueo_itla.png) | Intento fallido de acceso a itla.edu.do página de bloqueo de FortiGate. |

---

## 5. Video

*(Añadir aquí el enlace al video demostrativo una vez grabado, mostrando rostro/voz, fecha/hora del sistema, y cada uno de los bloqueos y pruebas funcionando en vivo.)*

---

## 6. Referencias

* Fortinet Document Library — *FortiGate Administration Guide, Firewall Policies*.
* Fortinet Document Library — *FortiGate Administration Guide, Application Control*.
* Fortinet Document Library — *FortiGate Administration Guide, DoS Policy*.
* Fortinet Document Library — *FortiGate Administration Guide, Web Application Firewall*.
* Fortinet Community — *Technical Tip: detecting and blocking port scans without dedicated IPS signatures*.
