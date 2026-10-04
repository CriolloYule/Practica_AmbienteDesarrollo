# Guía de Sustentación — Práctica 6: Cortafuegos UFW y Reenvío de Puertos

Esta guía está diseñada exclusivamente para la defensa y sustentación oral del laboratorio ante el docente (**Prof. Oscar Mondragón**). Se encuentra estructurada de manera directa en **tres secciones principales**, correspondientes a los **tres puntos de evaluación** requeridos en el documento oficial `2025-01 Practica Firewall.pdf`.

Cada comando y acción incluye:
* **Objetivo:** Qué se busca verificar o ejecutar de forma concreta.
* **Significado de la salida:** Explicación técnica, corta y concisa de lo que demuestran los datos devueltos en consola.

---

## 0. Resumen de Arquitectura y Topología de Red

| Nodo | Dirección IP | Interfaz | Servicios Locales | Puertos | Rol de Seguridad en la Práctica |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **servidor1** | `192.168.50.3` | `eth1` | UFW Firewall<br>Apache SSL (443)<br>vsftpd (FTP 21) | `22/tcp` (SSH)<br>`21/tcp` (FTP Bloqueado)<br>`443/tcp` (HTTPS Local)<br>`80/tcp` (DNAT Forward) | Firewall perimetral y pasarela de reenvío NAT hacia `servidor2`. |
| **servidor2** | `192.168.50.2` | `eth1` | Apache HTTP Server | `22/tcp` (SSH)<br>`80/tcp` (HTTP Web) | Servidor Web de Destino. Recibe el tráfico redirigido desde `servidor1`. |
| **Host Windows** | `192.168.50.1` | `vboxnet` | PowerShell / Navegador | Dinámicos | Cliente evaluador que ejecuta las pruebas cruzadas. |

---

## SECCIÓN 1: Sustentación del Punto 1
### *"Configure la máquina servidor1 para que deniegue el servicio de ftp"*

### 1.1 Objetivo y Fundamento Teórico
Demostrar que el cortafuegos UFW descarta las conexiones entrantes dirigidas al servicio FTP (puerto `21/tcp`), aplicando la acción `DROP` a nivel de paquetes TCP SYN, mientras el demonio FTP local continúa en ejecución.

---

### 1.2 Comprobaciones en la Máquina Virtual (`servidor1`)

#### Comando 1: Verificar el estado del socket FTP local
* **Objetivo:** Comprobar en el kernel de `servidor1` si el demonio `vsftpd` está realmente encendido y escuchando en el puerto 21, descartando que la falla sea por servicio inactivo.
* **Comando:**
  ```bash
  sudo ss -tlnp | grep :21
  ```
* **Salida esperada:**
  ```text
  LISTEN 0      32           0.0.0.0:21        0.0.0.0:*    users:(("vsftpd",pid=...,fd=...))
  ```
* **Significado de la salida:**
  * `LISTEN`: El socket TCP está abierto y a la espera de solicitudes de conexión.
  * `0.0.0.0:21`: El servicio está vinculado al puerto 21 en todas las interfaces de red de la máquina.
  * `users:(("vsftpd",...))`: Identifica que el proceso demonio oficial de FTP (`vsftpd`) está activo en memoria y operativo.

#### Comando 2: Mostrar la regla de bloqueo en UFW
* **Objetivo:** Demostrar que el cortafuegos UFW tiene registrada y priorizada la regla explícita de bloqueo para el servicio FTP.
* **Comando:**
  ```bash
  sudo ufw status numbered | grep 21
  ```
* **Salida esperada:**
  ```text
  [ 2] 21/tcp                     DENY IN     Anywhere                   # Bloquear servicio FTP (Ejercicio 3.1)
  ```
* **Significado de la salida:**
  * `[ 2]`: Número de orden de la regla dentro de la lista de evaluación secuencial de UFW.
  * `21/tcp`: Regla que coincide exclusivamente con el protocolo de transporte TCP sobre el puerto destino 21.
  * `DENY IN Anywhere`: Aplica la política de descarte silencioso (`DROP` en iptables) a cualquier datagrama entrante desde cualquier dirección IP origen.

---

### 1.3 Demostración de Bloqueo desde el Cliente (Host Windows)

#### Acción 1: Prueba de conectividad TCP y Ping desde PowerShell
* **Objetivo:** Probar desde el cliente externo la conectividad en Capa 3 (IP/ICMP) y Capa 4 (TCP) hacia `servidor1` para corroborar el descarte en el puerto 21.
* **Comando:**
  ```powershell
  Test-NetConnection -ComputerName 192.168.50.3 -Port 21
  ```
* **Salida exacta en consola:**
  ```text
  WARNING: TCP connect to (192.168.50.3 : 21) failed

  ComputerName           : 192.168.50.3
  RemoteAddress          : 192.168.50.3
  RemotePort             : 21
  InterfaceAlias         : Ethernet 3
  SourceAddress          : 192.168.50.1
  PingSucceeded          : True
  PingReplyDetails (RTT) : 1 ms
  TcpTestSucceeded       : False
  ```
* **Significado de la salida:**
  * `PingSucceeded : True`: La máquina `servidor1` es plenamente alcanzable a nivel de Capa de Red (ICMP). La red física y virtual funciona correctamente.
  * `TcpTestSucceeded : False`: El establecimiento de la conexión TCP (apretón de manos de tres vías) falló rotundamente.
  * `WARNING: TCP connect ... failed`: Se agotó el tiempo de espera (*timeout*) porque UFW aplica `DROP`, ignorando el paquete SYN y sin enviar respuestas `RST` que delaten el estado del puerto.

---

### 1.4 Guion Técnico ante el Profesor (¿Qué decir?)
> *"Profesor, para dar cumplimiento al Punto 1, configuramos en `servidor1` la regla `sudo ufw deny 21/tcp`. Como se observa con `ss -tlnp`, el servicio `vsftpd` está corriendo en el socket local 21. Sin embargo, al probar la conexión desde Windows con `Test-NetConnection`, el ping es exitoso (`PingSucceeded: True`), pero la conexión TCP falla (`TcpTestSucceeded: False`). Esto comprueba que el bloqueo no se debe a que el servicio esté apagado, sino a que UFW descarta silenciosamente los paquetes TCP SYN entrantes aplicando la política DROP de iptables."*

---

### 1.5 Preguntas Frecuentes del Evaluador sobre el Punto 1
* **¿Por qué la conexión genera un tiempo de espera (timeout) en vez de decir 'Connection refused'?**
  * *Respuesta:* Porque la regla `deny` de UFW aplica la acción `DROP` en iptables, descartando el paquete sin emitir ninguna respuesta. Si usáramos `reject`, el kernel devolvería activamente un paquete TCP RST o ICMP Port Unreachable avisando que el puerto está cerrado. `DROP` es más seguro porque dificulta el escaneo sigiloso de puertos.
* **¿Por qué fue fundamental autorizar SSH antes de habilitar el firewall?**
  * *Respuesta:* Porque al habilitar UFW con `ufw enable`, la política por defecto entrante es `deny (incoming)`. Si no se autoriza previamente `22/tcp` con `sudo ufw allow ssh`, la sesión de administración remota se interrumpe de forma irreversible por red.

---

## SECCIÓN 2: Sustentación del Punto 2
### *"Configure la máquina servidor1 para que permita el acceso al servicio de https instalado en el mismo servidor"*

### 2.1 Objetivo y Fundamento Teórico
Comprobar que el cortafuegos autoriza el tráfico cifrado entrante en el puerto seguro `443/tcp` mediante la regla `allow https`, permitiendo la negociación de sesiones seguras bajo el protocolo **TLS 1.3** con un certificado digital X.509 alojado en el propio `servidor1`.

---

### 2.2 Comprobaciones en la Máquina Virtual (`servidor1`)

#### Comando 1: Mostrar la regla permitida en el estado de UFW
* **Objetivo:** Demostrar que el cortafuegos UFW tiene cargada y activa la regla explícita que autoriza el tráfico HTTPS entrante.
* **Comando:**
  ```bash
  sudo ufw status numbered | grep 443
  ```
* **Salida esperada:**
  ```text
  [ 3] 443/tcp                    ALLOW IN    Anywhere                   # Permitir servicio HTTPS local (Ejercicio 3.2)
  ```
* **Significado de la salida:**
  * `[ 3]`: Posición secuencial de la regla en la tabla activa de UFW.
  * `443/tcp`: Coincide con el puerto estándar de comunicaciones seguras TLS/HTTPS sobre TCP.
  * `ALLOW IN Anywhere`: Autoriza incondicionalmente el ingreso de paquetes SYN para iniciar conexiones seguras desde cualquier origen de red.

#### Comando 2: Comprobar el socket del servidor web Apache en HTTPS
* **Objetivo:** Verificar que el servidor Apache esté activo en `servidor1` y vinculado al puerto seguro 443 con soporte SSL/TLS cargado.
* **Comando:**
  ```bash
  sudo ss -tlnp | grep :443
  ```
* **Salida esperada:**
  ```text
  LISTEN 0      511          0.0.0.0:443       0.0.0.0:*    users:(("apache2",pid=...,fd=...))
  ```
* **Significado de la salida:**
  * `LISTEN`: Socket listo para recibir apretones de manos criptográficos TLS.
  * `0.0.0.0:443`: Escucha en todas las interfaces de red de `servidor1` en el puerto 443.
  * `users:(("apache2",...))`: Identifica que el proceso web Apache gestiona el puerto mediante el módulo `mod_ssl`.

---

### 2.3 Demostración de Acceso HTTPS desde el Cliente (Host Windows)

#### Acción 1: Petición HTTPS mediante cURL en Windows PowerShell
* **Objetivo:** Realizar una solicitud HTTPS real desde la consola del cliente para verificar la negociación TLS, los encabezados de respuesta y el cuerpo HTML entregado.
* **Comando:**
  ```powershell
  curl.exe -k -i https://192.168.50.3
  ```
* **Salida exacta en consola:**
  ```http
  HTTP/1.1 200 OK
  Date: Wed, 30 Sep 2026 00:52:56 GMT
  Server: Apache/2.4.52 (Ubuntu)
  Content-Type: text/html

  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8">
    <title>Servidor 1</title>
  </head>
  <body>
    <h1>Bienvenido al Servidor 1</h1>
  </body>
  </html>
  ```
* **Significado de la salida:**
  * `HTTP/1.1 200 OK`: La sesión TLS se negoció exitosamente y el servidor procesó y respondió la solicitud sin bloqueos de firewall.
  * `Server: Apache/2.4.52 (Ubuntu)`: Confirma que responde el servidor web nativo de `servidor1`.
  * `<h1>Bienvenido al Servidor 1</h1>`: Contenido HTML simple y exclusivo de `servidor1`, demostrando que el tráfico es local y no reenviado.

#### Acción 2: Prueba visual en el Navegador Web
* **Objetivo:** Demostrar la interacción gráfica del usuario final con el servicio web seguro a través de un navegador web estándar.
* **Procedimiento:** Abrir la URL `https://192.168.50.3` en Chrome o Edge.
* **Significado de la salida / pantalla:**
  * **Alerta `NET::ERR_CERT_AUTHORITY_INVALID`:** Confirma que la conexión viaja cifrada bajo TLS, pero el certificado X.509 fue autofirmado para fines de laboratorio y no por una Autoridad Certificadora (CA) comercial.
  * **Página cargada con *"Bienvenido al Servidor 1"*:** Al hacer clic en *"Avanzado $\rightarrow$ Continuar"*, se renderiza la página directamente desde `servidor1`, comprobando compatibilidad total con navegadores.

---

### 2.4 Guion Técnico ante el Profesor (¿Qué decir?)
> *"En el Punto 2, se configuró la regla `sudo ufw allow 443/tcp` en `servidor1`. En este servidor tenemos configurado Apache con `mod_ssl` y un certificado digital autofirmado X.509. Al realizar la petición HTTPS desde Windows con `curl.exe -k -i`, se completa exitosamente el apretón de manos TLS 1.3 en el puerto 443 y el servidor responde con código `200 OK` entregando la página con el saludo 'Bienvenido al Servidor 1'. Esto demuestra que el firewall autoriza limpiamente el tráfico web cifrado local."*

---

### 2.5 Preguntas Frecuentes del Evaluador sobre el Punto 2
* **¿Por qué el comando UFW acepta tanto 'https' como '443/tcp'?**
  * *Respuesta:* Porque UFW consulta la base de datos estándar de servicios de red en `/etc/services`. Al escribir `allow https`, UFW traduce automáticamente la palabra al puerto 443 y protocolo TCP.
* **¿Qué significa el flag `-k` en el comando `curl.exe`?**
  * *Respuesta:* Significa *insecure*, e instruye a cURL a omitir la validación de la Autoridad Certificadora (CA) raíz, necesario aquí porque utilizamos un certificado autofirmado de laboratorio que no pertenece a una CA pública comercial.

---

## SECCIÓN 3: Sustentación del Punto 3
### *"Configure el reenvío de puertos para que todas las peticiones al servicio http (80) entrantes a la máquina servidor1 sean redirigidas al servicio http (80) instalado en otra máquina servidor2"*

### 3.1 Objetivo y Fundamento Teórico
Implementar una pasarela de traducción de direcciones de red (**NAT**) en `servidor1` (`192.168.50.3`) para que cuando un cliente solicite tráfico web no seguro en el puerto `80/tcp`, los paquetes sean reenviados de forma transparente a `servidor2` (`192.168.50.2:80`), donde responde Apache con una página propia.

Para lograr esto se requieren tres elementos arquitectónicos en `servidor1`:
1. **Reenvío de paquetes IPv4 en el kernel:** `net.ipv4.ip_forward = 1`.
2. **Política de reenvío en UFW:** `DEFAULT_FORWARD_POLICY="ACCEPT"` en `/etc/default/ufw`.
3. **Reglas NAT en `/etc/ufw/before.rules`:**
   - **DNAT (Destination NAT) en PREROUTING:** Cambia la IP de destino de `192.168.50.3` a `192.168.50.2:80`.
   - **MASQUERADE (SNAT) en POSTROUTING:** Cambia la IP de origen a la de `servidor1`, resolviendo el problema de **enrutamiento asimétrico** en la misma subred.

---

### 3.2 Archivos Clave y Verificaciones en `servidor1`

#### Comando 1: Mostrar las reglas NAT en `/etc/ufw/before.rules`
* **Objetivo:** Inspeccionar las reglas de traducción de red que UFW inyecta en la tabla `nat` de iptables antes de evaluar las reglas de filtrado convencionales.
* **Comando:**
  ```bash
  sudo head -n 25 /etc/ufw/before.rules
  ```
* **Salida esperada:**
  ```text
  # NAT table rules
  *nat
  :PREROUTING ACCEPT [0:0]
  :POSTROUTING ACCEPT [0:0]
  -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.50.2:80
  -A POSTROUTING -d 192.168.50.2 -p tcp --dport 80 -j MASQUERADE
  COMMIT

  *filter
  :ufw-before-input - [0:0]
  :ufw-before-output - [0:0]
  :ufw-before-forward - [0:0]
  ```
* **Significado de la salida:**
  * `*nat`: Inicializa la tabla de traducción de direcciones en el subsistema Netfilter del kernel.
  * `-A PREROUTING ... -j DNAT --to-destination 192.168.50.2:80`: Regla **DNAT** que intercepta cualquier paquete entrante con destino al puerto 80 de `servidor1` y sustituye su dirección IP destino por la de `servidor2`.
  * `-A POSTROUTING -d 192.168.50.2 ... -j MASQUERADE`: Regla **SNAT/MASQUERADE** que reemplaza la IP de origen del cliente con la IP de `servidor1` (`192.168.50.3`), forzando a que las respuestas de `servidor2` regresen a `servidor1` y evitando el colapso por enrutamiento asimétrico en la misma subred.

#### Comando 2: Comprobar el reenvío de paquetes en el kernel
* **Objetivo:** Verificar que el kernel de Linux tiene activada la capacidad de actuar como enrutador/pasarela para conmutar paquetes entre nodos de red.
* **Comando:**
  ```bash
  sysctl net.ipv4.ip_forward
  ```
* **Salida esperada:**
  ```text
  net.ipv4.ip_forward = 1
  ```
* **Significado de la salida:**
  * `1`: El bit de reenvío está habilitado en el stack de red del kernel. Si estuviera en `0`, Linux operaría como host terminal y descartaría cualquier paquete cuyo destino no coincida con sus propias IPs.

#### Comando 3: Mostrar la autorización de tráfico enrutado en UFW
* **Objetivo:** Comprobar que el cortafuegos autoriza explícitamente el paso de paquetes hacia la IP y puerto de `servidor2` a través de la cadena `FORWARD`.
* **Comando:**
  ```bash
  sudo ufw status verbose | grep FWD
  ```
* **Salida esperada:**
  ```text
  192.168.50.2 80/tcp        ALLOW FWD   Anywhere
  ```
* **Significado de la salida:**
  * `ALLOW FWD`: Confirma que la cadena `FORWARD` de iptables cuenta con una regla permisiva, asegurando que el firewall no bloquee los datagramas mientras transitan por la máquina.

---

### 3.3 Verificación en el Servidor de Destino (`servidor2`)

#### Comando 1: Comprobar el servicio HTTP en `servidor2`
* **Objetivo:** Confirmar que el demonio Apache está levantado y escuchando en el puerto 80 del nodo de destino (`servidor2`).
* **Comando:**
  ```bash
  sudo ss -tlnp | grep :80
  ```
* **Salida esperada:**
  ```text
  LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("apache2",pid=...,fd=...))
  ```
* **Significado de la salida:**
  * `LISTEN`: Socket HTTP disponible para recibir conexiones.
  * `0.0.0.0:80`: Escuchando peticiones en todas las interfaces de `servidor2`.

#### Comando 2: Inspeccionar la página web propia de `servidor2`
* **Objetivo:** Validar el archivo web local servido por `servidor2` para distinguir inequívocamente su contenido del de `servidor1`.
* **Comando:**
  ```bash
  cat /var/www/html/index.html
  ```
* **Salida esperada:**
  ```html
  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8">
    <title>Servidor 2</title>
  </head>
  <body>
    <h1>Bienvenido al Servidor 2</h1>
  </body>
  </html>
  ```
* **Significado de la salida:**
  * El texto `<h1>Bienvenido al Servidor 2</h1>` sirve como huella de identidad del servidor de destino.

---

### 3.4 Demostración de Reenvío de Puertos desde el Cliente (Host Windows)

#### Acción 1: Petición HTTP al puerto 80 de `servidor1` desde PowerShell
* **Objetivo:** Comprobar la redirección transparente enviando una petición HTTP al puerto 80 de `servidor1` (`192.168.50.3`) y verificando que responda `servidor2`.
* **Comando:**
  ```powershell
  curl.exe -i http://192.168.50.3
  ```
* **Salida exacta en consola:**
  ```http
  HTTP/1.1 200 OK
  Date: Wed, 30 Sep 2026 00:52:56 GMT
  Server: Apache/2.4.52 (Ubuntu)
  Content-Type: text/html

  <!DOCTYPE html>
  <html lang="es">
  <head>
    <meta charset="UTF-8">
    <title>Servidor 2</title>
  </head>
  <body>
    <h1>Bienvenido al Servidor 2</h1>
  </body>
  </html>
  ```
* **Significado de la salida:**
  * Aunque la conexión se inició hacia la dirección IP de `servidor1` (`192.168.50.3`), el cuerpo HTML devuelto es `Bienvenido al Servidor 2`.
  * Esto demuestra que la regla DNAT interceptó la solicitud entrante en el puerto 80, la reenvió a `servidor2`, y la regla MASQUERADE devolvió la respuesta al cliente sin que este detecte el cambio de servidor.

#### Acción 2: Prueba en Navegador Web
* **Objetivo:** Demostrar de forma gráfica la transparencia del port forwarding para el usuario final en un navegador web.
* **Procedimiento:** Abrir la URL `http://192.168.50.3` en el navegador.
* **Significado de la salida / pantalla:**
  * La barra de direcciones conserva `http://192.168.50.3`, pero se renderiza el título:
    > **Bienvenido al Servidor 2**
  * Demuestra que el cortafuegos UFW actúa como una pasarela invisible (*Reverse Proxy / Port Forwarding NAT*).

---

### 3.5 Verificación en Vivo de los Contadores de Paquetes en `servidor1`

#### Comando 1: Inspeccionar contadores de la regla DNAT en tiempo real
* **Objetivo:** Mostrar al profesor que el kernel está traduciendo los paquetes físicamente mediante los contadores en tiempo real de iptables.
* **Comando:**
  ```bash
  sudo iptables -t nat -L PREROUTING -n -v
  ```
* **Salida esperada:**
  ```text
  Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
   pkts bytes target     prot opt in     out     source               destination         
     14   840 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:192.168.50.2:80
  ```
* **Significado de la salida:**
  * `pkts` (14) y `bytes` (840): Cantidad acumulada de paquetes y bytes procesados por la regla DNAT. Al ejecutar una nueva petición `curl`, estos números se incrementan inmediatamente, probando en vivo la actividad del firewall.
  * `to:192.168.50.2:80`: Indica la dirección IP y puerto al que se reescribe el encabezado IP del paquete.

---

### 3.6 Guion Técnico ante el Profesor (¿Qué decir?)
> *"Profesor, para resolver el Punto 3, configuramos el reenvío de puertos (Port Forwarding) utilizando la tabla NAT de iptables dentro de `/etc/ufw/before.rules`. En `servidor1` definimos la regla DNAT en la cadena PREROUTING para interceptar cualquier paquete entrante con puerto destino 80 y cambiar su destino a `192.168.50.2:80` (servidor2). Además, agregamos la regla MASQUERADE en POSTROUTING para reemplazar la IP de origen con la de `servidor1`, solucionando el problema de enrutamiento asimétrico en la misma subred. Como se observa en la prueba con cURL, nos conectamos a la IP de `servidor1` (`192.168.50.3:80`), pero el contenido devuelto es 'Bienvenido al Servidor 2', demostrando la redirección transparente."*

---

### 3.7 Preguntas Frecuentes del Evaluador sobre el Punto 3
* **¿Por qué se debe usar la cadena PREROUTING y no POSTROUTING para DNAT?**
  * *Respuesta:* Porque `PREROUTING` actúa apenas el paquete entra a la interfaz de red, **antes** de que el kernel tome la decisión de enrutamiento (*Routing Decision*). Necesitamos cambiar la IP de destino a `192.168.50.2` antes de que el kernel decida si el paquete debe ser procesado localmente o reenviado por otra interfaz.
* **¿Qué es el problema de enrutamiento asimétrico (triangular) y por qué se necesita MASQUERADE aquí?**
  * *Respuesta:* Como el cliente Windows (`192.168.50.1`) y los dos servidores están en la misma subred `192.168.50.0/24`, si no usamos MASQUERADE, `servidor2` recibiría el paquete con IP de origen de Windows y le respondería **directamente** a Windows sin pasar por `servidor1`. Windows recibiría una respuesta con IP de origen `192.168.50.2` cuando él le habló a `192.168.50.3`, por lo que descartaría la conexión (TCP RST). Con `MASQUERADE`, `servidor1` enmascara la IP de origen, obligando a `servidor2` a responderle a él, y `servidor1` entrega la respuesta de vuelta al cliente de forma simétrica.
* **¿Por qué se requiere habilitar `net.ipv4.ip_forward = 1` y `DEFAULT_FORWARD_POLICY="ACCEPT"`?**
  * *Respuesta:* Por defecto Linux actúa como un anfitrión final (*end-system*), descartando paquetes que no vayan dirigidos a sus propias IPs. `ip_forward = 1` activa el comportamiento de enrutador en el kernel, y `DEFAULT_FORWARD_POLICY="ACCEPT"` en UFW evita que la cadena de filtrado `FORWARD` bloquee los datagramas que cruzan la máquina.
