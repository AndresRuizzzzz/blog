---
layout: single
title: GamingServer - TryHackMe
excerpt: " Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system. "
date: 2023-06-21
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-gamingserver/gamingserver.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - ssh2john
  - id_rsa
  - Gobuster
  - lxd
  - Docker
  - container
  - sudoers
---

Can you gain access to this gaming server built by amateurs with no experience of web development and take advantage of the deployment system.

# Summary
- IP: 10.10.43.138
- Ports: 22,80
- OS: Linux 
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Apache httpd 2.4.29
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.43.138 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 10.10.43.138 -oN targeted
```

<br><br>

# Análisis de página web (Port 80):

- Al entrar a la página web principal podemos ver la siguiente interfaz:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620221200.png">
</p>

- Hacemos enumeración de cada apartado disponible y no encontramos nada útil, sin embargo, si revisamos el código fuente podemos encontrar la siguiente linea:


<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620221343.png">
</p>

- Hemos encontrado un posible usuario potencial del sistema: john.

- Hacemos enumeración de directorios con Gobuster:

```bash
$gobuster dir -u 10.10.43.138 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
```

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620221610.png">
</p>

- Encontramos el directorio "/secret", lo abrimos:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620221715.png">
</p>

<br><br>
# SSH id_rsa cracking with ssh2john.py


- Encontramos un archivo "secretKey" y analizamos su contenido:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620221802.png">
</p>

- Todo parece indicar que se trata de una id_rsa encriptada.

- Con la información que tenemos hasta ahora, podemos probar crackear la id_rsa que hemos encontrado para validar que pertenezca al usuario que habíamos encontrado anteriormente (john) y acceder al sistema vía SSH sin contraseña.

- Primero, para crackear la id_rsa la guardamos en un archivo y la convertimos en un hash válido usando ssh2john para luego aplicar fuerza bruta con John The Ripper:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620222341.png">
</p>

- Bingo! hemos encontrado la contraseña del id_rsa que encontramos en la web, ahora le damos permisos de ejecución al archivo de identificación y lo usamos para acceder por SSH como el usuario "john" proporcionando la contraseña que hemos crackeado:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620222742.png">
</p>

- Hemos ganado acceso al sistema como el usuario "john" y podremos leer la primera flag.

<br><br>

# ESCALADA DE PRIVILEGIOS:

# Abuso de grupo "lxd"

- Si ejecutamos el comando "id" para verificar en qué grupo está el usuario john, encontramos lo siguiente:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223002.png">
</p>

- Como vemos, el usuario "john" pertenece al grupo "lxd", buscamos vulnerabilidades sobre esto con searchsploit:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223113.png">
</p>

- Leemos en qué consiste la script en bash:

```js
# Step 1: Download build-alpine => wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine [Attacker Machine]
# Step 2: Build alpine => bash build-alpine (as root user) [Attacker Machine]
# Step 3: Run this script and you will get root [Victim Machine]
# Step 4: Once inside the container, navigate to /mnt/root to see all resources from the host machine
```

- Así que descargamos el archivo que nos indica en nuestra máquina atacante, para luego aplicarle el build:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223525.png">
</p>

- Obtendremos un archivo ".tar.gz", este y la srcipt en bash los debemos transferir máquina víctima:


--- Máquina Atacante
<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223714.png">
</p>


---Máquina Víctima

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223837.png">
</p>

- Por último, le damos permisos de ejecución a la script y la ejecutamos pasándole como argumento el archivo ".tar.gz":

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620223940.png">
</p>

- Y como podemos apreciar, ya seríamos root, PEEERO hay que tener en cuenta que nos encontramos en un contenedor, no en la máquina real.

- Todos los archivos de la máquina real los podremos visualizar en el directorio "/mnt/root"

- Para salir del contenedor podemos modificar el archivo "/etc/sudoers" de la máquina real, añadiendo las siguientes lineas para decirle al sistema que cualquier usuario podrá usar el comando sudo sin necesidad de usar contraseña:

```
ALL ALL=(ALL) NOPASSWD: ALL
```

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620224412.png">
</p>

- Hecho esto, guardamos el archivo y salimos del contenedor con el comando "exit".

- Siendo nuevamente el usuario "john" en la máquina real solo resta ejecutar el comando "sudo su" para escalar privilegios y seremos ROOT pudiendo leer la última flag:

<p style="text-align: center;">
<img src="/blog/assets/images/thm-writeup-gamingserver/20230620224612.png">
</p>



