
Creamos una integración en rocketchat:

![[Pasted image 20231105130829.png]]

Mas abajo, tenemos el webhook que permitirá a Icinga autenticarse en Rocketchat:

![[Pasted image 20231105131017.png]]

Al final de la página se nos proporciona el comando curl con todos los parámetros necesarios para verificar la conexión a partir del webhook:

![[Pasted image 20231105131322.png]]

El resultado debería ser *{"success":true}*.
```bash
curl -X POST -H 'Content-Type: application/json' --data '{"text":"Example message","attachments":[{"title":"Rocket.Chat","title_link":"https://rocket.chat","text":"Rocket.Chat, the best open source chat","image_url":"http://192.168.1.107:3000/images/integration-attachment-example.png","color":"#764FA5"}]}' http://192.168.1.107:3000/hooks/652d122841c4c5f5f63918ed/dhzM2XM7yqZwAGwbY9bQ6fNoPwXF7G32MXnW23WuJjvdDRKN
```
```bash
{"success":true}
```

Descargamos el [plugin](https://exchange.icinga.com/Grandmaster/notify_rocketchat/releases).
```bash
cd /etc/icinga2/scripts/
wget https://exchange.icinga.com/Grandmaster/notify_rocketchat/files/13614/notify_rocketchat.amd64
chmod +x notify_rocketchat.amd64
```

export ROCKETCHAT_WEBHOOK_URL="https://rocketchat.javierlan.net/hooks/652d122841c4c5f5f63918ed/dhzM2XM7yqZwAGwbY9bQ6fNoPwXF7G32MXnW23WuJjvdDRKN"
Probamos a ejecutarlo.
```bash
export ROCKETCHAT_WEBHOOK_URL="http://192.168.1.107:3000/hooks/652d122841c4c5f5f63918ed/dhzM2XM7yqZwAGwbY9bQ6fNoPwXF7G32MXnW23WuJjvdDRKN"

/etc/icinga2/scripts/notify_rocketchat.amd64 -host.name hostname -host.state down -host.name hostname -service.name ping -service.state up
```
![[Pasted image 20231018112147.png]]

Añadimos el nuevo [NotificationCommand](https://github.com/Al2Klimov/notify_rocketchat/blob/master/icinga2/command.conf) a commands.conf 
```
vim /etc/icinga2/zones.d/master/commands.conf
```
```java
  object NotificationCommand "rocketchat" {
  import "plugin-notification-command"
    command = [ ConfigDir + "/scripts/notify_rocketchat.amd64" ]

    arguments = {
      "-icinga.timet" = "$icinga.timet$"
      "-host.name" = "$host.name$"
      "-host.display_name" = "$host.display_name$"
      "-host.action_url" = "$rocketchat_host_action_url$"
      "-host.state" = "$host.state$"
      "-host.output" = "$host.output$"
      "-service.name" = "$service.name$"
      "-service.display_name" = "$service.display_name$"
      "-service.action_url" = "$rocketchat_service_action_url$"
      "-service.state" = "$service.state$"
      "-service.output" = "$service.output$"
    }

    env = {
      "ROCKETCHAT_WEBHOOK_URL" = "$rocketchat_webhook_url$"
    }

    vars.rocketchat_host_action_url = "$host.action_url$"
    vars.rocketchat_service_action_url = "$service.action_url$"
  }
```

Aplicamos las notificaciones:
```bash
vim /etc/icinga2/zones.d/master/notifications.conf
```
```java
apply Notification "rocketchat-host" to Host {
	command = "rocketchat"
	users = [ "icingaadmin" ]
	assign where host.vars.notification_rocketchat
	vars.rocketchat_webhook_url = "http://192.168.1.107:3000/hooks/652d122841c4c5f5f63918ed/dhzM2XM7yqZwAGwbY9bQ6fNoPwXF7G32MXnW23WuJjvdDRKN"
}

apply Notification "rocketchat-host" to Service {
        command = "rocketchat"
        users = [ "icingaadmin" ]
        assign where service.vars.notification_rocketchat
        vars.rocketchat_webhook_url = "http://192.168.1.107:3000/hooks/652d122841c4c5f5f63918ed/dhzM2XM7yqZwAGwbY9bQ6fNoPwXF7G32MXnW23WuJjvdDRKN"
}
```

Para probar:
Añadimos la variable `vars.notification_rocketchat` al host `vm2.javier.lan`.
```bash
vim /etc/icinga2/zones.d/master/hosts.conf
```
```java
object Host "vm2.javier.lan" {
  check_command = "hostalive"
  address = "10.0.2.4"
  vars.notification["mail"] = {
    users = [ "icingaadmin" ]
  }
vars.agent_endpoint = name
vars.notification_rocketchat = true
}
```

Y al servicio LDAP:
```bash
vim /etc/icinga2/zones.d/master/notifications.conf
```
```java
apply Service "LDAP Service" {
  check_command = "ldap"

  vars.ldap_host = "ldap.javier.lan"
  vars.ldap_base = "ou=miembros,dc=javier,dc=lan"

  vars.ldap_v3 = true
  assign where host.name == "master.javier.lan"
  vars.notification["mail"] = {
    users = [ "icingaadmin" ]
  }

  vars.notification_rocketchat = true
}
```

![[Pasted image 20231116175853.png]]


