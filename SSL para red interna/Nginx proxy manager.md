
```
cd icinga2/
```
```
vim docker-compose.yml 
```
```yaml
version: '2.2'
services:
  nginxproxymanager:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./nginx/data:/data
      - ./nginx/letsencrypt:/etc/letsencrypt
    networks:
      - nginx_default
  icinga2:
    build:
      context: ./
      dockerfile: Dockerfile
    restart: on-failure:5
    hostname: master.javier.lan
    env_file:
      - secrets_sql.env
    environment:
      - DEFAULT_MYSQL_HOST=10.0.2.253
    volumes:
      - ./data/icinga/cache:/var/cache/icinga2
      - ./data/icinga/certs:/etc/apache2/ssl
      - ./data/icinga/etc/icinga2:/etc/icinga2
      - ./data/icinga/etc/icingaweb2:/etc/icingaweb2
      - ./data/icinga/lib/icinga:/var/lib/icinga2
      - ./data/icinga/lib/php/sessions:/var/lib/php/sessions
      - ./data/icinga/log/apache2:/var/log/apache2
      - ./data/icinga/log/icinga2:/var/log/icinga2
      - ./data/icinga/log/icingaweb2:/var/log/icingaweb2
      - ./data/icinga/spool:/var/spool/icinga2
      - ./msmtp/msmtprc:/etc/msmtprc:ro
      - ./msmtp/aliases:/etc/aliases:ro
    ports:
      - "5665:5665"
    networks:
      - nginx_default


networks:
  nginx_default:
    external: true
```
```
docker-compose up -d
```
```
docker ps
```
```
CONTAINER ID   IMAGE                             COMMAND      CREATED       STATUS       PORTS                                            NAMES
af48d234e025   icinga2_icinga2                   "/opt/run"   4 hours ago   Up 3 hours   80/tcp, 443/tcp, 0.0.0.0:5665->5665/tcp          icinga2_icinga2_1   
ffec170462f8   jc21/nginx-proxy-manager:latest   "/init"      4 hours ago   Up 3 hours   0.0.0.0:80-81->80-81/tcp, 0.0.0.0:443->443/tcp   icinga2_nginxproxyma
nager_1
```
Abrimos el navegador y accedemos a nginx a través del puerto 81: `http://192.168.1.166:81`.
![[Pasted image 20231208185001.png]]
- El usuario por defecto es `admin@example.com` con su contraseña `changeme`. 
Una vez autenticados nos pedirá que cambiemos los credenciales del administrador. En mi caso la dirección de email ser
