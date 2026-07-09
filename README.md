 # Implementación de Topología de Seguridad Perimetral en FortiGate v7.4.12

**Autor:** Henry Vicente Quezada
**Matrícula:** 2025-1332
**Asignatura / Práctica:** Práctica 5 — Configuración de FortiGate
**Firmware:** FortiOS v7.4.12
**Plataforma:** FortiGate VM64 (entorno virtualizado con VMware Workstation)

---

## Enlace al video demostrativo

📹 **Video de demostración (máx. 8 minutos):** [Ver video](https://youtu.be/shvBOsZEPXs)

*El video incluye la demostración del correcto funcionamiento de la topología, 
mostrando evidencias por GUI, la topología con nombre y matrícula, hora y fecha del sistema.*

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

![Topología de red](<img width="429" height="333" alt="1  Topología" src="https://github.com/user-attachments/assets/89671125-1de7-4e3f-9671-3540ce5245d3" />
)

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

![Interfaces de red configuradas](<img width="572" height="391" alt="2  Interfaces e IP" src="https://github.com/user-attachments/assets/326fd22b-4980-49ab-b3ad-a751f6ca2e33" />
 )

*La imagen anterior muestra las tres interfaces físicas configuradas en el FortiGate (port1, port2, port3) con sus respectivas direcciones IP, tal como se definieron en la tabla anterior.*

---

## 4. Configuraciones implementadas

A continuación se detalla cada una de las configuraciones realizadas, con su evidencia correspondiente capturada desde la GUI de FortiGate.

### 4.1 Servidor DHCP en la LAN de Usuarios

Se habilitó el servicio DHCP en la interfaz **port2**, con el fin de asignar direccionamiento IP dinámico a los equipos conectados a la LAN de usuarios.

![Configuración de DHCP Server](<img width="309" height="133" alt="3  DHCP-configuracion" src="https://github.com/user-attachments/assets/74e476b7-0167-4219-9d88-6269b37521c2" />
)

**Parámetros configurados:**
- Rango de direcciones: `192.168.13.2 - 192.168.13.126`
- Máscara de subred: `255.255.255.128`
- Gateway: mismo que la IP de la interfaz (192.168.13.2)
- Servidor DNS: mismo que el DNS del sistema
- Tiempo de concesión (lease): 604800 segundos (7 días)

La siguiente captura confirma, desde el cliente Windows 10, que la dirección IP fue asignada correctamente por el servidor DHCP del FortiGate:

![Confirmación de IP asignada por DHCP en el cliente](<img width="395" height="257" alt="4  DHCP Cliente-ipconfig" src="https://github.com/user-attachments/assets/3694683b-cb6f-4484-bf72-160d89d29bc4" />
 )

El cliente recibió la IP `192.168.13.3`, con gateway `192.168.13.2` y servidor DHCP `192.168.13.2`, confirmando el correcto funcionamiento del servicio.

---

### 4.2 Ruta por defecto

Se configuró una ruta estática por defecto que dirige todo el tráfico sin destino conocido hacia la interfaz WAN (port1), utilizando como siguiente salto la puerta de enlace de la red externa.

![Configuración de la ruta por defecto](<img width="443" height="271" alt="4  Ruta por defecto" src="https://github.com/user-attachments/assets/b6188edf-9b85-40ba-b62f-9e054081dd18" />
)

**Parámetros configurados:**
- Destino: `0.0.0.0/0.0.0.0`
- Interfaz de salida: `port1`
- Gateway: `192.168.210.2`
- Distancia administrativa: 10 (valor por defecto)

---

### 4.3 NAT (Network Address Translation)

Para permitir que los dispositivos con direccionamiento privado de las LANs internas puedan salir a Internet, se habilitó NAT en la política de firewall correspondiente a la salida hacia Internet, utilizando la dirección de la interfaz de salida (port1).

![NAT habilitado en la política de salida a Internet](<img width="398" height="327" alt="5  NAT" src="https://github.com/user-attachments/assets/a43a337c-f07b-4c7f-b27f-cafd7c8be4c0" />
)

**Configuración aplicada:**
- Política: `LAN-Usuarios_a_Internet`
- NAT: Habilitado
- IP Pool: `Use Outgoing Interface Address`
- Manage source port: Preserve source port

---

### 4.4 Restricción de tráfico: solo HTTP de Usuarios a Servidores

Se implementaron dos políticas de firewall consecutivas para controlar estrictamente el tráfico entre la LAN de usuarios y la LAN de servidores:

1. Una política que **permite únicamente el servicio HTTP** desde la LAN de usuarios hacia la LAN de servidores.
2. Una política inmediatamente posterior que **deniega cualquier otro tipo de tráfico** entre ambos segmentos.

![Políticas de firewall: HTTP permitido y todo lo demás denegado](<img width="720" height="328" alt="image" src="https://github.com/user-attachments/assets/31cb67e7-5fb5-44bb-963c-3da1a3dcaadc" />
)

El orden de evaluación es crítico en FortiGate, ya que las políticas se procesan de arriba hacia abajo; por ello, la política de aceptación de HTTP se ubicó antes que la política de denegación general.

**Prueba de funcionamiento — HTTP permitido:**

Se accedió desde el navegador del cliente Windows 10 al sitio web alojado en el servidor (`http://192.168.32.3`), confirmando que el servicio HTTP responde correctamente:

![Acceso HTTP exitoso al servidor web](<img width="709" height="656" alt="image" src="https://github.com/user-attachments/assets/fcdd977c-5864-4c05-9193-829975177524" />)

**Prueba de funcionamiento — Todo lo demás bloqueado:**

Se ejecutó un `ping` (ICMP) desde el cliente Windows 10 hacia el servidor, confirmando que el tráfico distinto a HTTP es correctamente bloqueado por la política DENY:

![Ping bloqueado hacia el servidor](<img width="466" height="296" alt="prueba-ping-bloqueado" src="https://github.com/user-attachments/assets/c5fa1394-9312-46f0-bb0b-781395d38ab6" />
)

El resultado `Request timed out` en las cuatro peticiones ICMP, con 100% de pérdida de paquetes, demuestra que la política de restricción funciona como se esperaba.

---

### 4.5 Bloqueo de redes sociales

Se configuró un perfil de **Application Control** llamado `Block-Social-Media`, en el cual se estableció la categoría **Social Media** en modo **Block**. Este perfil se aplicó a la política de salida a Internet (`LAN-Usuarios_a_Internet`).

![Perfil de Application Control con la categoría Social Media bloqueada](<img width="530" height="392" alt="social-media-perfil" src="https://github.com/user-attachments/assets/e5bc1cd2-adb8-4f1b-b7d2-cf0ed0182417" />
)

**Prueba de funcionamiento:**

Al intentar acceder desde el navegador del cliente a Facebook e Instagram, la conexión es interrumpida por el motor de inspección SSL del FortiGate, evidenciando que el tráfico está siendo interceptado y bloqueado:

![Facebook bloqueado](<img width="479" height="452" alt="social-media-bloqueado png Facebook" src="https://github.com/user-attachments/assets/26a2c6b5-d9c4-4168-8da0-48a8d4abb635" />
)

![Instagram bloqueado](<img width="480" height="442" alt="social-media-bloqueado png Instagram png" src="https://github.com/user-attachments/assets/8dc9649c-906e-433a-b78d-e43953583625" />
)

**Evidencia en logs:**

El registro de tráfico (`Forward Traffic`) confirma múltiples eventos de bloqueo (`Deny: UTM Blocked`) para las aplicaciones Facebook e Instagram, mientras que el tráfico correspondiente a servicios permitidos (como Microsoft Portal) continúa siendo aceptado con normalidad:

![Logs de bloqueo de redes sociales](<img width="475" height="329" alt="social-media-log png" src="https://github.com/user-attachments/assets/85a8d738-9c63-4b1a-95a5-637e65d538b5" />
)

---

### 4.6 Bloqueo de llamadas de WhatsApp

Dentro del mismo perfil de Application Control, se agregó una regla de anulación (*Application Override*) específica para la firma **WhatsApp_VoIP.Call**, configurada en modo **Block**. Esto permite que la mensajería de WhatsApp continúe funcionando con normalidad, restringiendo únicamente el establecimiento de llamadas de voz y video.

![Firma de llamadas de WhatsApp bloqueada en Application Control](<img width="394" height="262" alt="whatsapp-perfil" src="https://github.com/user-attachments/assets/f37482f5-4aa3-4f52-96e9-38179a1eaec4" />
)

Esta configuración se aplicó sobre la misma política de salida a Internet, garantizando que cualquier intento de establecer una llamada de voz o video mediante WhatsApp sea evaluado contra esta firma específica antes de permitir el tráfico.

---

### 4.7 Bloqueo de itla.edu.do y subdominios

Se configuró un perfil de **Web Filter** llamado `Block-ITLA`, en el cual se agregaron dos entradas de tipo *Static URL Filter* con coincidencia wildcard:

- `itla.edu.do` (dominio exacto)
- `*.itla.edu.do` (cualquier subdominio)

![Perfil de Web Filter con el dominio itla.edu.do bloqueado](<img width="697" height="563" alt="image" src="https://github.com/user-attachments/assets/f91e4b96-a198-4052-be51-6320243ab057" />
)

**Prueba de funcionamiento:**

Al intentar acceder a `https://itla.edu.do` desde el cliente Windows 10, FortiGate presenta su página de bloqueo nativa, confirmando que la URL fue correctamente identificada y denegada por el filtro configurado:

![Página de bloqueo de FortiGate al acceder a itla.edu.do](<img width="480" height="439" alt="itla-bloqueado" src="https://github.com/user-attachments/assets/dd626e42-9e95-49f7-a135-283319bbc6c7" />
)

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

![Política DoS con anomalías de escaneo configuradas en modo Block](<img width="893" height="652" alt="image" src="https://github.com/user-attachments/assets/3053fc84-959f-40b3-9069-83e1ce8abe61" />
)

**b) Perfil de Intrusion Prevention System (IPS):**

Se creó un perfil IPS personalizado llamado `Block-Scanners`, en el cual se incluyeron firmas específicas orientadas a la detección de herramientas de escaneo y reconocimiento de vulnerabilidades (Acunetix Web Vulnerability Scanner, Apache Tomcat Remote Exploit Account Scanner, Canvas FTPd Scan, entre otras), todas configuradas en modo **Block**.

![Perfil IPS con firmas de detección de escáneres](<img width="399" height="324" alt="ips-scanner" src="https://github.com/user-attachments/assets/2ba417f3-ec75-4b24-be5f-58088d1c4d3c" />
)

Ambos mecanismos —DoS Policy e IPS— trabajan de forma complementaria: el primero detecta patrones de comportamiento anómalo (múltiples conexiones en un corto periodo de tiempo), mientras que el segundo identifica firmas específicas asociadas a herramientas conocidas de escaneo.

---

### 4.9 Web Application Firewall (WAF) en el servidor web

Se habilitó la funcionalidad de Web Application Firewall desde **Feature Visibility**, y se creó un perfil personalizado llamado `WAF-Servidor-Web`, activando la protección contra las siguientes categorías de ataque:

- Cross-Site Scripting (XSS) — Extended
- SQL Injection
- Generic Attacks
- Known Exploits
- Credit Card Detection

![Perfil de WAF con firmas de protección activadas](<img width="371" height="324" alt="waf-perfil png" src="https://github.com/user-attachments/assets/c7b9751d-e8ea-4c62-b1d6-4d5de5d39e62" />
)

Debido a que el motor de WAF de FortiGate requiere que la política opere en modo de inspección **Proxy-based**, se modificó el modo de inspección de la política `Usuarios_a_Servidores_HTTP`, y se asignó el perfil WAF creado:

![Política en modo Proxy-based con el perfil WAF aplicado](images/20-waf-politica.png)

**Prueba de funcionamiento:**

Tras aplicar el perfil WAF, se verificó que el sitio web del servidor continúa siendo accesible con normalidad desde el cliente, confirmando que la protección no interfiere con el tráfico legítimo:

![Sitio web funcionando correctamente con WAF activo](<img width="310" height="253" alt="waf-politica" src="https://github.com/user-attachments/assets/c96b1502-34ef-4a3f-89d2-81ceab14483f" />
)

---

## 5. Pruebas generales de conectividad

Como validación adicional del correcto funcionamiento de la ruta por defecto y el NAT configurado, se realizaron las siguientes pruebas desde el cliente Windows 10:

**Ping hacia el gateway de la LAN de usuarios (192.168.13.2):**

![Ping exitoso hacia el gateway](<img width="374" height="232" alt="ping-gateway" src="https://github.com/user-attachments/assets/a12009fa-6c6b-4d15-8aad-5ad7844d7d58" />
)

**Ping hacia un servidor DNS público de Internet (8.8.8.8):**

![Ping exitoso hacia Internet](<img width="304" height="179" alt="ping-internet" src="https://github.com/user-attachments/assets/da23387d-25f5-40ec-99a5-f556ebedb2b8" />
)

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

