# ST0263 - Tópicos Especiales de Telemática

## Integrantes:
•	Simon Betancur Bedoya
•	Esteban Sierra Patiño
•	Daniel Cano Perez
•	Miguel Angel Cabrera Osorio

## Profesor
- **Nombre:** Edwin Nelson Montoya Munera

# Reto 2

## 1. Breve descripción de la actividad

En este proyecto, implementaremos una aplicación centralizada utilizando Moodle en un entorno Docker, centrándonos en garantizar alta disponibilidad y capacidad de ampliación. Nuestro enfoque implica el uso de Nginx como un equilibrador de carga para dirigir el tráfico entre dos instancias de Moodle, mejorando así el rendimiento y la confiabilidad del sistema. Además, la arquitectura contempla instancias separadas para la base de datos MySQL y el almacenamiento de archivos mediante NFS, todos alojados en AWS con cinco máquinas virtuales diseñadas para garantizar un funcionamiento continuo y eficiente. También se incluye una instancia adicional para acceder a las instancias privadas (Moodle, base de datos). 


### 1.1. Que aspectos cumplió o desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)

Aspectos cumplidos de la actividad propuesta por el profesor:

•	Implementar un balanceador de cargas basado en nginx que reciba el tráfico web https de Internet con múltiples instancias de procesamiento.
•	Tener 2 instancias de procesamiento Moodle detrás del balanceador de cargas.
•	Tener 1 instancia de bases de datos mysql.
•	Tener 1 instancia de archivos distribuidos en NFS.
•	Arquitectura propuesta en el documento guía implementada.
•	Certificado SSL.

### 1.2. Que aspectos NO cumplió o desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)

•	Dockerización de Moodle en AWS debido a un problema al momento de desplegar en el cual la instancia iniciaba correctamente y luego se detiene sin motivo aparente.
• Al desplegar el reto 2 parte 1 a un cluster en EKR tuvimos problemas con el acceso al cluster desde los kubernetes (Evidencias en el video)


## 2. Información general de diseño de alto nivel, arquitectura, patrones, mejores prácticas utilizadas.
Se utilizó la arquitectura presentada a continuación:

![image](https://github.com/user-attachments/assets/48a44612-bf35-4ecb-ad13-e56818259df4)

## 3. Descripción del ambiente de desarrollo y técnico: lenguaje de programación, librerias, paquetes, etc, con sus numeros de versiones.

Se utilizó Docker, con sus respectivos docker-compose para contenerizar correctamente las aplicaciones y servicios necesarios, puede encontrar estos archivos en la carpeta docker de este repositorio.

## 4. Descripción del ambiente de EJECUCIÓN (en producción) lenguaje de programación, librerias, paquetes, etc, con sus numeros de versiones.

## IP o nombres de dominio en nube o en la máquina servidor.

Para acceder a la página web: reto2moodle.hopto.org

* **Nginx:** IP privada: 172.31.11.53

* **Server 1:** IP privada: 172.31.2.205

* **Server 2:** IP privada: 172.31.8.57

* **MYSQL:** IP privada: 172.31.12.235

* **NFS:** IP privada: 172.31.13.233

![image](https://github.com/user-attachments/assets/a5771b18-34a2-4c6d-a28a-55e03cd3ae9d)

### Docker Compose Moodle
```yaml
version: '3.1'

services:
  moodle:
    image: bitnami/moodle:latest
    ports:
      - "80:80"
    volumes:
      - /mount/point:/bitnami/moodle
    environment:
      MOODLE_DATABASE_HOST: 172.31.12.235
      MOODLE_DATABASE_USER: moodle_user
      MOODLE_DATABASE_PASSWORD: strongpassword
      MOODLE_DATABASE_NAME: moodle_db
    networks:
      - vpc_network
networks:
  vpc_network:
    driver: bridge
```

### Docker Compose MySQL Moodle

```yaml
version: '3.1'

services:
  mysql:
    image: mysql:latest
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: moodle_db
      MYSQL_USER: moodle_user
      MYSQL_PASSWORD: strongpassword
    networks:
      - vpc_network
networks:
  vpc_network:
    driver: bridge
```

### Docker Compose Wordpress
```yaml
version: '3.1'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "80:80"
    volumes:
      - /mount/point:/var/www/html/wp-content/uploads
    environment:
      WORDPRESS_DB_HOST: 172.31.12.235
      WORDPRESS_DB_USER: wordpress_user
      WORDPRESS_DB_PASSWORD: strongpassword
      WORDPRESS_DB_NAME: wordpress_db
    networks:
      - vpc_network
networks:
  vpc_network:
    driver: bridge
```

### Docker Compose MySQL Wordpress

```yaml
version: '3.1'

services:
  mysql:
    image: mysql:latest
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress_db
      MYSQL_USER: wordpress_user
      MYSQL_PASSWORD: strongpassword
    networks:
      - vpc_network
networks:
  vpc_network:
    driver: bridge
```

## Configuración Nginx
```bash
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;

        upstream moodle_backend {
           ip_hash;
           server 172.31.2.205:80;
           server 172.31.8.57:80;
        }

        server {
            server_name reto2moodle.hopto.org;

            location / {
                 proxy_pass http://moodle_backend;
                 proxy_set_header Host $host;
                 proxy_set_header X-Real-IP $remote_addr;
                 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header X-Forwarded-Proto $scheme;
            }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/reto2moodle.hopto.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/reto2moodle.hopto.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}



        server {
    if ($host = reto2moodle.hopto.org) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


            listen 80;
            server_name reto2moodle.hopto.org;
    return 404; # managed by Certbot


}}


#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
```
