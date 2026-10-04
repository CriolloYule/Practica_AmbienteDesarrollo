# Guía de Sustentación Rápida — Práctica 5: Servidores Web Seguros con Apache y Nginx (SSL/TLS)

Este documento sirve como instrumento directo de apoyo para la sustentación del laboratorio técnico ante el docente o evaluador. Está estructurado rigurosamente en **Paso 0** (preparación, arquitectura, certificados y archivos clave), **Paso 1** (demostración de Apache seguro) y **Paso 2** (instalación y demostración de Nginx seguro), concluyendo con el banco de preguntas frecuentes de defensa.

---

## Paso 0: Preparación del Entorno, Arquitectura y Material Criptográfico (Archivos y Certificados)

En esta fase se sustenta la infraestructura base, el estándar X.509, las directivas de configuración de los servidores y el aseguramiento del material criptográfico.

### 0.1 Resumen de Arquitectura, Topología y Roles

| Nodo | IP Privada | Servicio Principal | Puertos Escuchando | Rol Técnico y Criptográfico |
| :--- | :--- | :--- | :--- | :--- |
| **servidor** | `192.168.50.3` | **Apache2** (`mod_ssl`) | `80/TCP` (HTTP)<br>`443/TCP` (HTTPS) | Servidor Web seguro principal. Aloja el VirtualHost `servicios.com` con certificado autofirmado X.509 RSA de 2048 bits. |
| **servidor2** | `192.168.50.2` | **Nginx** (SSL/TLS) | `80/TCP` (HTTP)<br>`443/TCP` (HTTPS) | Servidor Web secundario (Ejercicio 2). Aloja el bloque de servidor SSL con terminación TLS y página segura independiente. |
| **Host (Windows)** | `192.168.50.1` | Cliente / Navegador | Dinámicos de salida | Realiza peticiones HTTPS, inspección de certificados X.509, pruebas con `curl` e inspección del canal seguro. |

> [!NOTE]
> **Modalidad Mono-nodo Alternativa:** Si el evaluador solicita ejecutar ambos servidores web en la misma máquina virtual (`servidor`), Apache conserva los puertos estándar `80/443`, mientras que Nginx se configura en el puerto seguro alternativo `8443/TCP`, evitando colisiones de socket (`Address already in use`).

---

### 0.2 Certificados Digitales X.509 y Protección de Claves Privadas

#### A. Inspección del Certificado Digital X.509 Autofirmado
* **Ubicación:** `servidor` (`/etc/ssl/certs/apache-selfsigned.crt`)
* **Comando para mostrar:**
  ```bash
  openssl x509 -in /etc/ssl/certs/apache-selfsigned.crt -noout -text | grep -E "(Issuer:|Subject:|Not Before|Not After|Public Key Algorithm|RSA Public-Key)" -A 1
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Aquí inspeccionamos el certificado digital bajo el estándar ITU-T X.509 generado con OpenSSL. Como se observa en la salida, el `Issuer` y el `Subject` son idénticos (`CN = server.servicios.com`), confirmando que es un certificado autofirmado (Self-Signed) con algoritmo de clave pública RSA de 2048 bits y una vigencia exacta de 365 días."*

#### B. Permisos de Seguridad de la Clave Privada (Menor Privilegio)
* **Ubicación:** `servidor` (`/etc/ssl/private/apache-selfsigned.key`)
* **Comando para mostrar:**
  ```bash
  sudo ls -la /etc/ssl/private/apache-selfsigned.key
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Por principio de menor privilegio y seguridad criptográfica, la clave privada generada con el flag `-nodes` tiene permisos estrictos `600` (`-rw-------`) perteneciendo exclusivamente al usuario `root`. Esto previene que usuarios no privilegiados del sistema operativo puedan extraer el material criptográfico con el que se descifra el tráfico de la sesión."*

---

### 0.3 Archivos de Configuración Base y Páginas de Inicio

#### A. Configuración del Virtual Host Apache SSL
* **Ubicación:** `servidor` (`/etc/apache2/sites-available/servicios.com.conf`)
* **Comando para mostrar limpiamente:**
  ```bash
  cat /etc/apache2/sites-available/servicios.com.conf
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"En este archivo definimos dos bloques de VirtualHost. El primero escucha en el puerto seguro `443` con `SSLEngine on`, asociando el certificado digital público en `/etc/ssl/certs/apache-selfsigned.crt` y la clave privada en `/etc/ssl/private/apache-selfsigned.key`. El segundo bloque atiende en el puerto `80` para tráfico estándar, garantizando la compatibilidad HTTP."*

#### B. Página Web de Inicio Segura de Apache
* **Ubicación:** `servidor` (`/var/www/html/index.html`)
* **Comando para mostrar:**
  ```bash
  head -n 25 /var/www/html/index.html
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Es el documento HTML raíz servido bajo cifrado SSL/TLS. Cuenta con estructura semántica y estilos CSS integrados para identificar visualmente que la conexión viaja autenticada e íntegra a través del túnel seguro de la práctica."*

---

## Paso 1: Demostración del Funcionamiento del Servicio Web Seguro en Apache (Ejercicio 1)

En esta fase se demuestra al evaluador que el servidor web Apache se encuentra operativo, con el módulo SSL cargado, escuchando en los puertos respectivos y negociando correctamente el protocolo HTTPS de forma local y remota.

### 1.1 Verificación del Servicio, Módulos y Puertos en el Servidor (`servidor`)

1. **Verificar estado activo del servicio Apache:**
   ```bash
   sudo systemctl status apache2 --no-pager
   ```
   * **Resultado esperado:** Estado en verde `active (running)`.

2. **Comprobar puertos HTTP (80) y HTTPS (443) en escucha:**
   ```bash
   sudo ss -tlnp | grep -E ':(80|443)'
   ```
   * **Resultado esperado:** Sockets en estado `LISTEN` en `:::80` y `:::443` pertenecientes al proceso `apache2`.

3. **Verificar que el módulo `mod_ssl` está cargado en runtime:**
   ```bash
   sudo apache2ctl -M | grep ssl
   ```
   * **Resultado esperado:** `ssl_module (shared)`.

4. **Validar la sintaxis de configuración:**
   ```bash
   sudo apache2ctl configtest
   ```
   * **Resultado esperado:** `Syntax OK`.

---

### 1.2 Validación Local del Canal Seguro y Handshake Criptográfico

5. **Prueba local HTTPS e inspección de cabeceras:**
   ```bash
   curl -kIv https://localhost
   ```
   * **Resultado esperado:** Handshake TLS exitoso, código `HTTP/1.1 200 OK`, cabecera `Server: Apache/...` y cuerpo HTML devuelto.

6. **Comprobación del Handshake Criptográfico con OpenSSL `s_client`:**
   ```bash
   echo | openssl s_client -connect localhost:443 -brief
   ```
   * **Resultado esperado:** `CONNECTION ESTABLISHED`, protocolo `TLSv1.3` (o `TLSv1.2`) y cipher suite segura (ej. `TLS_AES_256_GCM_SHA384`).

---

### 1.3 Demostración Remota desde el Host Anfitrión (Windows)

1. **Comprobar accesibilidad de red en el puerto 443 de Apache:**
   ```powershell
   Test-NetConnection -ComputerName 192.168.50.3 -Port 443
   ```
   * **Resultado esperado:** `TcpTestSucceeded : True`.

2. **Petición HTTPS con cURL ignorando el certificado autofirmado (`-k`):**
   ```powershell
   curl.exe -k -i https://192.168.50.3
   ```
   * **Resultado esperado:** Cabecera `HTTP/1.1 200 OK` y el cuerpo HTML personalizado servido por Apache.

3. **Visualización y comprobación en Navegador Web:**
   * Abrir `https://192.168.50.3` en Chrome, Edge o Firefox.
   * **Resultado esperado:** Se despliega la advertencia esperada de certificado autofirmado (`NET::ERR_CERT_AUTHORITY_INVALID`). Al hacer clic en *"Avanzado $\rightarrow$ Continuar a 192.168.50.3"*, carga la página web con el candado de seguridad. Al pulsar en el candado $\rightarrow$ *"El certificado no es válido"*, se observa la entidad emisora `server.servicios.com` generada en la práctica.

---

## Paso 2: Instalación, Configuración y Demostración del Servicio Web Seguro en Nginx (Ejercicio 2)

En esta fase se demuestra la instalación, configuración del bloque de servidor SSL nativo y el funcionamiento integral de Nginx en `servidor2` (o puerto alternativo), contrastando su arquitectura frente a Apache.

### 2.1 Instalación y Material Criptográfico de Nginx

Si el evaluador solicita revisar el procedimiento de despliegue en `servidor2`:
```bash
# Instalación del servidor web y utilidades criptográficas
sudo apt update && sudo apt install -y nginx openssl

# Generación del par de claves y certificado para Nginx
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/C=CO/ST=Valle/L=Cali/O=UAO/OU=ServiciosTelematicos/CN=www.servicios-nginx.com"

# Aseguramiento de permisos
sudo chmod 600 /etc/ssl/private/nginx-selfsigned.key
```

---

### 2.2 Archivo de Configuración del Bloque de Servidor SSL en Nginx

* **Ubicación:** `servidor2` (`/etc/nginx/sites-available/nginx-ssl.conf`)
* **Comando para mostrar:**
  ```bash
  cat /etc/nginx/sites-available/nginx-ssl.conf
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Para resolver el requerimiento de Nginx seguro, configuramos una directiva `listen 443 ssl http2`, habilitando protocolos modernos `TLSv1.2` y `TLSv1.3` junto con cifrado de alta seguridad. Adicionalmente, se configuró un bloque en el puerto 80 que realiza una redirección 301 permanente hacia HTTPS. Nginx gestiona el canal seguro de forma nativa en su arquitectura asíncrona no bloqueante sin requerir módulos externos como ocurre en Apache."*

---

### 2.3 Batería de Pruebas y Validación en `servidor2`

1. **Validar sintaxis del archivo de configuración:**
   ```bash
   sudo nginx -t
   ```
   * **Resultado esperado:** `syntax is ok` y `test is successful`.

2. **Verificar estado del servicio Nginx:**
   ```bash
   sudo systemctl status nginx --no-pager
   ```
   * **Resultado esperado:** Estado en verde `active (running)`.

3. **Comprobar puerto 443 en escucha en Nginx:**
   ```bash
   sudo ss -tlnp | grep :443
   ```
   * **Resultado esperado:** Socket en estado `LISTEN` en `0.0.0.0:443` perteneciente al proceso `nginx`.

4. **Prueba local HTTPS contra Nginx:**
   ```bash
   curl -kIv https://localhost
   ```
   * **Resultado esperado:** `HTTP/1.1 200 OK` (o HTTP/2) con cabecera `Server: nginx/...`.

---

### 2.4 Demostración Remota desde el Host Anfitrión (Windows)

1. **Petición HTTPS hacia Nginx (`servidor2`) con cURL:**
   ```powershell
   curl.exe -k -i https://192.168.50.2
   ```
   * **Resultado esperado:** Cabecera `HTTP/2 200` (o `HTTP/1.1 200 OK`) con cabecera `server: nginx/1.18.0` y cuerpo HTML correspondiente a la página segura de Nginx.

2. **Visualización y comprobación en Navegador Web:**
   * Abrir `https://192.168.50.2` en el navegador.
   * **Resultado esperado:** Despliegue de la página *"Servidor Web Nginx Seguro"* bajo canal HTTPS, validando el cumplimiento del Ejercicio 2.

---

## Preguntas Frecuentes del Evaluador (Cheat Sheet)

* **¿Por qué el navegador web muestra una advertencia de seguridad si el certificado es técnicamente válido?**
  * *Respuesta:* Porque es un certificado **autofirmado** (*Self-Signed*). El navegador confía únicamente en certificados cuya firma digital pertenezca a una Autoridad Certificadora (CA) preinstalada en su almacén raíz de confianza (como DigiCert, Let's Encrypt o Sectigo). Criptográficamente el túnel está cifrado y seguro, pero falta la validación de identidad por un tercero confiable.

* **¿Qué significa el parámetro `-nodes` en el comando `openssl req`?**
  * *Respuesta:* Significa *"No DES"* (sin cifrado de contraseña). Le indica a OpenSSL que no proteja la clave privada con una frase de contraseña (*passphrase*). Es fundamental para servidores de producción donde el servicio (Apache/Nginx) debe reiniciar de forma desatendida; de lo contrario, el arranque del sistema se detendría pidiendo la clave por teclado.

* **¿Cuál es la diferencia entre cifrado simétrico y asimétrico en una sesión SSL/TLS?**
  * *Respuesta:* En el **apretón de manos (Handshake)** se utiliza **cifrado asimétrico** (clave pública RSA y clave privada) para autenticar al servidor e intercambiar de forma segura una clave efímera de sesión. Una vez acordada dicha clave, la transferencia masiva de datos utiliza **cifrado simétrico** (como AES-GCM o ChaCha20) porque es varios órdenes de magnitud más rápido en CPU.

* **¿Qué función cumple la directiva `SSLEngine on` en Apache?**
  * *Respuesta:* Activa el motor de cifrado SSL/TLS dentro de un bloque `<VirtualHost>`. Sin esta directiva, Apache intentaría procesar las solicitudes en texto claro HTTP sobre el puerto 443, provocando errores de protocolo en el cliente.

* **¿Cuál es la diferencia operativa entre habilitar SSL en Apache vs en Nginx?**
  * *Respuesta:* Apache requiere un módulo compilado dinámico (`mod_ssl`) habilitado mediante `a2enmod ssl` y directivas como `SSLCertificateFile` y `SSLCertificateKeyFile`. Nginx incorpora el soporte SSL/TLS de manera nativa mediante la directiva `listen 443 ssl` y las directivas `ssl_certificate` y `ssl_certificate_key`, resultando en una configuración más limpia y de menor overhead.
