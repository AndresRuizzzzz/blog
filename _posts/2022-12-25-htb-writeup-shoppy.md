---
layout: single
title: Shoppy - HackTheBox
excerpt: " Shoppy was one of the easier HackTheBox weekly machines to exploit, though identifying the exploits for the initial foothold could be a bit tricky. "
date: 2022-12-25
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-shoppy/shoppy.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:  
  - nosql
  - mongodb
  - subdomain
  - docker
---

Shoppy was one of the easier HackTheBox weekly machines to exploit, though identifying the exploits for the initial foothold could be a bit tricky.

# # Summary
- IP: 10.10.11.180
- Ports: 22,80,9093,
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 8.4p1 Debian 5+deb11u1
	-  80 -> nginx 1.23.1
	-  9093 -> copycat?

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80,9093 -sCV 10.10.116.216 -oN targeted
``` 

- Análisis de directorios web con gobuster:

$sudo gobuster dir -u shoppy.htb -w /home/dante/SecLists/Discovery/Web-Content/common.txt

- Nos encontramos con un directorio "login" el cual nos despliega una ventana de logeo, al no poseer credenciales intentamos inyectar payload nosql:

```
admin'||'1==1
```

- Accedemos a la web de shoppy, en la que encontramos un buscador de usuarios, insertamos de nuevo el mismo payload que inyectamos anteriormente y nos arroja un archivo json con dos usuarios y credenciales hasheadas.

- Intentamos romper los hashes con john:

$sudo john -w:/home/dante/SecLists/Passwords/Leaked-Databases/rockyou.txt hash --format=Raw-MD5

- Encontramos la crendencial de "josh" pero por ahora no nos sirve; buscamos subdominios de "shoppy.htb":

```bash
$sudo gobuster vhost -w /home/dante/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 100 -u shoppy.htb
```

- Encontramos "mattermost.shoppy.htb" accedemos y en la web depslegada al analizar encontramos una conversación privada en la que hay credenciales, ingresamos por SSH como el usuario jaeger y encontramos la user flag.


# ESCALANDO PRIVILEGIOS:


- Al hacer sudo -l vemos que podemos ejecutar el binario "password-manager" como el usuario "deploy", lo hacemos:

```bash
$sudo su -u deploy ./password-manager
```

- Pero nos pide una contraseña maestra; hacemos cat al binario y encontramos una crendencial, la probamos y nos despliega las credenciales del usuario "deploy":

```bash
cat password-manager
```

- Accedemos por SSH como "deploy" y nos damos cuenta que pertenecemos al grupo "docker", haciendo una búsqueda en gtfobins encontramos una forma de ser root con esta información, que consiste en montar la raiz de la máquina en un contenedor y por medio de una montura tener acceso como root dentro del contenedor:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

- Aquí podemos encontrar la root flag, sin embargo aún no tenemos la máquina ocmpletamente vulnerada, pues estamos dentro de un contenedor, para tener acceso completo aprovechamos que estamos en la raíz de la máquina original y modificamos el archivo de sudoers:

```bash
echo 'ALL ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
```

- Con el archivo ya modificado podemos salir del contenedor con el comando "exit" y comprobamos que podemos ejecutar sudo como cualquier usuario con sudo -l; hacemos "sudo su" y estaremos como root con acceso completo.

# ROOT FLAG: c03008085c5a5a10634d7b6c0de88bb5
