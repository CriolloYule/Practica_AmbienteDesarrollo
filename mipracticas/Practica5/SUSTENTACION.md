# Guía de Sustentación Rápida — Práctica 5: Servidores Web Seguros con Apache y Nginx (SSL/TLS)

Este documento sirve como instrumento directo de apoyo para la sustentación del laboratorio técnico ante el docente o evaluador. Contiene la explicación concisa de la topología y arquitectura, los archivos clave a proyectar con sus comandos limpios de terminal, el guion técnico justificativo y la batería completa de pruebas de verificación en tiempo real.

---

## 1. Resumen de Arquitectura y Roles

| Nodo | IP Privada | Servicio Principal | Puertos Escuchando | Rol Técnico y Criptográfico |
| :--- | :--- | :--- | :--- | :--- |
| **servidor** | `192.168.50.3` | **Apache2** (`mod_ssl`) | `80/TCP` (HTTP)<br>`443/TCP` (HTTPS) | Servidor Web seguro principal. Aloja el VirtualHost `servicios.com` con certificado autofirmado X.509 RSA de 2048 bits. |
| **servidor2** | `192.168.50.2` | **Nginx** (SSL/TLS) | `80/TCP` (HTTP)<br>`443/TCP` (HTTPS) | Servidor Web secundario (Ejercicio 2). Aloja el bloque de servidor SSL con terminación TLS y página segura independiente. |
| **Host (Windows)** | `192.168.50.1` | Cliente / Navegador | Dinámicos de salida | Realiza peticiones HTTPS, inspección de certificados X.509, pruebas con `curl` e inspección del canal seguro. |

> [!NOTE]
> **Modalidad Mono-nodo Alternativa:** Si el evaluador solicita ejecutar ambos servidores web en la misma máquina virtual (`servidor`), Apache conserva los puertos estándar `80/443`, mientras que Nginx se configura en el puerto seguro alternativo `8443/TCP`, evitando colisiones de socket (`Address already in use`).

---

## 2. Archivos Clave a Mostrar y Guion Técnico

Ejecuta estos comandos en la terminal durante la sustentación para proyectar de forma limpia cada configuración y emplea el guion sugerido para justificar las decisiones técnicas ante el evaluador.

### Archivo 1: Configuración del Virtual Host Apache SSL

* **Ubicación:** `servidor` (`/etc/apache2/sites-available/servicios.com.conf`)
* **Comando para mostrar limpiamente:**
  ```bash
  cat /etc/apache2/sites-available/servicios.com.conf
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"En este archivo definimos dos bloques de VirtualHost. El primero escucha en el puerto seguro `443` con `SSLEngine on`, asociando el certificado digital público en `/etc/ssl/certs/apache-selfsigned.crt` y la clave privada en `/etc/ssl/private/apache-selfsigned.key`. El segundo bloque atiende en el puerto `80` para tráfico estándar, garantizando la compatibilidad HTTP."*

---

### Archivo 2: Certificado Digital X.509 Autofirmado

* **Ubicación:** `servidor` (`/etc/ssl/certs/apache-selfsigned.crt`)
* **Comando para mostrar:**
  ```bash
  openssl x509 -in /etc/ssl/certs/apache-selfsigned.crt -noout -text | grep -E "(Issuer:|Subject:|Not Before|Not After|Public Key Algorithm|RSA Public-Key)" -A 1
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Aquí inspeccionamos el certificado digital bajo el estándar ITU-T X.509 generado con OpenSSL. Como se observa en la salida, el `Issuer` y el `Subject` son idénticos, confirmando que es un certificado autofirmado (Self-Signed) con algoritmo de clave pública RSA de 2048 bits y una vigencia exacta de 365 días."*

---

### Archivo 3: Permisos de Seguridad de la Clave Privada

* **Ubicación:** `servidor` (`/etc/ssl/private/apache-selfsigned.key`)
* **Comando para mostrar:**
  ```bash
  ls -la /etc/ssl/private/apache-selfsigned.key
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Por principio de menor privilegio y seguridad criptográfica, la clave privada generada con el flag `-nodes` tiene permisos estrictos `600` (`-rw-------`) perteneciendo exclusivamente al usuario `root`. Esto previene que usuarios no privilegiados del sistema operativo puedan extraer el material criptográfico con el que se descifra el tráfico de la sesión."*

---

### Archivo 4: Configuración del Bloque de Servidor SSL en Nginx (Ejercicio 2)

* **Ubicación:** `servidor2` (`/etc/nginx/sites-available/nginx-ssl.conf`)
* **Comando para mostrar:**
  ```bash
  cat /etc/nginx/sites-available/nginx-ssl.conf
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Para resolver el requerimiento de Nginx seguro, configuramos una directiva `listen 443 ssl`, habilitando protocolos modernos `TLSv1.2` y `TLSv1.3` junto con el cifrado de alta seguridad. Nginx gestiona el canal seguro de forma nativa en su arquitectura asíncrona no bloqueante sin requerir módulos externos como ocurre en Apache."*

---

### Archivo 5: Página Web de Inicio Segura

* **Ubicación:** `servidor` (`/var/www/html/index.html`)
* **Comando para mostrar:**
  ```bash
  head -n 25 /var/www/html/index.html
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Es el documento HTML raíz servido bajo cifrado SSL/TLS. Cuenta con estructura semántica y estilos CSS integrados para identificar visualmente que la conexión viaja autenticada e íntegra a través del túnel seguro de la práctica."*

---

## 3. Batería de Pruebas de Verificación en Vivo

Ejecuta estas pruebas en orden frente al profesor para comprobar el funcionamiento end-to-end de los servicios web seguros.

### Bloque A: Pruebas en `servidor` (Servidor Apache SSL)

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

3. **Verificar que el módulo `mod_ssl` está cargado:**
   ```bash
   sudo apache2ctl -M | grep ssl
   ```
   * **Resultado esperado:** `ssl_module (shared)`.

4. **Validar la sintaxis de configuración:**
   ```bash
   sudo apache2ctl configtest
   ```
   * **Resultado esperado:** `Syntax OK`.

5. **Prueba local HTTPS e inspección de cabeceras:**
   ```bash
   curl -kIv https://localhost
   ```
   * **Resultado esperado:** Handshake TLS exitoso, código `HTTP/1.1 200 OK`, cabecera `Server: Apache/...` y contenido HTML.

6. **Comprobación del Handshake Criptográfico con OpenSSL s_client:**
   ```bash
   echo | openssl s_client -connect localhost:443 -brief
   ```
   * **Resultado esperado:** `CONNECTION ESTABLISHED`, protocolo `TLSv1.3` (o `TLSv1.2`) y cipher suite segura (ej. `TLS_AES_256_GCM_SHA384`).

---

### Bloque B: Pruebas en `servidor2` (Servidor Nginx SSL - Ejercicio 2)

1. **Verificar estado del servicio Nginx:**
   ```bash
   sudo systemctl status nginx --no-pager
   ```
   * **Resultado esperado:** Estado en verde `active (running)`.

2. **Comprobar puerto 443 en escucha en Nginx:**
   ```bash
   sudo ss -tlnp | grep :443
   ```
   * **Resultado esperado:** Socket en estado `LISTEN` en `0.0.0.0:443` perteneciente al proceso `nginx`.

3. **Validar sintaxis del archivo de configuración:**
   ```bash
   sudo nginx -t
   ```
   * **Resultado esperado:** `syntax is ok` y `test is successful`.

4. **Prueba local HTTPS contra Nginx:**
   ```bash
   curl -kIv https://localhost
   ```
   * **Resultado esperado:** `HTTP/1.1 200 OK` con cabecera `Server: nginx/...`.

---

### Bloque C: Pruebas desde el Host Anfitrión (Windows)

1. **Comprobar accesibilidad de red en el puerto 443 de Apache:**
   ```powershell
   Test-NetConnection -ComputerName 192.168.50.3 -Port 443
   ```
   * **Resultado esperado:** `TcpTestSucceeded : True`.

2. **Petición HTTPS con cURL ignorando el certificado autofirmado (`-k`):**
   ```powershell
   curl.exe -k -i https://192.168.50.3
   ```
   * **Resultado esperado:** Cabecera `HTTP/1.1 200 OK` y el cuerpo HTML servido por Apache.

3. **Petición HTTPS hacia Nginx (`servidor2`):**
   ```powershell
   curl.exe -k -i https://192.168.50.2
   ```
   * **Resultado esperado:** Cabecera `HTTP/1.1 200 OK` servida por Nginx.

4. **Visualización y comprobación en Navegador Web:**
   * Abrir `https://192.168.50.3` en Chrome, Edge o Firefox.
   * **Resultado esperado:** Se despliega la advertencia esperada de certificado autofirmado (`NET::ERR_CERT_AUTHORITY_INVALID`). Al hacer clic en *"Avanzado $\rightarrow$ Continuar a 192.168.50.3"*, carga la página web con el candado rojo/gris. Al pulsar en el candado $\rightarrow$ *"El certificado no es válido"*, se observa la entidad emisora `server.servicios.com` generada en la práctica.

---

## 4. Preguntas Frecuentes del Evaluador (Cheat Sheet)

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
