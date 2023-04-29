---
date: 2023-04-25 23:48:05
layout: post
title: Personalización shell con ZSH y Powelevel10k
subtitle: 
description: >-
    🤠En esta ocasión vamos a personalizar la shell de nuestro linux usando zsh y powerlevel10k🤠
image: /assets/images/articles/personalizar-terminal/zsh.png
optimized_image: /assets/images/articles/personalizar-terminal/zsh.png
category: blog
tags:
  - Linux
  - blog
author: elc4br4
paginate: true
---


## Requisitos

1. Instalamos zsh

```bash
sudo apt-get install zsh

brew install zsh
```

2. Establecer ZSH como shell predeterminada

```bash
chsh -s $(which zsh)
```


3. Cerramos y volvemos a abrir la terminal para verificar que ZSH esta como predeterminada (en ocasiones puede que haya que reiniciar el sistema).

```bash
❯ echo $SHELL
/usr/bin/zsh
```

## INSTALACIÓN OH-MY-ZSH

Ejecutamos el siguiente comando para descargar e instalar oh-my-zsh

```bash 
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" 
```


Una vez instalado accedemos a la configuración de oh-my-zsh

```bash 
nano ~/.zshrc
```


Se nos abrirá un documento que debemos editar añadiendo la siguiente línea

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```


![](/assets/images/personalizar-terminal/powerlevel10k.png)

En este caso añadimos ZSH_THEME=”powerlevel10k/powerlevel10k”

Pero hay muchos más temas disponibles para oh-my-zsh en el siguiente enlace

[https://github.com/ohmyzsh/ohmyzsh/wiki/Themes](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)


## INSTALACIÓN POWERLEVEL10K

Para instalar powerlevel10k debemos ejecutar el siguiente comando

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k 
```

Una vez lo ejecutamos cerramos la terminal y volvemos a abrirla y seguimos los pasos de configuración que más nos gusten.


**¡¡¡La instalación y configuración ha de realizarse dos veces, para el usuario sin permisos y para el usuario root!!!**

Un dato curioso además de la propia personalización es la posibilidad de instalar plugins para la misma.

Aquí una lista → [https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins)



