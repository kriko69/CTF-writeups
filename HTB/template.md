#  MACHINE

**Autor: Christian Jimenez**

![[Pasted image 20210518115903.png]]

## ESCANEO Y ENUMERACION

vamos a realizar un escaneo con nmap:

```
nmap -p- --open -T5 - v -n 10.10.10.226 -oG allPorts
```

La salida nos muesta el puerto 22 y 500 abiertos:

![[Pasted image 20210518114510.png]]

Vamos a realizar una enumeracion de los servicios en los puertos:

```
nmap -p22,5000 -sV -sC 10.10.10.226 -oN targeted
```

## EXPLOTACION



## ELEVACION DE PRIVILEGIOS

