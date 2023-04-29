---
layout      : post
title       : "Return - HackTheBox"
author      : elc4br4
image       : /assets/images/HTB/Return-HackTheBox/Return.webp
original_image: /assets/images/HTB/Lame-HackTheBox/Return.webp
category    : [ htb ]
tags        : [ Windows ]
description : Esta vez tenemos una máquina Windows de nivel Easy en la que explotaremos un servidor web de Administración de Impresora para obtener unas credenciales con las que posteriormente nos conectaremos a la máquina a través de Winrm y escalaremos privilegios a través de una mala configuración de grupos de usuario con la que podremos crear, modificar, iniciar y parar servicios.
---

Esta vez tenemos una máquina Windows de nivel Easy en la que explotaremos un servidor web de Administración de Impresora para obtener unas credenciales con las que posteriormente nos conectaremos a la máquina a través de Winrm y escalaremos privilegios a través de una mala configuración de grupos de usuario con la que podremos crear, modificar, iniciar y parar servicios.

![](/assets/images/HTB/Return-HackTheBox/rating-return.png)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)

...


**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [SMB](#smb).
3. [Enumeración](#enumeración).
 * [Enumeración Web](#enum-web).
4. [Escalada de Privilegios](#privesc). 
    * [Server Operators](#server-operators).      


...

# Reconocimiento [#](reconocimiento) {#reconocimiento}

----

## Reconocimiento de Puertos [📌](#recon-nmap) {#recon-nmap}

```nmap
PORT      STATE SERVICE          REASON
53/tcp    open  domain           syn-ack ttl 127
80/tcp    open  http             syn-ack ttl 127
88/tcp    open  kerberos-sec     syn-ack ttl 127
135/tcp   open  msrpc            syn-ack ttl 127
139/tcp   open  netbios-ssn      syn-ack ttl 127
445/tcp   open  microsoft-ds     syn-ack ttl 127
464/tcp   open  kpasswd5         syn-ack ttl 127
636/tcp   open  ldapssl          syn-ack ttl 127
3269/tcp  open  globalcatLDAPssl syn-ack ttl 127
5985/tcp  open  wsman            syn-ack ttl 127
49664/tcp open  unknown          syn-ack ttl 127
49665/tcp open  unknown          syn-ack ttl 127
49674/tcp open  unknown          syn-ack ttl 127
49682/tcp open  unknown          syn-ack ttl 127
```

Como podemos ver tenemos un listado de puertos bastante amplio, pero esa información no es suficiente, necesito saber que servicios y versiones se ejecutan en cada puerto.

Lo más relevante es que tenemos el protocolo SMB, HTTP y en el puerto 5985 el servicio de Administración Remota (WinRM)

# SMB [#](smb) {#smb}

Comenzaré por enumerar un poco más el protocolo SMB.

Usaré crackmapexec bajo el protocolo SMB para obtener más información acerca del objetivo.

![](/assets/images/HTB/Return-HackTheBox/crackmapexecsmb.png)


Como vemos ya vamos obteniendo algo de info

1. Nombre de la máquina --> _Printer_

2. Dominio --> _return.local_

3. Protocolo _SMB firmado_

Ahora vamos a ver si tenemos algún recurso compartido pero claro, no tenemos credenciales, asique debemos usar el parámetro -N (Null session)

![](/assets/images/HTB/Return-HackTheBox/smbclient.png)


> NO hay nada

Por lo tanto hasta que no tengamos credenciales no hay mucho que hacer. Asique paso a enumerar el seevidor web.

# Enumeración [#](enumeración) {#enumeración}

----

## Enumeración Web [📌](#enum-web) {#enum-web}

Al acceder a la ip desde el navegador encontramos lo siguiente:

![](/assets/images/HTB/Return-HackTheBox/web1.png)

Es un panel de administración de una impresora

Enuemerando encuentro que en settings tenemos un formulario con credenciales y el dominio...

![](/assets/images/HTB/Return-HackTheBox/web2.png)

Igual si cambio el campo del dominio por la ip de mi VPN (interfaz tun0) y abro un listener en netcat podría conectarme a la máquina asique probemos.

Ponemos el netcat en escucha en puerto 389

Y en la web añadimos nuestra ip 

![](/assets/images/HTB/Return-HackTheBox/web3.png)

Clickamos en update y en la sesión de netcat recibimos unas credenciales

![](/assets/images/HTB/Return-HackTheBox/netcat.png)

# Evil Winrm [#](evil-winrm) {#evil-winrm}

Ahora que tenemos credenciales podríamos probar a conectarnos a través de WinRm...

Y conseguimos conectarnos 

![](/assets/images/HTB/Return-HackTheBox/evilwinrm1.png)

![](/assets/images/HTB/Return-HackTheBox/impresora.gif)


Ahora toca escalar privilegios

# Escalada de Privilegios [#](privesc) {#privesc}

----

## Server Operators[👽](server-operators) {#server-operators}

Para escalar primero debemos enumerar, comenzando por los permisos del usuario actual svc-printer

![](/assets/images/HTB/Return-HackTheBox/escalada1.png)

Si nos fijamos el usuario pertenece a los grupos "Print Operators" , "Remote Management Use" y "Server Operators".

Lo que más me llama la atención es "Server Operators", pero no tengo ni idea de que podríamos hacer para escalar asique busqué info sobre este grupo.

Y buscando di con este artículo:

[http://0xma.com/hacking/privilege_escalation_via_server_operators_group.html](http://0xma.com/hacking/privilege_escalation_via_server_operators_group.html)

Por lo que he podido investigar, al pertener a este grupo podemos arrancar, parar y reiniciar servicios, por lo tanto podríamos crear o modificar un servicio en ejecución y editarlo para así obtener una conexión reversa.

Sigo los pasos del post:

1.Subir netcat a la máquina víctima

![](/assets/images/HTB/Return-HackTheBox/nc.png)

2.Vemos que servicios hay en ejecución y lo modificamos

![](/assets/images/HTB/Return-HackTheBox/services.png)

Tenemos varios servicios en ejecución, ahora procedemos a modificar uno de ellos.

![](/assets/images/HTB/Return-HackTheBox/services2.png)

Ya lo tenemos modificado

3.A continaución toca poner netcat en escucha en el puerto configurado (443), y ahora debemos arrancar el servicio.

![](/assets/images/HTB/Return-HackTheBox/services3.png)

![](/assets/images/HTB/Return-HackTheBox/root.png)


> Y ya hemos escalado privilegios, somos el usuario Administrador.

