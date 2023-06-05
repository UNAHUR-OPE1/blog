---
title: Docker Compose Reverse Proxy
date: 2023-05-29 00:00:00 -300
published: true
categories: [ docker,docker-compose,nginx, proxy]
tags: []
mermaid: true
---

Un proxy reverso o _reverse proxy_ es un componente que actua de intermediero entre el cliente y uno o varios servidores
Se puede uitilizar para:

- Balanceo de carga
- Implementacion de cache
- Utilizacion de SSL
- Enrutamiento

## Redes en docker compose
![Sincronizacion](/assets/img/2023/docker-reverse-rpoxy/reverse-proxy.drawio.svg)

{% drawio path="/assets/img/2023/docker-reverse-rpoxy/reverse-proxy.drawio" page_number=1 height=400 %}

## Fuentes de informacion

https://httpd.apache.org/docs/2.0/mod/mod_proxy.html#forwardreverse
