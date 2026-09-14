# INFORME TÉCNICO: CONFIGURACIÓN DE NGINX, HOST VIRTUAL Y RESOLUCIÓN DNS

**Asignatura:** Ambiente de Desarrollo / Servicios Telemáticos
**Profesor:** Prof. Oscar Mondragón
**Semestre:** 9no Semestre
**Repositorio:** [Practica_AmbienteDesarrollo](https://github.com/CriolloYule/Practica_AmbienteDesarrollo)
**Directorio de la Práctica:** `mipracticas/Practica4`

---

## 1. Resumen Ejecutivo y Objetivos

### 1.1 Resumen Ejecutivo

El presente informe documenta de manera técnica, práctica y paso a paso la implementación, administración y puesta en marcha de un servidor web de alto rendimiento **Nginx** con la habilitación de un **Host Virtual (Server Block)** para el dominio corporativo `www.ejercicio.com`, integrando la resolución de nombres autoritativa mediante el servicio DNS **BIND9**.

La infraestructura se ejecuta en un ambiente multinodo completamente virtualizado sobre **Ubuntu Server 22.04 LTS** aprovisionado con **Vagrant** y **VirtualBox** en una red privada dedicada (`192.168.50.0/24`), garantizando la separación funcional entre el rol de servidor web (`servidor2`) y el rol de servidor de resolución de nombres (`servidor`).

### 1.2 Objetivos del Laboratorio

1. **Aprovisionar la Infraestructura Multinodo:** Desplegar dos máquinas virtuales Ubuntu Server 22.04 LTS conectadas en una red interna privada fija mediante un `Vagrantfile` dedicado.
2. **Instalación y Gestión de Nginx:** Instalar, asegurar y validar el funcionamiento del demonio Nginx en el nodo `servidor2` (`192.168.50.2`).
3. **Implementación de Virtual Host:** Configurar un bloque de servidor (*server block*) en `/etc/nginx/sites-available/` para el dominio `www.ejercicio.com`, habilitando una raíz de documentos personalizada en `/var/www/ejercicio.com/html` con su respectiva página de inicio.
4. **Configuración de Zona DNS Autoritativa en BIND9:** Definir la zona directa `ejercicio.com` en `servidor` (`192.168.50.3`), creando los registros SOA, NS y A correspondientes para enlazar el dominio `www.ejercicio.com` con la IP del servidor web.
5. **Comprobación y Validación End-to-End:** Realizar pruebas de conectividad de red, resolución de nombres (`dig`, `nslookup`) e inspección de respuestas HTTP (`curl`, navegador web del Host) desde múltiples extremos de la topología.

---

## 2. Arquitectura de Infraestructura y Topología de Red

La práctica se implementa sobre un entorno multinodo aislado con direccionamiento estático bajo el segmento `192.168.50.0/24`:

```
+-----------------------------------------------------------------------------------+
|                              WINDOWS ANFITRIÓN (HOST)                             |
|             IP Interfaz VirtualBox Host-Only: 192.168.50.1                        |
|             Pruebas: Navegador Web / cURL / nslookup                              |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          | Red Privada Host-Only (192.168.50.0/24)
                                          v
+-----------------------------------------+-----------------------------------------+
|                  ENTORNO DE VIRTUALIZACIÓN VAGRANT / VIRTUALBOX                   |
|                                                                                   |
|   +------------------------------------+     +--------------------------------+   |
|   |          NODO: SERVIDOR            |     |         NODO: SERVIDOR2        |   |
|   | Hostname: servidor                 |     | Hostname: servidor2            |   |
|   | IP Privada: 192.168.50.3           | DNS | IP Privada: 192.168.50.2       |   |
|   | Servicio: BIND9 (Puerto 53 UDP/TCP)|<--->| Servicio: Nginx (Puerto 80 TCP)|   |
|   | Rol: Servidor DNS Autoritativo     |     | Rol: Servidor Web / VHost      |   |
|   | Zona: ejercicio.com                |     | Dominio: www.ejercicio.com     |   |
|   +------------------------------------+     +--------------------------------+   |
+-----------------------------------------------------------------------------------+
```

---

## 3. Desarrollo Detallado de los Puntos del Taller y Evidencias Visuales

---

### Punto 1: Instalación y Verificación de Nginx en "servidor2"

Para atender peticiones HTTP de forma eficiente, se despliega el servidor web Nginx en la máquina virtual `servidor2` (`192.168.50.2`).

#### Comandos Ejecutados

```bash
# Conexión SSH al nodo servidor2
vagrant ssh servidor2

# Actualización de repositorios e instalación
sudo apt update && sudo apt install -y nginx

# Verificación del estado del servicio
sudo systemctl status nginx

# Comprobación de sockets en escucha en el puerto 80
sudo ss -tlnp | grep :80
```

#### Evidencia 01: Instalación del Paquete Nginx

* **Ruta de Evidencia:** `images/01_nginx_instalacion.png`

![01_nginx_instalacion.png](images/01_nginx_instalacion.png)

* **Análisis Técnico:**
  La ejecución de `sudo apt install -y nginx` gestiona las dependencias necesarias de las librerías `nginx-core`, `nginx-common` y paquetes de soporte SSL. El servicio queda configurado para arranque automático con `systemd`.

#### Evidencia 02: Estado Activo y Puerto en Escucha

* **Ruta de Evidencia:** `images/02_nginx_status_activo.png`

![02_nginx_status_activo.png](images/02_nginx_status_activo.png)

* **Análisis Técnico:**
  El demonio se reporta en estado `active (running)` bajo el proceso principal de Nginx. El comando `ss -tlnp` confirma un socket TCP escuchando en `0.0.0.0:80` (IPv4) y `[::]:80` (IPv6), listo para recibir tráfico web entrante.

---

### Punto 2: Configuración de Host Virtual (Server Block) y Página Web Personalizada

En Nginx, la directiva `server_name` dentro de un bloque `server` permite discriminar dominios virtuales sobre una misma IP. Se crea el Virtual Host para responder al dominio `www.ejercicio.com` con contenido exclusivo.

#### Procedimiento Técnico

```bash
# 1. Creación de la estructura del Document Root
sudo mkdir -p /var/www/ejercicio.com/html
sudo chown -R $USER:$USER /var/www/ejercicio.com/html
sudo chmod -R 755 /var/www/ejercicio.com

# 2. Creación del archivo index.html personalizado
cat << 'EOF' > /var/www/ejercicio.com/html/index.html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>www.ejercicio.com | Nginx Virtual Host</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
            color: #ffffff;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }
        .card {
            background: rgba(255, 255, 255, 0.12);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.2);
            padding: 2.5rem;
            border-radius: 16px;
            box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37);
            text-align: center;
            max-width: 600px;
        }
        h1 { font-size: 2.2rem; margin-bottom: 0.5rem; color: #00d2ff; }
        p { font-size: 1.1rem; line-height: 1.6; margin: 0.5rem 0; }
        .badge {
            display: inline-block;
            background: #00d2ff;
            color: #003366;
            padding: 0.3rem 0.8rem;
            border-radius: 20px;
            font-weight: bold;
            margin-top: 1rem;
        }
    </style>
</head>
<body>
    <div class="card">
        <h1>🌐 www.ejercicio.com</h1>
        <p><strong>Servicio:</strong> Servidor Web Nginx con Host Virtual</p>
        <p><strong>Nodo:</strong> servidor2 (<code>192.168.50.2</code>)</p>
        <p><strong>Resolución DNS:</strong> Servidor BIND9 (<code>192.168.50.3</code>)</p>
        <div class="badge">Práctica 4 — Servicios Telemáticos</div>
        <p style="margin-top: 1.5rem; font-size: 0.9rem; opacity: 0.85;">
            Eduard Criollo Yule — Universidad Autónoma de Occidente
        </p>
    </div>
</body>
</html>
EOF

# 3. Creación del archivo de configuración del Host Virtual
sudo tee /etc/nginx/sites-available/ejercicio.com << 'EOF'
server {
    listen 80;
    listen [::]:80;

    root /var/www/ejercicio.com/html;
    index index.html index.htm;

    server_name www.ejercicio.com ejercicio.com;

    access_log /var/log/nginx/ejercicio.com.access.log;
    error_log /var/log/nginx/ejercicio.com.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

# 4. Habilitación del sitio mediante enlace simbólico
sudo ln -s /etc/nginx/sites-available/ejercicio.com /etc/nginx/sites-enabled/

# 5. Validación de sintaxis y recarga de Nginx
sudo nginx -t
sudo systemctl reload nginx
```

#### Evidencia 03: Creación del Document Root e Index HTML

* **Ruta de Evidencia:** `images/03_crear_directorio_sitio.png`

![03_crear_directorio_sitio.png](images/03_crear_directorio_sitio.png)

* **Análisis Técnico:**
  Se establece el directorio de publicación en `/var/www/ejercicio.com/html`. Los permisos `755` y la propiedad asignada garantizan que el usuario del proceso de Nginx (`www-data`) disponga de permisos de lectura y ejecución en el árbol de rutas sin comprometer la seguridad.

#### Evidencia 04: Configuración del Bloque de Servidor (Server Block)

* **Ruta de Evidencia:** `images/04_vhost_config_nginx.png`

![04_vhost_config_nginx.png](images/04_vhost_config_nginx.png)

* **Análisis Técnico:**
  La directiva `server_name www.ejercicio.com ejercicio.com;` instruye a Nginx a comparar la cabecera HTTP `Host` enviada por el navegador con este bloque. La directiva `try_files $uri $uri/ =404;` asegura que cualquier recurso inexistente retorne un código de estado HTTP 404 estandarizado.

#### Evidencia 05: Habilitación de Sitio y Validación con `nginx -t`

* **Ruta de Evidencia:** `images/05_habilitar_sitio_symlink.png`

![05_habilitar_sitio_symlink.png](images/05_habilitar_sitio_symlink.png)

* **Análisis Técnico:**
  En distribuciones Debian/Ubuntu, los sitios activos deben vincularse en `/etc/nginx/sites-enabled/`. El comando `sudo nginx -t` ejecuta un análisis sintáctico integral de los archivos de configuración incluidos en `nginx.conf`, retornando `syntax is ok` y `test is successful` antes de aplicar la recarga con `systemctl reload`.

---

### Punto 3: Configuración del Servidor DNS (BIND9) en "servidor"

Para resolver el nombre `www.ejercicio.com` hacia la dirección IP de `servidor2` (`192.168.50.2`), se configura el servicio BIND9 en la máquina virtual `servidor` (`192.168.50.3`).

#### Procedimiento Técnico

```bash
# Conexión SSH al nodo servidor
vagrant ssh servidor

# Instalación de paquetes de BIND9 (si no se encontraban previamente aprovisionados)
sudo apt update && sudo apt install -y bind9 bind9utils bind9-doc

# 1. Declaración de la zona en /etc/bind/named.conf.local
sudo tee -a /etc/bind/named.conf.local << 'EOF'

zone "ejercicio.com" {
    type master;
    file "/etc/bind/db.ejercicio.com";
};
EOF

# 2. Creación del archivo de base de datos de la zona directa
sudo tee /etc/bind/db.ejercicio.com << 'EOF'
;
; Archivo de Zona DNS para ejercicio.com
;
$TTL    604800
@       IN      SOA     servidor.ejercicio.com. admin.ejercicio.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; Servidores de Nombres (NS)
@       IN      NS      servidor.ejercicio.com.

; Direcciones IP de los Servidores de Nombres (Glue Records)
servidor IN     A       192.168.50.3

; Registros A para el Host Virtual y el dominio raíz
www     IN      A       192.168.50.2
@       IN      A       192.168.50.2
EOF

# 3. Comprobación de sintaxis de configuración y de zona
sudo named-checkconf
sudo named-checkzone ejercicio.com /etc/bind/db.ejercicio.com

# 4. Reinicio y verificación del servicio
sudo systemctl restart bind9
sudo systemctl status bind9
```

#### Evidencia 06: Declaración de Zona en `named.conf.local`

* **Ruta de Evidencia:** `images/06_bind9_named_conf_local.png`

![06_bind9_named_conf_local.png](images/06_bind9_named_conf_local.png)

* **Análisis Técnico:**
  Se declara la zona `ejercicio.com` bajo el rol `type master;`, definiendo a este nodo como la fuente autoritativa primaria de la información de resolución.

#### Evidencia 07: Archivo de Registros de Zona `db.ejercicio.com`

* **Ruta de Evidencia:** `images/07_zona_dns_ejercicio_com.png`

![07_zona_dns_ejercicio_com.png](images/07_zona_dns_ejercicio_com.png)

* **Análisis Técnico:**
  El registro SOA inicializa los parámetros de réplica y expiración. Los registros tipo `A` asignan con precisión el nombre canónico `www.ejercicio.com` hacia la dirección IPv4 privada `192.168.50.2`, correspondiente a la máquina donde se ejecuta Nginx.

#### Evidencia 08: Verificación con `named-checkzone` y Servicio Activo

* **Ruta de Evidencia:** `images/08_named_checkzone_ok.png`

![08_named_checkzone_ok.png](images/08_named_checkzone_ok.png)

* **Análisis Técnico:**
  `named-checkzone` valida la coherencia sintáctica, números de serie y registros obligatorios, confirmando `OK`. El demonio `named` queda activo en el puerto estándar 53 UDP/TCP.

#### Evidencia 09: Resolución de Nombres desde `servidor2`

* **Ruta de Evidencia:** `images/09_nslookup_resolucion.png`

![09_nslookup_resolucion.png](images/09_nslookup_resolucion.png)

* **Análisis Técnico:**
  Al ejecutar `dig @192.168.50.3 www.ejercicio.com +short` o `nslookup www.ejercicio.com 192.168.50.3` desde `servidor2`, la consulta responde de forma directa con la IP esperada: `192.168.50.2`.

---

### Punto 4: Verificación y Pruebas desde la Máquina Anfitriona (Host)

Se valida el flujo completo de servicio desde la máquina anfitriona Windows, garantizando que un cliente real pueda resolver el dominio y visualizar la página personalizada.

#### Procedimiento Técnico en Host Windows (PowerShell)

```powershell
# 1. Consulta DNS dirigida al servidor BIND9
nslookup www.ejercicio.com 192.168.50.3

# 2. Configuración en archivo hosts de Windows (si no se cambia el DNS global de la interfaz)
# Ejecutar Bloc de notas como Administrador y agregar en C:\Windows\System32\drivers\etc\hosts:
# 192.168.50.2    www.ejercicio.com    ejercicio.com

# 3. Prueba de respuesta HTTP mediante cURL
curl.exe -i http://www.ejercicio.com

# 4. Despliegue en navegador web
Start-Process "http://www.ejercicio.com"
```

#### Evidencia 10: Resolución DNS desde Host Windows

* **Ruta de Evidencia:** `images/10_host_nslookup.png`

![10_host_nslookup.png](images/10_host_nslookup.png)

* **Análisis Técnico:**
  La utilidad `nslookup` ejecutada en Windows consulta directamente el socket `192.168.50.3:53` y retorna una respuesta no autoritativa/autoritativa válida vinculando `www.ejercicio.com` a la dirección `192.168.50.2`.

#### Evidencia 11: Petición HTTP con Cabeceras vía cURL

* **Ruta de Evidencia:** `images/11_curl_respuesta_html.png`

![11_curl_respuesta_html.png](images/11_curl_respuesta_html.png)

* **Análisis Técnico:**
  La respuesta HTTP retornada por Nginx muestra `HTTP/1.1 200 OK`, `Server: nginx/1.18.0 (Ubuntu)` y el payload HTML con el diseño personalizado, demostrando que el Virtual Host identificó satisfactoriamente el encabezado `Host: www.ejercicio.com`.

#### Evidencia 12: Despliegue Visual en Navegador Web del Host

* **Ruta de Evidencia:** `images/12_navegador_pagina_inicio.png`

![12_navegador_pagina_inicio.png](images/12_navegador_pagina_inicio.png)

* **Análisis Técnico:**
  El navegador web (Chrome/Edge/Firefox) renderiza correctamente el sitio web corporativo bajo la URL `http://www.ejercicio.com`, completando de extremo a extremo el ciclo de resolución de nombres, enrutamiento en red local y despacho del servidor web.

---

## 4. Tabla Resumen de Evidencias e Imágenes

|   Número   | Archivo Renombrado                 | Descripción de la Evidencia                                                      | Punto del Taller |
| :----------: | :--------------------------------- | :-------------------------------------------------------------------------------- | :--------------: |
| **01** | `01_nginx_instalacion.png`       | Instalación del paquete Nginx en`servidor2` mediante APT                       |     Punto 1     |
| **02** | `02_nginx_status_activo.png`     | Estado`active (running)` del servicio y puertos de escucha con `ss`           |     Punto 1     |
| **03** | `03_crear_directorio_sitio.png`  | Creación de Document Root y permisos en`/var/www/ejercicio.com/html`           |     Punto 2     |
| **04** | `04_vhost_config_nginx.png`      | Configuración del Virtual Host en`/etc/nginx/sites-available/ejercicio.com`    |     Punto 2     |
| **05** | `05_habilitar_sitio_symlink.png` | Creación de enlace simbólico en`sites-enabled` y validación con `nginx -t` |     Punto 2     |
| **06** | `06_bind9_named_conf_local.png`  | Definición de zona`ejercicio.com` en `/etc/bind/named.conf.local`            |     Punto 3     |
| **07** | `07_zona_dns_ejercicio_com.png`  | Archivo de base de datos de zona`/etc/bind/db.ejercicio.com` con registros A    |     Punto 3     |
| **08** | `08_named_checkzone_ok.png`      | Comprobación de sintaxis con`named-checkzone` y estado activo de BIND9         |     Punto 3     |
| **09** | `09_nslookup_resolucion.png`     | Comprobación de resolución DNS desde`servidor2` hacia `servidor`            |     Punto 3     |
| **10** | `10_host_nslookup.png`           | Consulta DNS con`nslookup` desde la consola del Host Windows                    |     Punto 4     |
| **11** | `11_curl_respuesta_html.png`     | Petición HTTP e inspección de cabeceras`200 OK` con cURL                      |     Punto 4     |
| **12** | `12_navegador_pagina_inicio.png` | Visualización en navegador del Host cargando`http://www.ejercicio.com`         |     Punto 4     |

---

## 5. Conclusiones Técnicas

1. **Eficiencia y Modularidad en Nginx:** La arquitectura de Server Blocks de Nginx permite alojar múltiples identidades web independientes bajo una misma instancia física o virtual. La separación entre `sites-available` y `sites-enabled` proporciona un control operacional limpio y seguro para habilitar o inhabilitar sitios sin destruir su configuración.
2. **Rol Crucial de la Cabecera `Host` en HTTP/1.1:** El funcionamiento de los Virtual Hosts basados en nombre depende enteramente del campo de cabecera `Host` enviado por el cliente en la petición HTTP. Nginx evalúa este valor contra la directiva `server_name` para dirigir la solicitud al bloque correspondiente; en caso de no coincidir, despacha el bloque por defecto (*default_server*).
3. **Interdependencia entre Servicios Web y DNS:** Un despliegue web profesional requiere una infraestructura de nombres sólida. La integración con BIND9 demostró cómo un registro tipo `A` enlaza el nombre comprensible para el usuario (`www.ejercicio.com`) con la IP destino (`192.168.50.2`), eliminando la necesidad de recordar direcciones numéricas.
4. **Reproducibilidad y Aislamiento con Vagrant:** La definición de un `Vagrantfile` multinodo dedicado asegura la reproducibilidad inmediata del laboratorio en cualquier equipo de cómputo, garantizando el aislamiento de red, IP estáticas consistentes y un ciclo ágil de pruebas DevOps.
