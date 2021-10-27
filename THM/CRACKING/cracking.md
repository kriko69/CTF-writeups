# CRACKING ROOMS

- [[#INTRODUCCION|INTRODUCCION]]
- [[#CONSEJOS|CONSEJOS]]
- [[#HERRAMIENTAS PARA IDENTIFICAR EL HASH|HERRAMIENTAS PARA IDENTIFICAR EL HASH]]
	- [[#HASH IDENTIFIER|HASH IDENTIFIER]]
	- [[#HASHID|HASHID]]
	- [[#RECURSOS ONLINE|RECURSOS ONLINE]]
	- [[#TIPOS DE HASHES|TIPOS DE HASHES]]
- [[#HERRAMIENTSA PARA CREAR DICCIONARIOS|HERRAMIENTSA PARA CREAR DICCIONARIOS]]
	- [[#CWEL|CWEL]]
	- [[#CRUNCH|CRUNCH]]
- [[#RECURSOS ONLINE DE CRACKING|RECURSOS ONLINE DE CRACKING]]
- [[#HERRAMIENTAS DE CRACKING|HERRAMIENTAS DE CRACKING]]
	- [[#JOHN THE RIPPER|JOHN THE RIPPER]]
		- [[#CRACKING WINDOWS HASHES|CRACKING WINDOWS HASHES]]
		- [[#CRACKING LINUX HASHES|CRACKING LINUX HASHES]]
- [[#CRACKING FILES|CRACKING FILES]]
	- [[#HYDRA|HYDRA]]
	- [[#HASHCAT|HASHCAT]]


## INTRODUCCION

Vamos a ver todo lo relacionado con el cracking de contraseñsa a traves de varios **rooms**.

## CONSEJOS

- usar diccionarios como:

```bash
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/rockyou.txt.gz
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.tx
```

- buscar un diccionario en especifico en **seclist**:

[https://github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists)

- usar el nuevo dicconario **kaonashi**:

[https://github.com/kaonashi-passwords/Kaonashi](https://github.com/kaonashi-passwords/Kaonashi)

## HERRAMIENTAS PARA IDENTIFICAR EL HASH

### HASH IDENTIFIER

[https://github.com/blackploit/hash-identifier](https://github.com/blackploit/hash-identifier)

```bash
hash-identifier
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/THM/CRACKING/images/1.png)

pegas el hash y listo.

### HASHID

```bash
sudo apt install hashid
```

PARAMETROS OPCIONALES:

- -e: muestra una lista de los posibles hashes mas extendida.
- -m: muestra el modulo necesario para **hashcat**.
- -j: muestra el formato necesario para **JohnTheRipper**.
- -o FILE: defines un archivo para la salida (output).

```bash
hashid [-e] [-m] [-j] [-o FILE] HASH
```

### RECURSOS ONLINE

- [https://www.onlinehashcrack.com/hash-identification.php](https://www.onlinehashcrack.com/hash-identification.php)
- [https://md5hashing.net/hash_type_checker](https://md5hashing.net/hash_type_checker)

### TIPOS DE HASHES

Los hash de contraseña de estilo Unix son muy fáciles de reconocer, ya que tienen un prefijo. El prefijo le indica el algoritmo de hash que se utiliza para generar el hash. El formato estándar es `$format$rounds$salt$hash`.

Las contraseñas de Windows se procesan utilizando NTLM, que es una variante de md4. Son visualmente idénticos a los hash md4 y md5, por lo que es muy importante usar el contexto para determinar el tipo de hash.

En Linux, los hash de contraseña se almacenan en / etc / shadow. Este archivo normalmente solo es legible por root. Solían estar almacenados en / etc / passwd y todos podían leerlos.

En Windows, los hashes de contraseña se almacenan en SAM. Windows intenta evitar que los usuarios normales los descarguen, pero existen herramientas como mimikatz para ello. Es importante destacar que los hashes encontrados allí se dividen en hashes NT y hashes LM.

Aquí hay una tabla rápida de la mayoría de los prefijos de contraseña de estilo Unix que verá.

|Prefijo|Algoritmo|
|:---:|:---:|
|$ 1 $|md5crypt, utilizado en Cisco y en sistemas Linux / Unix más antiguos|
|$ 2 $, $ 2a $, $ 2b $, $ 2x $, $ 2y $|Bcrypt (popular para aplicaciones web)|
|$ 5 $|sha256crypt|
|$ 6 $|sha512crypt (predeterminado para la mayoría de los sistemas Linux / Unix)|

## HERRAMIENTSA PARA CREAR DICCIONARIOS

### CWEL

Cwel te permite armar un diccionario con las palabras recolectadas de una pagina web en especifica. Muy util para ataques de fuerza bruta de inicio de sesion a paginas web.

MODO PREDETERMINADO

```bash
cewl https://www.facebook.com
```

PARAMETROS OPCIONALES

- -w FILE: guardar output en un archivo.

```bash
cewl https://www.facebook.com -w dic.txt
```

- -m NUM: define una longitud de palabras especifica.

```bash
cewl https://www.facebook.com -m 9
```

- -e: modo para recuperar correos electronicos.
- -n: oculta la lista de palabras obtenida.

```bash
cewlhttps://www.facebook.com -n -e
```

- -c: contar el numero de palabras repetidas en una pagina.

```bash
cewl https://www.facebook.com -c
```

- -d: define la profundidad de busqueda de palabras. A mayor profundidad mas palabras pero en mas tiempo.

```bash
cewl https://www.facebook.com -d 3
```

- --debug: modo debug.

```bash
cewl https://www.facebook.com --debug
```

- -v: verbose.

```bash
cewl https://www.facebook.com -v
```

- --with-numbers: genera una lista alfanumerica.

```bash
cewl http://testphp.vulnweb.com/ --with-numbers
```

**en caso de que la pagina tenga una autenticacion basica:**

```
cewl  --auth_type Digest --auth_user admin --auth_pass password -v
```

Fuente: [https://esgeeks.com/como-utilizar-cewl/](https://esgeeks.com/como-utilizar-cewl/)

### CRUNCH

```bash
crunch <min-len> <max-len> [charset string] [options]
```

- min-len: longitud minima de la cadena.
- max-len: longitud maxima de la cadena.
- charset string: cadena que toma como base. (**OPCIONAL**)

sin string:


```bash
crunch 2 3 -o  /root/Escritorio/0.txt
```

con string base:

```bash
crunch 4 5 palabra -o  /root/Escritorio/1.txt
```

crear lista de palabras alfanumericas:

```bash
crunch 3 4 geek123 -o  /root/Escritorio/2.txt
```

crear lista con caracteres especiales, todos los caracteres especiales deben esacparse. Un espacio se represeta asi **(\\)**, las comillas asi **(\")** y asi los caracteres especiales:

```
crunch 1 3  geek\  /root/Escritorio/3.txt
```

lista con un patron especifico.

Usando la opción `-t` puedes generar 4 tipos de patrones como se especifica a continuación:

-   **@**: para alfabetos en minúsculas
-   **,**: (‘coma’) para alfabetos en mayúsculas
-   **%**: para caracteres numéricos
-   **^**: para el símbolo de carácter especial

```
crunch 6 6 -t geek%% -o /root/Escritorio/5.txt
```


Fuente: [https://blog.ehcgroup.io/2019/01/09/15/29/15/4518/como-utilizar-crunch-una-guia-completa/hacking/ehacking/](https://blog.ehcgroup.io/2019/01/09/15/29/15/4518/como-utilizar-crunch-una-guia-completa/hacking/ehacking/)

## RECURSOS ONLINE DE CRACKING

- [https://crackstation.net/](https://crackstation.net/)
- [https://www.onlinehashcrack.com/](https://www.onlinehashcrack.com/)

## HERRAMIENTAS DE CRACKING

### JOHN THE RIPPER

**modo predeterminado:**

```bash
john hash.txt
```

**ver resultados:**

```bash
john hash.txt --show
```

**puedes especificar reglas de contraseña que quieres crackear con `--rule`:**

Las reglas personalizadas se definen en el `john.conf`archivo, generalmente ubicadas en `/etc/john/john.conf`si ha instalado John usando un administrador de paquetes o construido desde la fuente con `make`

ejemplo de regla:

La primera línea:

`[List.Rules:NombreRegla]` - Se usa para definir el nombre de su regla, esto es lo que usará para llamar a su regla personalizada como un argumento de John.

Luego usamos una coincidencia de patrón de estilo de expresión regular para definir en qué parte de la palabra se modificará, nuevamente, solo cubriremos los modificadores básicos y más comunes aquí:

`Az` - Toma la palabra y la agrega con los caracteres que defina  

`A0` - Toma la palabra y la antepone con los caracteres que defina  

`c` - Capitaliza el personaje posicionalmente

  
Estos se pueden usar en combinación para definir dónde y qué en la palabra desea modificar.

Por último, necesitamos definir qué caracteres deben agregarse, anteponerse o incluirse de otra manera, lo hacemos agregando juegos de caracteres entre corchetes `[ ]`en el orden en que deben usarse. Estos siguen directamente los patrones de modificación dentro de las comillas dobles `" "`.

 A continuación, se muestran algunos ejemplos comunes:

  
- `[0-9]` - Incluirá los números 0-9  
- `[0]` - Incluirá solo el número 0  
- `[A-z]` - Incluirá mayúsculas y minúsculas  
- `[A-Z]` - Incluirá solo letras mayúsculas  
- `[a-z]` - Incluirá solo letras minúsculas  
- `[a]` - Incluirá solo un  
- `[!£$%@]` - ¡Incluirá los símbolos! £ $% @  

  Poniendo todo esto junto, para generar una lista de palabras a partir de las reglas que coincidiría con la contraseña de ejemplo "Polopassword1!" (suponiendo que la palabra polopassword esté en nuestra lista de palabras), crearíamos una entrada de regla que se ve así:

```bash
[List.Rules:NombreRegla]

cAz"[0-9] [!£$%@]"
```

  Con el fin de:

- Ponga en mayúscula la primera letra - `c`
- Agregar al final de la palabra - `Az`
- Un número en el rango 0-9 - `[0-9]`
- Seguido de un símbolo que es uno de `[!£$%@]`

Entonces podríamos llamar a esta regla personalizada como un argumento de John usando  `--rule=NombreRegla`.  

Como comando completo: `john --wordlist=[path to wordlist] --rule=NombreRegla hash.txt`

**metodo single crack:**

 En este modo, John usa solo la información proporcionada en el nombre de usuario, para tratar de encontrar posibles contraseñas heurísticamente, cambiando ligeramente las letras y números contenidos en el nombre de usuario.
 
 Si tomamos el nombre de usuario: Markus

Algunas posibles contraseñas podrían ser:

-   Markus1, Markus2, Markus3 (etc.)
-   MArkus, MARkus, MARKus (etc.)
-   Markus !, Markus $, Markus * (etc.)

```bash
john --single --format=[format] hash.txt
```

cambiar el formato del hash:

```bash
**De:**  

1efee03cdcb96d90ad48ccc7b8666033

**Para**

mike: 1efee03cdcb96d90ad48ccc7b8666033
```


**puedes especificar el formato (tipo) de contraseña que quieres crackear con `--format`:**

[lista de formatos](http://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats)

```bash
john --list=formats #ver los formatos, con grep se filtra
```

#### CRACKING WINDOWS HASHES

la estructura de un hash de windows es la siguiente:

**USUARIO:ID:HASH_LM:HASH_NT:::**

decides si crackear la parte LM o NT.

**CASO NT**

hash.ntlm:

```bash
Usuario:1000::204rfr3454fj5t5jfg5tydvdvf4::
```

```bash
john --wordlist=/path/of/dictionary hash.ntlm --format=NT
```

**CASO LM**

hash.ntlm:

```bash
Usuario:1000:204rfr3454fj5t5jfg5tydvdvf4:::
```

```bash
john --wordlist=/path/of/dictionary hash.ntlm --format=LM
```

#### CRACKING LINUX HASHES

Primero necesitas extraer los archivos **/etc/shadow** y **/etc/passwd** y combinarlos con **unshadow**:

```bash
unshadow passwd.txt shadow.txt > passwords
```

pasas ese archivo a john y listo, si quieres puedes especificar un diccionario:

```bash
john passwords
```

```bash
john passwords --show
```

## CRACKING FILES

**ZIP**

```bash
zip2john file.zip > zip.john 

john zip.john
```

asi como zip2john existe para muchos archivos, rar2john, ssh2john, etc.

```bash
fcrackzip -u -D -p '/usr/share/wordlists/rockyou.txt' chall.zip
```

**7z**

```bash
cat /usr/share/wordlists/rockyou.txt | 7za t backup.7z
```

```bash
#Download and install requirements for 7z2john 

wget https://raw.githubusercontent.com/magnumripper/JohnTheRipper/bleeding-jumbo/run/7z2john.pl

apt-get install libcompress-raw-lzma-perl 

./7z2john.pl file.7z > 7zhash.john
```

**PDF**

```bash
apt-get install pdfcrack 

pdfcrack encrypted.pdf -w /usr/share/wordlists/rockyou.txt 

#pdf2john didnt worked well, john didnt know which hash type was
```

**JWT**

```bash
git clone https://github.com/Sjord/jwtcrack.git 

cd jwtcrack

#Bruteforce using crackjwt.py 

python crackjwt.py eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJkYXRhIjoie1widXNlcm5hbWVcIjpcImFkbWluXCIsXCJyb2xlXCI6XCJhZG1pblwifSJ9.8R-KVuXe66y_DXVOVgrEqZEoadjBnpZMNbLGhM8YdAc /usr/share/wordlists/rockyou.txt

#Bruteforce using john

python jwt2john.py eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJkYXRhIjoie1widXNlcm5hbWVcIjpcImFkbWluXCIsXCJyb2xlXCI6XCJhZG1pblwifSJ9.8R-KVuXe66y_DXVOVgrEqZEoadjBnpZMNbLGhM8YdAc > jwt.john 

john jwt.john #It does not work with Kali-John

```

**KEEPASS**

```bash
sudo apt-get install -y kpcli #Install keepass tools like keepass2john 

keepass2john file.kdbx > hash #The keepass is only using password 

keepass2john -k <file-password> file.kdbx > hash # The keepas is also using a file as a needed credential


#The keepass can use password and/or a file as credentials, if it is using both you need to provide them to keepass2john 

john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

**SSH ID_RSA**

[https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py)

aqui un sencillo tutorial:

[TUTORIAL](https://vk9-sec.com/ssh2john-how-to/)

Necesitamos un **id_rsa** de un usuario (Llave privada), para crackear su contraseña:

```bash
ssh2john.py id_rsa > id_rsa.john

john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.john
```

una vez hallada las credenciales:

```bash
chmod 600 id_rsa
```

```bash
ssh -i id_rsa stefano@192.168.0.7 -p 4655
Password: la que se encontro
```

### HYDRA

Para cracking de servicios:

- -l: usuario.
- -L: lista de usuarios.
- -P: diccionario de contraseñas.
- -V: verbose.
- -s: especificas el puerto del servicio si cambia el por defecto.
- -S: coneccion por SSL.
- -C: Esta sentencia se usa eliminando -l/-L y -p/-P ya que aquí se especifica un diccionario combo, osea que tenga tanto usuario como contraseña con el formato user:pass \[-C diccionario-combo.txt\].
- -o: output file.
- -f: cierra Hydra al encontrar la primera contraseña.
- -t: hilos para ir mas rapido.

```bash
hydra -l root -P /path/rockyou -s 2222 ssh -V -t 4
```

```bash
hydra -l admin -P /usr/share/seclists/Passwords/10k_most_common.txt <targetip> http-post-form "/department/login.php:username=^USER^&password=^PASS^:Invalid" -t 16
```

servicios soportados:

```bash
adam6500 
afp
asterisk
cisco
cisco-enable
cvs
firebird
ftp
ftps
http[s]-{head|get|post}
http[s]-{get|post}-form
http-proxy
http-proxy-urlenum
icq
imap[s]
irc
ldap2[s]
ldap3[-{cram|digest}md5][s]
mssql
mysql(v4)
mysql5
ncp
nntp
oracle
oracle-listener
oracle-sid
pcanywhere
pcnfs
pop3[s]
postgres
rdp
radmin2
redis
rexec
rlogin
rpcap
rsh
rtsp
s7-300
sapr3
sip
smb
smtp[s]
smtp-enum 
snmp
socks5
ssh
sshkey
svn
teamspeak
telnet[s]
vmauthd
vnc
xmpp
```

### HASHCAT

MODULOS

* -m: indica el tipo de hash a crackear segun un valor numerico.

Por ejemplo 0 es md5 asi que **-m 0** indica el uso del hash md5

puedes consultar la lista de modulos en el siguiente link: [https://hashcat.net/wiki/doku.php?id=example_hashes](https://hashcat.net/wiki/doku.php?id=example_hasheshttps://hashcat.net/wiki/doku.php?id=example_hashes)

o accediente a u panel de ayuda:

```
hashcat --help | more
```

ATAQUE

* -a: indica el tipo de ataque a utilizar

Se tiene las siguientes opciones de ataque:

* 0: (Straight) ataque de fuerza bruta con un diccionario contra un archivo que contine los hashes.

```
hashcat -m 400 -a 0 file.hash worlist.dict
```

* 1: (Combination) ataca mediante la combinacion de palabras de dos diccionarios

``` 
hashcat -a 1 -m 400 file.hash dict1.dict dict2.dict
```

* 3: (Brute Force) realiza un ataque de fuerza bruta pero en lugar de diccionario se le asigna un patron.


```

? | charset

l | abcdefghijklmnopqrstuvwxyz
u | ABCDEFGHIJKLMNOPQRSTUVWXYZ
d | 0123456789
h | 0123456789abcdef
H | 0123456789ABCDEF
s |  !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
a | ?l?u?d?s
b | 0x00 - 0xff

```

Estos son los patrones disponibles, se antepone el caracter '?' y luego la letra del patron a utilizar.
El patron a es una combinacion de otros patrones.

```
hashcat -a 3 -m 400 file.hash ?a?a?a?d?d
```

El anterior ejemplo realizara un ataque de fuerza bruta segun un patron pensando que la contrasena tiene un longitud de 5 caracteres (por eso 5 veces '?')

* 6: (Hybrid wordlist + Mask) realiza un ataque hibrido combinando un diccionario y una mascara.

una mascara es como un patron del anterior tipo de ataque.

```
hashcat -a 6 -m 3000 file.hash worlist.dict ?a?a?a
```

* 7: (Hybrid mask + wordlist) en la forma inversa del modo 6.

```
hashcat -a 7 -m 3000 file.hash ?a?a?a wordlist.dict
```

