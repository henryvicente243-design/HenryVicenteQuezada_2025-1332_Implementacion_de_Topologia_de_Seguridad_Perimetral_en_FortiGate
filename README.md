# Implementación de Topología de Seguridad Perimetral en FortiGate v7.4.12

**Autor:** Henry Vicente Quezada
**Matrícula:** 2025-1332
**Asignatura / Práctica:** Asignación 2 — Configuración de FortiGate
**Firmware:** FortiOS v7.4.12
**Plataforma:** FortiGate VM64 (entorno virtualizado con VMware Workstation)

---

## 📹 Enlace al video demostrativo

**Video de demostración (máx. 8 minutos):** (https://youtu.be/shvBOsZEPXs)

El video incluye la demostración del correcto funcionamiento de la topología, mostrando evidencias por GUI, la topología con nombre y matrícula, y hora y fecha del sistema.

---

## 1. Objetivo de la red

El objetivo de esta práctica es diseñar e implementar, en su totalidad a través de la interfaz gráfica (GUI) de FortiGate, una topología de red segmentada que separe el tráfico de usuarios y el tráfico de servidores, controlando de forma granular las comunicaciones entre ambos segmentos y hacia Internet.

La red busca garantizar:

- **Segmentación**: separación física/lógica entre la LAN de usuarios y la LAN de servidores, evitando que el tráfico entre ambas fluya sin control.
- **Acceso controlado a Internet**: salida a Internet para los usuarios internos mediante NAT, con inspección de tráfico (Application Control, Web Filter, IPS) para mitigar riesgos de fuga de información, uso de aplicaciones no autorizadas y acceso a sitios prohibidos institucionalmente.
- **Disponibilidad de servicios internos**: exposición controlada del servicio web (HTTP) del servidor interno únicamente hacia la LAN de usuarios, restringiendo cualquier otro tipo de tráfico (ICMP, RDP, SSH, etc.) entre ambos segmentos.
- **Protección del servidor web**: aplicación de un perfil de Web Application Firewall (WAF) para proteger la aplicación web publicada contra ataques comunes como SQL Injection y Cross-Site Scripting (XSS).
- **Detección de amenazas**: identificación y bloqueo de actividades de reconocimiento de red (escaneo de puertos) mediante políticas de anomalías (DoS Policy) y firmas de Intrusion Prevention System (IPS).

Todo el direccionamiento IP utilizado en esta práctica está basado en la matrícula del autor (**2025-1332**), conforme lo solicitado en el enunciado.

---

## 2. Topología de red

La topología implementada consta de un FortiGate como punto central de control, con tres interfaces físicas: una hacia Internet (WAN) y dos hacia las redes internas (LAN de usuarios y LAN de servidores).

<img width="754" height="580" alt="image" src="https://github.com/user-attachments/assets/f80af16f-efe6-4772-953d-07b4e834f84a" />


**Descripción de la topología:**

| Segmento | Interfaz FortiGate | Rango de red | Dispositivo conectado |
|---|---|---|---|
| WAN / Internet | port1 (e0/0) | 192.168.210.0/24 | Gateway de salida (192.168.210.2) |
| LAN de Usuarios | port2 (e0/2) | 192.168.13.0/25 | Windows 10 (WINDOW-CLIENT-L) |
| LAN de Servidores | port3 (e0/1) | 192.168.32.0/28 | Windows Server 2022 (WAF + IIS) |

El equipo cliente (Windows 10) recibe direccionamiento dinámico vía DHCP, mientras que el servidor (Windows Server 2022) posee una dirección fija dentro del rango de la LAN de servidores.

---

## 3. Direccionamiento IP

| Interfaz | Dirección IP | Máscara | Descripción |
|---|---|---|---|
| port1 (WAN) | 192.168.210.128 | /24 | Salida a Internet |
| port2 (LAN Usuarios) | 192.168.13.2 | /25 | Gateway de la LAN de usuarios |
| port3 (LAN Servidores) | 192.168.32.2 | /28 | Gateway de la LAN de servidores |
| Cliente Windows 10 | 192.168.13.3 | /25 | Asignada por DHCP |
| Servidor Windows Server | 192.168.32.3 | /28 | Estática |
| Gateway de salida a Internet | 192.168.210.2 | /24 | Next-hop de la ruta por defecto |

 <img width="900" height="480" alt="image" src="https://github.com/user-attachments/assets/a1133df8-4013-45ad-822b-f6072dc2be0f" />


---

## 4. Configuraciones implementadas

A continuación se detalla cada una de las configuraciones realizadas, con su evidencia correspondiente capturada desde la GUI de FortiGate.

### 4.1 Servidor DHCP en la LAN de Usuarios

Se habilitó el servicio DHCP en la interfaz **port2**, con el fin de asignar direccionamiento IP dinámico a los equipos conectados a la LAN de usuarios.

**Parámetros configurados:**
- Rango de direcciones: `192.168.13.2 - 192.168.13.126`
- Máscara de subred: `255.255.255.128`
- Gateway: mismo que la IP de la interfaz (192.168.13.2)
- Servidor DNS: mismo que el DNS del sistema
- Tiempo de concesión (lease): 604800 segundos (7 días)

 <img width="707" height="351" alt="image" src="https://github.com/user-attachments/assets/ac240244-4566-4784-8093-e9399a16749c" />


El cliente recibió la IP `192.168.13.3`, con gateway `192.168.13.2` y servidor DHCP `192.168.13.2`, confirmando el correcto funcionamiento del servicio (verificado mediante `ipconfig /all` desde el cliente Windows 10).

 <img width="715" height="468" alt="image" src="https://github.com/user-attachments/assets/f1b7a535-1c90-45b0-839f-36b07067ae7c" />

---

### 4.2 Ruta por defecto

Se configuró una ruta estática por defecto que dirige todo el tráfico sin destino conocido hacia la interfaz WAN (port1), utilizando como siguiente salto la puerta de enlace de la red externa.

**Parámetros configurados:**
- Destino: `0.0.0.0/0.0.0.0`
- Interfaz de salida: `port1`
- Gateway: `192.168.210.2`
- Distancia administrativa: 10 (valor por defecto)

 <img width="684" height="367" alt="image" src="https://github.com/user-attachments/assets/aff6ec57-5489-4dd2-8c06-9fa26773be7a" />

---

### 4.3 NAT (Network Address Translation)

Para permitir que los dispositivos con direccionamiento privado de las LANs internas puedan salir a Internet, se habilitó NAT en la política de firewall correspondiente a la salida hacia Internet, utilizando la dirección de la interfaz de salida (port1).

**Configuración aplicada:**
- Política: `LAN-Usuarios_a_Internet`
- NAT: Habilitado
- IP Pool: `Use Outgoing Interface Address`
- Manage source port: Preserve source port

 <img width="755" height="520" alt="image" src="https://github.com/user-attachments/assets/33660e5b-0bce-4af3-bb2c-fedb4816b98e" />

---

### 4.4 Restricción de tráfico: solo HTTP de Usuarios a Servidores

Se implementaron dos políticas de firewall consecutivas para controlar estrictamente el tráfico entre la LAN de usuarios y la LAN de servidores:

1. Una política que **permite únicamente el servicio HTTP** desde la LAN de usuarios hacia la LAN de servidores.
2. Una política inmediatamente posterior que **deniega cualquier otro tipo de tráfico** entre ambos segmentos.

<img width="901" height="409" alt="image" src="https://github.com/user-attachments/assets/737ea2ff-1091-4dcd-ad3c-4516926967e8" />


El orden de evaluación es crítico en FortiGate, ya que las políticas se procesan de arriba hacia abajo; por ello, la política de aceptación de HTTP se ubicó antes que la política de denegación general.

**Prueba de funcionamiento — HTTP permitido:**

Se accedió desde el navegador del cliente Windows 10 al sitio web alojado en el servidor (`http://192.168.32.3`), confirmando que el servicio HTTP responde correctamente.

 <img width="642" height="597" alt="image" src="https://github.com/user-attachments/assets/aedc417d-667c-42c8-a0e2-c8aa69506250" />

**Prueba de funcionamiento — Todo lo demás bloqueado:**

Se ejecutó un `ping` (ICMP) desde el cliente Windows 10 hacia el servidor, confirmando que el tráfico distinto a HTTP es correctamente bloqueado por la política DENY. El resultado `Request timed out` en las cuatro peticiones ICMP, con 100% de pérdida de paquetes, demuestra que la política de restricción funciona como se esperaba.

 <img width="829" height="499" alt="image" src="https://github.com/user-attachments/assets/b531bf46-158a-4373-a6bf-6af639262893" />

---

### 4.5 Bloqueo de redes sociales

Se configuró un perfil de **Application Control** llamado `Block-Social-Media`, en el cual se estableció la categoría **Social Media** en modo **Block**. Este perfil se aplicó a la política de salida a Internet (`LAN-Usuarios_a_Internet`).

<img width="832" height="615" alt="image" src="https://github.com/user-attachments/assets/fecab040-baa0-40c5-81a4-e04d7d521337" />


**Prueba de funcionamiento:**

Al intentar acceder desde el navegador del cliente a Facebook e Instagram, la conexión es interrumpida por el motor de inspección SSL del FortiGate, evidenciando que el tráfico está siendo interceptado y bloqueado.

 <img width="714" height="671" alt="image" src="https://github.com/user-attachments/assets/ba49e390-84a9-4002-98fb-d06f98e1a641" />

 ---

 <img width="681" height="624" alt="image" src="https://github.com/user-attachments/assets/355768dc-bde0-41b4-838e-d8095553316f" />


**Evidencia en logs:**

El registro de tráfico (`Forward Traffic`) confirma múltiples eventos de bloqueo (`Deny: UTM Blocked`) para las aplicaciones Facebook e Instagram, mientras que el tráfico correspondiente a servicios permitidos (como Microsoft Portal) continúa siendo aceptado con normalidad.

 <img width="621" height="435" alt="image" src="https://github.com/user-attachments/assets/82e280b6-9383-49ee-b9a2-a6cb85745c90" />

---

 ### 4.6 Bloqueo de llamadas de WhatsApp

Dentro del mismo perfil de Application Control, se agregó una regla de anulación (*Application Override*) específica para la firma **WhatsApp_VoIP.Call**, configurada en modo **Block**. Esto permite que la mensajería de WhatsApp continúe funcionando con normalidad, restringiendo únicamente el establecimiento de llamadas de voz y video.

Esta configuración se aplicó sobre la misma política de salida a Internet, garantizando que cualquier intento de establecer una llamada de voz o video mediante WhatsApp sea evaluado contra esta firma específica antes de permitir el tráfico.

<img width="583" height="389" alt="image" src="https://github.com/user-attachments/assets/7a0149bd-7ed7-4817-b688-38ef3b2ca456" />

**funcionamiento:**

Log de Forward Traffic: la sesión correspondiente a la llamada se registró con Action: BLOCK
Log de Application Control: se detectó la firma WhatsApp_VoIP.Call con Action: Blocked
Comportamiento observado: la llamada no logró establecerse, quedando bloqueada antes de iniciar la comunicación de voz/video

El bloqueo de llamadas de voz/video de WhatsApp se validó correctamente tras habilitar SSL Deep Inspection con el certificado de CA confiado en el endpoint, lo que permitió al motor de Application Control identificar y bloquear el flujo de tráfico VoIP cifrado antes de su establecimiento.

---

### 4.7 Bloqueo de itla.edu.do y subdominios

Se configuró un perfil de **Web Filter** llamado `Block-ITLA`, en el cual se agregaron dos entradas de tipo *Static URL Filter* con coincidencia wildcard:

- `itla.edu.do` (dominio exacto)
- `*.itla.edu.do` (cualquier subdominio)

**Prueba de funcionamiento:**

Al intentar acceder a `https://itla.edu.do` desde el cliente Windows 10, FortiGate presenta su página de bloqueo nativa, confirmando que la URL fue correctamente identificada y denegada por el filtro configurado.

 <img width="743" height="600" alt="image" src="https://github.com/user-attachments/assets/1e3075f9-e408-4ae4-8dee-1a7ada076a3e" />

<img width="707" height="642" alt="image" src="https://github.com/user-attachments/assets/f4f61df0-8d22-412a-8d4f-7dfd93f186ef" />

---

### 4.8 Detección y bloqueo de escáneres de red

Para la detección de actividades de reconocimiento de red (escaneo de puertos), se implementaron dos mecanismos complementarios:

**a) Política de DoS (Denial of Service):**

Se creó una política DoS sobre la interfaz `port2`, habilitando y configurando en modo **Block** las siguientes anomalías de capa 3 y capa 4:

- `ip_src_session` / `ip_dst_session`
- `tcp_syn_flood`
- `tcp_port_scan`
- `tcp_src_session` / `tcp_dst_session`
- `udp_flood` / `udp_scan`

 <img width="799" height="557" alt="image" src="https://github.com/user-attachments/assets/d032efbe-4e70-419a-974c-2e1408e1bd9e" />


**b) Perfil de Intrusion Prevention System (IPS):**

Se creó un perfil IPS personalizado llamado `Block-Scanners`, en el cual se incluyeron firmas específicas orientadas a la detección de herramientas de escaneo y reconocimiento de vulnerabilidades (Acunetix Web Vulnerability Scanner, Apache Tomcat Remote Exploit Account Scanner, Canvas FTPd Scan, entre otras), todas configuradas en modo **Block**.

 <img width="740" height="601" alt="image" src="https://github.com/user-attachments/assets/71be608b-a1cd-4442-9343-58c118d10191" />


Ambos mecanismos —DoS Policy e IPS— trabajan de forma complementaria: el primero detecta patrones de comportamiento anómalo (múltiples conexiones en un corto periodo de tiempo), mientras que el segundo identifica firmas específicas asociadas a herramientas conocidas de escaneo.

**Prueba de funcionamiento:**

Como validación adicional, se ejecutó un escaneo de puertos con Nmap (`nmap -sS -Pn -p- 192.168.32.3`) desde un cliente de la LAN de usuarios (192.168.13.4) hacia el servidor, con el fin de confirmar que la política DoS detecta y bloquea este tipo de actividad de reconocimiento.

El escaneo, que abarcó los 65535 puertos del servidor, resultó en 65532 puertos reportados como `filtered` por Nmap (sin respuesta), evidencia de que el tráfico fue descartado activamente por las políticas de firewall. De forma complementaria, el log de `Log & Report → Security Events → Anomaly` confirmó que la política DoS detectó el patrón de escaneo mediante la firma `tcp_port_scan`, registrando múltiples eventos de severidad **Critical** originados desde el cliente escaneador, con la acción `clear_session` aplicada por la política `Anti-ScanDetect`. Esto demuestra que ambos mecanismos —segmentación por política y detección de anomalías— funcionan de forma coordinada.

 <img width="985" height="467" alt="image" src="https://github.com/user-attachments/assets/7215a3e5-3136-42fb-820f-c16be14569bc" />

---

### 4.9 Web Application Firewall (WAF) en el servidor web

Se habilitó la funcionalidad de Web Application Firewall desde **Feature Visibility**, y se creó un perfil personalizado llamado `WAF-Servidor-Web`, activando la protección contra las siguientes categorías de ataque:

- Cross-Site Scripting (XSS) — Extended
- SQL Injection
- Generic Attacks
- Known Exploits
- Credit Card Detection

 <img width="719" height="628" alt="image" src="https://github.com/user-attachments/assets/477ed38c-662a-422b-b3c5-a16be33a4561" />

Debido a que el motor de WAF de FortiGate requiere que la política opere en modo de inspección **Proxy-based**, se modificó el modo de inspección de la política `Usuarios_a_Servidores_HTTP`, y se asignó el perfil WAF creado.

 <img width="695" height="580" alt="image" src="https://github.com/user-attachments/assets/a2cf398b-7a56-46ba-a9ff-e144578e46c7" />


**Prueba de funcionamiento — tráfico legítimo:**

Tras aplicar el perfil WAF, se verificó que el sitio web del servidor continúa siendo accesible con normalidad desde el cliente, confirmando que la protección no interfiere con el tráfico legítimo.

<img width="704" height="622" alt="image" src="https://github.com/user-attachments/assets/0046aac6-741d-45c7-9c2a-d351814de38d" />


**Prueba de funcionamiento — ataque bloqueado:**

Adicionalmente, para validar que el WAF no solo permite el tráfico legítimo sino que también detiene ataques reales, se envió desde el navegador del cliente una petición con un payload de inyección SQL hacia el servidor:

```
http://192.168.32.3/?id=1' OR '1'='1
```

FortiGate interceptó la solicitud y presentó su página nativa de bloqueo de Web Application Firewall, confirmando que la transferencia fue detenida por firma (Event ID 30000040, Event Type: signature).

 <img width="704" height="622" alt="image" src="https://github.com/user-attachments/assets/3256b998-481b-4b67-8deb-ee024bb13ad3" />


Esta prueba, en conjunto con la anterior, demuestra que el perfil `WAF-Servidor-Web` está correctamente activo: permite el tráfico HTTP legítimo sin interferir, y al mismo tiempo detecta y bloquea patrones de ataque conocidos como SQL Injection.

---

## 5. Pruebas generales de conectividad

Como validación adicional del correcto funcionamiento de la ruta por defecto y el NAT configurado, se realizaron las siguientes pruebas desde el cliente Windows 10:

**Ping hacia el gateway de la LAN de usuarios (192.168.13.2):** exitoso, 0% de pérdida.

 <img width="631" height="392" alt="image" src="https://github.com/user-attachments/assets/f72a43b1-fb5f-4262-bbb6-a87d8a2c9d59" />


**Ping hacia un servidor DNS público de Internet (8.8.8.8):** exitoso, 0% de pérdida.

 <img width="659" height="390" alt="image" src="https://github.com/user-attachments/assets/d5f47e51-d414-4805-b827-aedf309d370a" />


Ambas pruebas confirman conectividad local (hacia el FortiGate) y conectividad externa (hacia Internet a través del NAT y la ruta por defecto configurada), validando el correcto funcionamiento de la infraestructura de red implementada.

---

## 6. Resumen de cumplimiento de requisitos

| # | Requisito | Estado |
|---|---|---|
| 1 | Configuración realizada íntegramente por GUI | ✅ Cumplido |
| 2 | Acceso a Internet | ✅ Cumplido |
| 3 | LAN de usuarios (/25) | ✅ Cumplido — 192.168.13.0/25 |
| 4 | LAN de servidores (/28) | ✅ Cumplido — 192.168.32.0/28 |
| 5 | IP en interfaces | ✅ Cumplido |
| 6 | DHCP en LAN de usuarios | ✅ Cumplido |
| 7 | Ruta por defecto | ✅ Cumplido |
| 8 | NAT | ✅ Cumplido |
| 9 | Solo HTTP de Usuarios a Servidores | ✅ Cumplido |
| 10 | Bloqueo de redes sociales | ✅ Cumplido |
| 11 | Bloqueo de llamadas de WhatsApp | ✅ Cumplido |
| 12 | Bloqueo de itla.edu.do y subdominios | ✅ Cumplido |
| 13 | Detección y bloqueo de escáneres de red | ✅ Cumplido |
| 14 | WAF aplicado al servidor web | ✅ Cumplido |

---
## 7. Conclusiones

La implementación de esta topología permitió aplicar de forma práctica los principales mecanismos de seguridad perimetral disponibles en FortiGate: segmentación de red, control de acceso mediante políticas de firewall, traducción de direcciones (NAT), y perfiles de seguridad avanzados (Application Control, Web Filter, IPS y WAF).

La configuración realizada demuestra un enfoque de seguridad en capas (*defense in depth*), donde cada mecanismo cumple una función específica: el control de políticas garantiza la segmentación del tráfico entre LANs, los perfiles de Application Control y Web Filter restringen el uso de aplicaciones y sitios no autorizados, el sistema IPS y la política DoS detectan actividades de reconocimiento, y el WAF protege específicamente la aplicación web publicada contra los vectores de ataque más comunes en el entorno actual de amenazas.

Las pruebas realizadas —incluyendo el escaneo de puertos con Nmap y el intento de inyección SQL contra el servidor web— confirmaron que las políticas y perfiles configurados responden correctamente ante actividad maliciosa real, validando la efectividad de la implementación en un entorno práctico.
 


