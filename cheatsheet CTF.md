# CONSIDRECIONES A LA HORA DE HACER UNA MAQUINA

## ENUMERACION Y EXPLOTACION

* si hay pagina web enumerar sub directorios con fwuzz

```
wfuzz -c -hc 404 -w /usr/share/dirbuster/wordlist/directory-list-2.3-medium.txt http://10.10.10.229/FUZZ
```

* en sitios wordpress se puede obtener usuarios en la siguiente ruta:

```
curl direccion\_del\_panel\_wp/wp-json/wp/v2/users/
```

* revizar siempre archivos de configuracion, ahi normalmente hay credenciales
* una vez dentro verificar el /etc/passwd para ver que usuarios existen
* el puerto 22 ssh normalmente es para conectarse con credenciales que encontremos
* revisar los directorios /opt /home /tmp en linux
* si hay pagina web y no hay directorios enumerar subdominios:

```
wfuzz -c -hc 404 -w /usr/share/dirbuster/wordlist/directory-list-2.3-medium.txt -H "Host: FUZZ.machine.htb" machine.htb

diccionarios buenos para subdirectorios:

/usr/share/dirbuster/wordlist/directory-list-2.3-medium.txt
https://github.com/danielmiessler/SecLists/blob/master/Discovery/DNS/namelist.txt
```

* si hay pagina web y hay una opcion de registrarse, hagamoslo y tratemos de iniciar sesion
* si hay pagina web leer bien el contenido por si dice algo
* cuando hemos encontrado un exploit potencial, leerlo bien para sabver como adaptarlo a nuestras necesidades
* si hay pagina web verificar las versiones de tods las tecnologias que use, estan en los footer, helper o codigo fuente.
* si hay pagina web revisar el codigo fuente (Ctrl+u) y archivos robots.txt
* palabras que esten entre comillas o en negrilla guardar en algun archivo que sirve como contenido para un diccionario. (pagina web)
* todos los nombres de usuarios que veamos nos sirven para realizar fuerza bruta. (pagina web)
* revisar archivos de configuracion y .json en la pagina web y dentro del servidor
* verificar si se puede cargar archivos en la pagina web, que tipo de extension y buscar un archivo malicioso en esa extension para obtener una reverse shell
* si se encuentra varias versiones de un mismo exploit tratar siempre de usar la ultima.
* ponerse a la escucha con netcat y rlwrap
* Las paginas 403 forbidden ocultan algo
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*
*

## ESCALACION DE PRIVILEGIOS LINUX

* verificar que podemos ejecutar como sudo:

```
sudo -l
```

* verificar permisos SUID en directorios:

```
find directorio -user root -perm -4000 -exec ls -ldb {} \; > outputfile
```


* si tenemos el poder de asignar permisos a algo asignar permisos SUID a /bin/bash

```
chmod +s /bin/bash

/bin/bash -p -> obtiene la bash como root
```

* verificar tareas cron que se ejcutan:

```
#!/bin/bash 
old\_process = $(ps -eo command) 
while true; do 
	new\_process=$(ps -eo command) 
	diff <(echo "$old\_process") <(echo "$new\_process") | grep "\[\\>\\<\]" | 		grep -v -E 'procmon|command|' 
	old\_process=$new\_process done
```

```
chmod +x proc.sh 
./proc.sh
```

*
*
*
*
*
*
*
*
