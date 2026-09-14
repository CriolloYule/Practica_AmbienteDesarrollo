# Guía de Sustentación Rápida — Práctica 4: Nginx, Virtual Host y DNS

Este documento sirve como apoyo directo para la sustentación del laboratorio ante el docente o evaluador. Contiene la explicación concisa de la arquitectura, los archivos clave a mostrar con sus comandos limpios de terminal, el guion técnico y la batería de pruebas de verificación en vivo.

---

## 1. Resumen de Arquitectura y Roles

| Nodo | IP Privada | Servicio Principal | Rol Técnico |
| :--- | :--- | :--- | :--- |
| **servidor** | `192.168.50.3` | **BIND9** (DNS) | Servidor DNS autoritativo maestro para la zona `ejercicio.com`. Resuelve `www.ejercicio.com` hacia `192.168.50.2`. |
| **servidor2** | `192.168.50.2` | **Nginx** (HTTP) | Servidor Web que aloja el Host Virtual (Server Block) con la página web personalizada. |
| **Host (Windows)** | `192.168.50.1` | Cliente / Navegador | Realiza consultas DNS y peticiones HTTP para comprobar el funcionamiento end-to-end. |

---

## 2. Archivos Clave a Mostrar y Guion Técnico

Usa estos comandos en la terminal durante la sustentación para proyectar de forma limpia cada configuración y usa el guion sugerido para justificar su propósito.

### Archivo 1: Configuración del Virtual Host en Nginx
* **Ubicación:** `servidor2` (`/etc/nginx/sites-available/ejercicio.com`)
* **Comando para mostrar:**
  ```bash
  cat /etc/nginx/sites-available/ejercicio.com
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Este archivo define el bloque de servidor en Nginx que escucha en el puerto 80. Con la directiva `server_name www.ejercicio.com ejercicio.com`, Nginx examina la cabecera HTTP `Host` que envía el navegador y le sirve el contenido ubicado exclusivamente en `/var/www/ejercicio.com/html`. Además, `try_files` gestiona adecuadamente las peticiones retornando un error 404 si el archivo solicitado no existe."*

---

### Archivo 2: Enlace Simbólico de Activación del Sitio
* **Ubicación:** `servidor2` (`/etc/nginx/sites-enabled/`)
* **Comando para mostrar:**
  ```bash
  ls -la /etc/nginx/sites-enabled/
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Siguiendo las mejores prácticas de administración en distribuciones Debian y Ubuntu, los sitios web no se configuran directamente en `sites-enabled`, sino que se crea un enlace simbólico desde `sites-available`. Esto permite activar o suspender sitios web al instante sin eliminar ni alterar los archivos fuente."*

---

### Archivo 3: Página Web de Inicio Personalizada
* **Ubicación:** `servidor2` (`/var/www/ejercicio.com/html/index.html`)
* **Comando para mostrar:**
  ```bash
  head -n 25 /var/www/ejercicio.com/html/index.html
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Es el documento HTML raíz (`index.html`) que contiene la estructura y estilos CSS integrados. Cumple con el requerimiento de desplegar una página personalizada que identifica el servicio, la materia y los datos del estudiante."*

---

### Archivo 4: Declaración de Zona DNS en BIND9
* **Ubicación:** `servidor` (`/etc/bind/named.conf.local`)
* **Comando para mostrar:**
  ```bash
  cat /etc/bind/named.conf.local
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"En este archivo registramos la zona directa `ejercicio.com` con `type master;`, estableciendo que este servidor tiene la autoridad principal sobre los registros del dominio y enlazándolo a su archivo de base de datos en `/etc/bind/db.ejercicio.com`."*

---

### Archivo 5: Registros de Zona DNS (SOA, NS y A)
* **Ubicación:** `servidor` (`/etc/bind/db.ejercicio.com`)
* **Comando para mostrar:**
  ```bash
  cat /etc/bind/db.ejercicio.com
  ```
* **Guion Técnico (¿Qué decir?):**
  > *"Aquí se definen el registro SOA de autoridad y el registro NS. Los registros tipo `A` asignan con exactitud el nombre `www` y el dominio raíz `@` hacia la dirección IP `192.168.50.2`, correspondiente a la máquina `servidor2` donde se encuentra ejecutándose Nginx."*

---

## 3. Batería de Pruebas de Verificación en Vivo

Ejecuta estas pruebas en orden frente al evaluador para validar que todos los objetivos del taller están cumplidos.

### Bloque A: Pruebas en `servidor2` (Servidor Web Nginx)

1. **Verificar estado del servicio Nginx:**
   ```bash
   sudo systemctl status nginx --no-pager
   ```
   * **Resultado esperado:** Estado en verde `active (running)`.

2. **Comprobar puerto 80 en escucha:**
   ```bash
   sudo ss -tlnp | grep :80
   ```
   * **Resultado esperado:** Sockets en estado `LISTEN` en `0.0.0.0:80` y `[::]:80` pertenecientes al proceso `nginx`.

3. **Validar sintaxis de configuración de Nginx:**
   ```bash
   sudo nginx -t
   ```
   * **Resultado esperado:** `nginx: the configuration file /etc/nginx/nginx.conf syntax is ok` y `test is successful`.

4. **Prueba local HTTP por nombre de dominio:**
   ```bash
   curl -I -H "Host: www.ejercicio.com" http://localhost
   ```
   * **Resultado esperado:** `HTTP/1.1 200 OK` y `Server: nginx/...`.

---

### Bloque B: Pruebas en `servidor` (Servidor DNS BIND9)

1. **Verificar estado del servicio BIND9:**
   ```bash
   sudo systemctl status bind9 --no-pager
   ```
   * **Resultado esperado:** Estado en verde `active (running)`.

2. **Validar sintaxis de la zona DNS:**
   ```bash
   sudo named-checkzone ejercicio.com /etc/bind/db.ejercicio.com
   ```
   * **Resultado esperado:** `zone ejercicio.com/IN: loaded serial 3` y `OK`.

3. **Consulta de resolución local con `dig`:**
   ```bash
   dig @127.0.0.1 www.ejercicio.com +short
   ```
   * **Resultado esperado:** Retorna directamente la IP: `192.168.50.2`.

---

### Bloque C: Pruebas desde el Host Anfitrión (Windows)

1. **Comprobar resolución DNS hacia el servidor virtual:**
   ```powershell
   nslookup www.ejercicio.com 192.168.50.3
   ```
   * **Resultado esperado:** Retorna `Name: www.ejercicio.com` con `Address: 192.168.50.2`.

2. **Petición HTTP con cURL:**
   ```powershell
   curl.exe -i http://www.ejercicio.com
   ```
   * **Resultado esperado:** Cabecera `HTTP/1.1 200 OK` y el cuerpo HTML de la página web personalizada.

3. **Visualización en Navegador Web:**
   * Abrir `http://www.ejercicio.com` en Chrome, Edge o Firefox.
   * **Resultado esperado:** Carga fluida de la tarjeta estilizada con el título **www.ejercicio.com**, demostrando la integración completa de DNS, Nginx y Virtual Host.

---

## 4. Preguntas Frecuentes del Evaluador (Cheat Sheet)

* **¿Por qué usar `sites-available` y `sites-enabled`?**
  * *Respuesta:* Permite modularidad. En `sites-available` se guardan todas las plantillas y configuraciones posibles, y solo las que tienen enlace simbólico en `sites-enabled` son cargadas por el demonio.
* **¿Qué sucede si un cliente entra por IP en lugar de dominio?**
  * *Respuesta:* Nginx responde con el servidor por defecto (`default_server`). Para que cargue `www.ejercicio.com`, el cliente debe enviar la cabecera `Host: www.ejercicio.com`.
* **¿Qué función cumple el registro SOA en DNS?**
  * *Respuesta:* El registro *Start of Authority* define los parámetros operativos globales de la zona: el servidor maestro, el correo del administrador, el número de serie para control de versiones y los tiempos de refresco/expiración en caché.
