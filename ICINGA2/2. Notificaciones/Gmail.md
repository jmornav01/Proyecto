- [[#Acciones previas]]
- [[#SMTP]]
- [[#MSMTP]]
- [[#Configuración en Icinga2]]
- [[#Ajustes útiles]]

## Acciones previas
Para enviar correos por Gmail, tendremos que irnos a los ajustes de seguridad de la cuenta que utilizaremos para enviar los correos a otros usuarios.

![[Pasted image 20231031131143.png]]

![[Pasted image 20231031131208.png]]

## SMTP 

>[!info] Conviene utilizar [[#MSMTP]] que viene ya incluido en el docker-compose.yml que estamos utilizando. 

Nos vamos al contendor:
```bash
docker exec -u root -it icinga2_icinga2_1 /bin/bash
``````

```bash
apt update
apt-get install mailutils ssmtp -y
```
```bash
vim /etc/ssmtp/ssmtp.conf
```
```shell
#
# Config file for sSMTP sendmail
#
# The person who gets all mail for userids < 1000
# Make this empty to disable rewriting.
#root=5582351@alu.murciaeduca.es
root=postmaster
# The place where the mail goes. The actual machine name is required no 
# MX records are consulted. Commonly mailhosts are named mail.domain.com
AuthUser=5582351@alu.murciaeduca.es
AuthPass=tucontraseña
mailhub=smtp.gmail.com:587
UseSTARTTLS=YES
# Where will the mail seem to come from?
#rewriteDomain=

# The full hostname
hostname=a2a99138f464

# Are users allowed to set their own From: address?
# YES - Allow the user to specify their own From: address
# NO - Use the system generated From: address
FromLineOverride=YES
```

Es probable que el siguiente archivo no exista en el contenedor por lo que tendremos que crearlo:
```bash
vim /etc/mail.rc 
```
```shell
set mail="/usr/bin/msmtp -t"
```

Probamos:
```bash
/etc/icinga2/scripts/mail-host-notification.sh -d 'test' -l 'test' -n 'test' -o 'test' -r 'jmornav@gmail.com' -s 'OK' -t 'Problema'
```

![[Pasted image 20231102125439.png]]

## MSMTP 

```bash
cd icinga2/msmtp/
```
```bash
vim msmtprc
```

```shell
# Set default values for all following accounts.
defaults

auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log
aliases        /etc/aliases

# Gmail
account        gmail
host           smtp.gmail.com
port           587
from           5582351@alu.murciaeduca.es
user           5582351@alu.murciaeduca.es
password      <contraseña en texto claro, si así lo desea>
#passwordeval	gpg2 --no-tty -q -d /etc/apache2/ssl/password-gmail.gpg
# Set a default account
account default: gmail
```

### Asegurar la contraseña

#### Contraseñas de aplicación
En vez de proporcionar la contraseña de la cuenta, conviene utilizar las *contraseñas de aplicación* de Google que se diseñaron para este mismo propósito. 

![[Pasted image 20231219114122.png]]

![[Pasted image 20231219114652.png]]

#### Archivo encriptado

Generamos una clave completa:
```bash
gpg --full-generate-key
```
- Nombre y apellidos: `javier
- Dirección de correo electrónico: `5582351@alu.murciaeduca.es
Exportamos la clave y la movemos a cualquier directorio del contenedor.
```
gpg --export-secret-keys --armor 5582351@alu.murciaeduca.es > javier_secret_key.asc
```

 Añadimos los siguientes volúmenes en `docker-compose.yml`:
```
      - ./data/icinga/etc/claves:/etc/claves
      - ./rootkeysfolder:/root/.gnupg/
```
Creamos el directorio y metemos ahí la clave privada. 
```
mkdir data/icinga/etc/claves
mv javier_secret_key.asc data/icinga/etc/claves
```

Ahora crearemos un archivo encriptado con la contraseña. 
```
gpg --encrypt -o data/icinga/etc/claves/password-gmail.gpg -r 5582351@alu.murciaeduca.es -
Nuestracontraseña # Y ahora intro
# CTRL + D
```
De esta manera, evitamos que la contraseña se guarde en el historial, al contrario de si hiciéramos "`echo -e "password\n" | gpg --encrypt -o .msmtp-gmail.gpg -r <email>`", aunque el servidor dónde se hospede el contenedor debería estar totalmente asegurado frente amenazas. 

Nos vamos al contendor:
```
docker exec -u root -it icinga2_icinga2_1 /bin/bash
```
Importamos las claves para el usuario root:
```
gpg --import /etc/claves/javier_secret_key.asc
```

Probamos que funcione:
```
gpg --quiet --no-tty --decrypt /etc/claves/password-gmail.gpg
```

> [!warning] Problema
> Tendrá que repetir este comando cada vez que reinicie el contenedor, ya que GPG necesita al usuario para introducir la contraseña, por lo menos una vez. Despues de ejecutarlo de manera manual, msmtp podrá enviar correos sin problema. 

Enviamos un mensaje de prueba:
```
echo "hello there username." | msmtp -a default jmornav@gmail.com
```

En caso de error: 
```
cat /var/log/msmtp.log
```
## Configuración en Icinga2

Necesitamos algunos archivos del directorio conf.d (que deshabilitamos por defecto) en nuestra zona master.
```
cp /etc/icinga2/conf.d/commands.conf /etc/icinga2/zones.d/master/
cp /etc/icinga2/conf.d/templates.conf /etc/icinga2/zones.d/master/
cp /etc/icinga2/conf.d/templates.conf /etc/icinga2/zones.d/master/
cp /etc/icinga2/conf.d/timeperiods.conf /etc/icinga2/zones.d/master/
cp /etc/icinga2/conf.d/notifications.conf /etc/icinga2/zones.d/master/
```

Utilizaré el usuario `javier` para ser el receptor de las notificaciones, y le añadiré mi correo. Icinga2 nos obliga a asignar un grupo a las notificaciones.
```
vim /etc/icinga2/zones.d/master/users.conf
```         
```java
object User "javier" {
  display_name = "Javier"
  enable_notifications = true
  states = [ OK, Warning, Critical, Down ]
  types = [ Problem, Recovery ]
  email = "jmornav@gmail.com"
  vars.notification.mail.users = true
}

object UserGroup "grupojavier" {
  display_name = "Grupo de Javier"
}

```

Creamos la notificación y le especificaremos 
```
vim /etc/icinga2/zones.d/master/notifications.conf
```
```java
apply Notification "mail-icingaadmin" to Host {
  import "mail-host-notification"
  user_groups = [ "grupojavier" ]
  users = [ "javier" ]
  assign where host.vars.notification.mail
}

apply Notification "mail-icingaadmin" to Service {
  import "mail-service-notification"
  user_groups = [ "grupojavier" ]
  users = [ "javier" ]
  assign where service.vars.notification.mail
}
```

```
vim /etc/icinga2/zones.d/master/hosts.conf 
```
```java
object Host "ldap.javier.lan" {
  check_command = "hostalive"
  address = "10.0.2.253"
}

object Host "vm1.javier.lan" {
  import "generic-host"
  address = "10.0.2.1"
  vars.agent_endpoint = name 
  vars.notification["mail"] = {
    users = [ "javier" ]
  }
}

object Host "vm2.javier.lan" {
  import "generic-host"
  address = "10.0.2.2"
  vars.agent_endpoint = name
  vars.notification["mail"] = {
    users = [ "javier" ]
  }
}
```

## Ajustes útiles

Una vez icinga realiza el check y descubre que ha habido un cambio en el endpoint, icinga espera 30 minutos para notificar un cambio. Podemos cambiarlo a 10 segundos tal que así:
```yaml
apply Notification "mail-icingaadmin" to Host {
  import "mail-host-notification"
  user_groups = [ "icingaadmins" ]
  users = [ "icingaadmin" ]

  interval = 10s
  assign where host.vars.notification.mail
}
```

Una vez realizado el check (cada minuto, con la plantilla `generic-host` al menos), esperará 10 segundos a notificar el estado y enviar el correo. 



