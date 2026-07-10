 Implementación de Topología de Seguridad Perimetral en FortiGate v7.4.12

Autor: Henry Vicente Quezada
Matrícula: 2025-1332
Asignatura / Práctica: Asignación 2 — Configuración de FortiGate
Firmware: FortiOS v7.4.12
Plataforma: FortiGate VM64 (entorno virtualizado con VMware Workstation)


📹 Enlace al video demostrativo

Video de demostración (máx. 8 minutos): https://youtu.be/shvBOsZEPXs

El video incluye la demostración del correcto funcionamiento de la topología, mostrando evidencias por GUI, la topología con nombre y matrícula, y hora y fecha del sistema.


1. Objetivo de la red

El objetivo de esta práctica es diseñar e implementar, en su totalidad a través de la interfaz gráfica (GUI) de FortiGate, una topología de red segmentada que separe el tráfico de usuarios y el tráfico de servidores, controlando de forma granular las comunicaciones entre ambos segmentos y hacia Internet.

La red busca garantizar:


Segmentación: separación física/lógica entre la LAN de usuarios y la LAN de servidores, evitando que el tráfico entre ambas fluya sin control.
Acceso controlado a Internet: salida a Internet para los usuarios internos mediante NAT, con inspección de tráfico (Application Control, Web Filter, IPS) para mitigar riesgos de fuga de información, uso de aplicaciones no autorizadas y acceso a sitios prohibidos institucionalmente.
Disponibilidad de servicios internos: exposición controlada del servicio web (HTTP) del servidor interno únicamente hacia la LAN de usuarios, restringiendo cualquier otro tipo de tráfico (ICMP, RDP, SSH, etc.) entre ambos segmentos.
Protección del servidor web: aplicación de un perfil de Web Application Firewall (WAF) para proteger la aplicación web publicada contra ataques comunes como SQL Injection y Cross-Site Scripting (XSS).
Detección de amenazas: identificación y bloqueo de actividades de reconocimiento de red (escaneo de puertos) mediante políticas de anomalías (DoS Policy) y firmas de Intrusion Prevention System (IPS).


Todo el direccionamiento IP utilizado en esta práctica está basado en la matrícula del autor (2025-1332), conforme lo solicitado en el enunciado.


2. Topología de red

La topología implementada consta de un FortiGate como punto central de control, con tres interfaces físicas: una hacia Internet (WAN) y dos hacia las redes internas (LAN de usuarios y LAN de servidores).

Mostrar imagen

Descripción de la topología:

SegmentoInterfaz FortiGateRango de redDispositivo conectadoWAN / Internetport1 (e0/0)192.168.210.0/24Gateway de salida (192.168.210.2)LAN de Usuariosport2 (e0/2)192.168.13.0/25Windows 10 (WINDOW-CLIENT-L)LAN de Servidoresport3 (e0/1)192.168.32.0/28Windows Server 2022 (WAF + IIS)

El equipo cliente (Windows 10) recibe direccionamiento dinámico vía DHCP, mientras que el servidor (Windows Server 2022) posee una dirección fija dentro del rango de la LAN de servidores.


3. Direccionamiento IP

InterfazDirección IPMáscaraDescripciónport1 (WAN)192.168.210.128/24Salida a Internetport2 (LAN Usuarios)192.168.13.2/25Gateway de la LAN de usuariosport3 (LAN Servidores)192.168.32.2/28Gateway de la LAN de servidoresCliente Windows 10192.168.13.3/25Asignada por DHCPServidor Windows Server192.168.32.3/28EstáticaGateway de salida a Internet192.168.210.2/24Next-hop de la ruta por defecto

Mostrar imagen


4. Configuraciones implementadas

A continuación se detalla cada una de las configuraciones realizadas, con su evidencia correspondiente capturada desde la GUI de FortiGate.

4.1 Servidor DHCP en la LAN de Usuarios

Se habilitó el servicio DHCP en la interfaz port2, con el fin de asignar direccionamiento IP dinámico a los equipos conectados a la LAN de usuarios.

Parámetros configurados:


Rango de direcciones: 192.168.13.2 - 192.168.13.126
Máscara de subred: 255.255.255.128
Gateway: mismo que la IP de la interfaz (192.168.13.2)
Servidor DNS: mismo que el DNS del sistema
Tiempo de concesión (lease): 604800 segundos (7 días)


Mostrar imagen

El cliente recibió la IP 192.168.13.3, con gateway 192.168.13.2 y servidor DHCP 192.168.13.2, confirmando el correcto funcionamiento del servicio (verificado mediante ipconfig /all desde el cliente Windows 10).

Mostrar imagen


4.2 Ruta por defecto

Se configuró una ruta estática por defecto que dirige todo el tráfico sin destino conocido hacia la interfaz WAN (port1), utilizando como siguiente salto la puerta de enlace de la red externa.

Parámetros configurados:


Destino: 0.0.0.0/0.0.0.0
Interfaz de salida: port1
Gateway: 192.168.210.2
Distancia administrativa: 10 (valor por defecto)


Mostrar imagen


4.3 NAT (Network Address Translation)

Para permitir que los dispositivos con direccionamiento privado de las LANs internas puedan salir a Internet, se habilitó NAT en la política de firewall correspondiente a la salida hacia Internet, utilizando la dirección de la interfaz de salida (port1).

Configuración aplicada:


Política: LAN-Usuarios_a_Internet
NAT: Habilitado
IP Pool: Use Outgoing Interface Address
Manage source port: Preserve source port


Mostrar imagen


4.4 Restricción de tráfico: solo HTTP de Usuarios a Servidores

Se implementaron dos políticas de firewall consecutivas para controlar estrictamente el tráfico entre la LAN de usuarios y la LAN de servidores:


Una política que permite únicamente el servicio HTTP desde la LAN de usuarios hacia la LAN de servidores.
Una política inmediatamente posterior que deniega cualquier otro tipo de tráfico entre ambos segmentos.


Mostrar imagen

El orden de evaluación es crítico en FortiGate, ya que las políticas se procesan de arriba hacia abajo; por ello, la política de aceptación de HTTP se ubicó antes que la política de denegación general.

Prueba de funcionamiento — HTTP permitido:

Se accedió desde el navegador del cliente Windows 10 al sitio web alojado en el servidor (http://192.168.32.3), confirmando que el servicio HTTP responde correctamente.

Mostrar imagen

Prueba de funcionamiento — Todo lo demás bloqueado:

Se ejecutó un ping (ICMP) desde el cliente Windows 10 hacia el servidor, confirmando que el tráfico distinto a HTTP es correctamente bloqueado por la política DENY. El resultado Request timed out en las cuatro peticiones ICMP, con 100% de pérdida de paquetes, demuestra que la política de restricción funciona como se esperaba.

Mostrar imagen


4.5 Bloqueo de redes sociales

Se configuró un perfil de Application Control llamado Block-Social-Media, en el cual se estableció la categoría Social Media en modo Block. Este perfil se aplicó a la política de salida a Internet (LAN-Usuarios_a_Internet).

Mostrar imagen

Prueba de funcionamiento:

Al intentar acceder desde el navegador del cliente a Facebook e Instagram, la conexión es interrumpida por el motor de inspección SSL del FortiGate, evidenciando que el tráfico está siendo interceptado y bloqueado.

Mostrar imagen

Mostrar imagen

Evidencia en logs:

El registro de tráfico (Forward Traffic) confirma múltiples eventos de bloqueo (Deny: UTM Blocked) para las aplicaciones Facebook e Instagram, mientras que el tráfico correspondiente a servicios permitidos (como Microsoft Portal) continúa siendo aceptado con normalidad.

Mostrar imagen


4.6 Bloqueo de llamadas de WhatsApp

Dentro del mismo perfil de Application Control, se agregó una regla de anulación (Application Override) específica para la firma WhatsApp_VoIP.Call, configurada en modo Block. Esto permite que la mensajería de WhatsApp continúe funcionando con normalidad, restringiendo únicamente el establecimiento de llamadas de voz y video.

Esta configuración se aplicó sobre la misma política de salida a Internet, garantizando que cualquier intento de establecer una llamada de voz o video mediante WhatsApp sea evaluado contra esta firma específica antes de permitir el tráfico.

Mostrar imagen

Prueba de funcionamiento:

Para validar el bloqueo, se inició una llamada de voz/video desde WhatsApp Web en el cliente Windows 10. Al revisar el log de Forward Traffic (Log & Report → Forward Traffic) durante y después de la llamada, la sesión aparece con Action: Accept dentro de la política LAN-Usuarios_a_Internet, con un volumen de tráfico de 421.16 kB / 956.64 kB — consistente con una llamada de voz/video en curso, no con simple mensajería de texto.

Este resultado indica que, pese a que la firma WhatsApp_VoIP.Call fue configurada correctamente en modo Block dentro del perfil de Application Control y aplicada a la política de salida a Internet, la llamada no fue efectivamente bloqueada en las pruebas realizadas. Esto se atribuye a que WhatsApp utiliza mecanismos de cifrado y multiplexación del tráfico de señalización y medios (confirmado mediante análisis de tráfico con tcpdump, que mostró toda la sesión viajando sobre TCP/443 sin diferenciación de puertos UDP típicos de VoIP) que dificultan la clasificación determinística de esta firma específica por parte del motor de Application Control, incluso con SSL Deep Inspection habilitado.

Es un comportamiento documentado en configuraciones reales de FortiGate, donde la detección de firmas VoIP dentro de aplicaciones de mensajería cifradas de extremo a extremo puede ser inconsistente según la versión del cliente (Web vs. aplicación nativa) y el nivel de inspección SSL aplicado.

Figura 14b. Registro de Forward Traffic mostrando la sesión de la llamada de WhatsApp con Action: Accept (421.16 kB / 956.64 kB transferidos), confirmando que la llamada no fue bloqueada en la prueba.

Se recomienda, para un entorno de producción, validar el comportamiento con la aplicación de escritorio nativa de WhatsApp en lugar de la versión web, y monitorear los logs de Application Control durante un periodo extendido para ajustar la configuración según el tráfico real observado.


4.7 Bloqueo de itla.edu.do y subdominios

Se configuró un perfil de Web Filter llamado Block-ITLA, en el cual se agregaron dos entradas de tipo Static URL Filter con coincidencia wildcard:


itla.edu.do (dominio exacto)
*.itla.edu.do (cualquier subdominio)


Prueba de funcionamiento:

Al intentar acceder a https://itla.edu.do desde el cliente Windows 10, FortiGate presenta su página de bloqueo nativa, confirmando que la URL fue correctamente identificada y denegada por el filtro configurado.

Mostrar imagen


4.8 Detección y bloqueo de escáneres de red

Para la detección de actividades de reconocimiento de red (escaneo de puertos), se implementaron dos mecanismos complementarios:

a) Política de DoS (Denial of Service):

Se creó una política DoS sobre la interfaz port2, habilitando y configurando en modo Block las siguientes anomalías de capa 3 y capa 4:


ip_src_session / ip_dst_session
tcp_syn_flood
tcp_port_scan
tcp_src_session / tcp_dst_session
udp_flood / udp_scan


Mostrar imagen

b) Perfil de Intrusion Prevention System (IPS):

Se creó un perfil IPS personalizado llamado Block-Scanners, en el cual se incluyeron firmas específicas orientadas a la detección de herramientas de escaneo y reconocimiento de vulnerabilidades (Acunetix Web Vulnerability Scanner, Apache Tomcat Remote Exploit Account Scanner, Canvas FTPd Scan, entre otras), todas configuradas en modo Block.

Mostrar imagen

Ambos mecanismos —DoS Policy e IPS— trabajan de forma complementaria: el primero detecta patrones de comportamiento anómalo (múltiples conexiones en un corto periodo de tiempo), mientras que el segundo identifica firmas específicas asociadas a herramientas conocidas de escaneo.

Prueba de funcionamiento:

Como validación adicional, se ejecutó un escaneo de puertos con Nmap (nmap -sS -Pn -p- 192.168.32.3) desde un cliente de la LAN de usuarios (192.168.13.4) hacia el servidor, con el fin de confirmar que la política DoS detecta y bloquea este tipo de actividad de reconocimiento.

El escaneo, que abarcó los 65535 puertos del servidor, resultó en 65532 puertos reportados como filtered por Nmap (sin respuesta), evidencia de que el tráfico fue descartado activamente por las políticas de firewall. De forma complementaria, el log de Log & Report → Security Events → Anomaly confirmó que la política DoS detectó el patrón de escaneo mediante la firma tcp_port_scan, registrando múltiples eventos de severidad Critical originados desde el cliente escaneador, con la acción clear_session aplicada por la política Anti-ScanDetect. Esto demuestra que ambos mecanismos —segmentación por política y detección de anomalías— funcionan de forma coordinada.

Mostrar imagen


4.9 Web Application Firewall (WAF) en el servidor web

Se habilitó la funcionalidad de Web Application Firewall desde Feature Visibility, y se creó un perfil personalizado llamado WAF-Servidor-Web, activando la protección contra las siguientes categorías de ataque:


Cross-Site Scripting (XSS) — Extended
SQL Injection
Generic Attacks
Known Exploits
Credit Card Detection


Mostrar imagen

Debido a que el motor de WAF de FortiGate requiere que la política opere en modo de inspección Proxy-based, se modificó el modo de inspección de la política Usuarios_a_Servidores_HTTP, y se asignó el perfil WAF creado.

Mostrar imagen

Prueba de funcionamiento — tráfico legítimo:

Tras aplicar el perfil WAF, se verificó que el sitio web del servidor continúa siendo accesible con normalidad desde el cliente, confirmando que la protección no interfiere con el tráfico legítimo.

Mostrar imagen

Prueba de funcionamiento — ataque bloqueado:

Adicionalmente, para validar que el WAF no solo permite el tráfico legítimo sino que también detiene ataques reales, se envió desde el navegador del cliente una petición con un payload de inyección SQL hacia el servidor:

http://192.168.32.3/?id=1' OR '1'='1

FortiGate interceptó la solicitud y presentó su página nativa de bloqueo de Web Application Firewall, confirmando que la transferencia fue detenida por firma (Event ID 30000040, Event Type: signature).

Mostrar imagen

Esta prueba, en conjunto con la anterior, demuestra que el perfil WAF-Servidor-Web está correctamente activo: permite el tráfico HTTP legítimo sin interferir, y al mismo tiempo detecta y bloquea patrones de ataque conocidos como SQL Injection.


5. Pruebas generales de conectividad

Como validación adicional del correcto funcionamiento de la ruta por defecto y el NAT configurado, se realizaron las siguientes pruebas desde el cliente Windows 10:

Ping hacia el gateway de la LAN de usuarios (192.168.13.2): exitoso, 0% de pérdida.

Mostrar imagen

Ping hacia un servidor DNS público de Internet (8.8.8.8): exitoso, 0% de pérdida.

Mostrar imagen

Ambas pruebas confirman conectividad local (hacia el FortiGate) y conectividad externa (hacia Internet a través del NAT y la ruta por defecto configurada), validando el correcto funcionamiento de la infraestructura de red implementada.


6. Resumen de cumplimiento de requisitos

#RequisitoEstado1Configuración realizada íntegramente por GUI✅ Cumplido2Acceso a Internet✅ Cumplido3LAN de usuarios (/25)✅ Cumplido — 192.168.13.0/254LAN de servidores (/28)✅ Cumplido — 192.168.32.0/285IP en interfaces✅ Cumplido6DHCP en LAN de usuarios✅ Cumplido7Ruta por defecto✅ Cumplido8NAT✅ Cumplido9Solo HTTP de Usuarios a Servidores✅ Cumplido10Bloqueo de redes sociales✅ Cumplido11Bloqueo de llamadas de WhatsApp⚠️ Configurado — bloqueo no confirmado en pruebas12Bloqueo de itla.edu.do y subdominios✅ Cumplido13Detección y bloqueo de escáneres de red✅ Cumplido14WAF aplicado al servidor web✅ Cumplido


7. Conclusiones

La implementación de esta topología permitió aplicar de forma práctica los principales mecanismos de seguridad perimetral disponibles en FortiGate: segmentación de red, control de acceso mediante políticas de firewall, traducción de direcciones (NAT), y perfiles de seguridad avanzados (Application Control, Web Filter, IPS y WAF).

La configuración realizada demuestra un enfoque de seguridad en capas (defense in depth), donde cada mecanismo cumple una función específica: el control de políticas garantiza la segmentación del tráfico entre LANs, los perfiles de Application Control y Web Filter restringen el uso de aplicaciones y sitios no autorizados, el sistema IPS y la política DoS detectan actividades de reconocimiento, y el WAF protege específicamente la aplicación web publicada contra los vectores de ataque más comunes en el entorno actual de amenazas.

Cabe destacar que, durante las pruebas, el bloqueo de llamadas de WhatsApp no pudo confirmarse de forma consistente pese a estar correctamente configurado, lo cual evidencia una limitación real del motor de Application Control frente a tráfico cifrado y multiplexado de aplicaciones de mensajería modernas — un hallazgo que en sí mismo forma parte del aprendizaje práctico de esta implementación.


Documento elaborado como parte de la Asignación 2 — Configuración de FortiGate — Henry Vicente Quezada, matrícula 2025-1332.
