---
title: Docker Compose Reverse Proxy
date: 2023-05-29 00:00:00 -300
published: false
categories: [ docker,docker-compose,nginx, proxy]
tags: []
mermaid: true
---
Este documento explica la impoortancia de implementar un proxy reverso en los despliegues con `docker-compose`.

## ¿Que es un Proxy Reverso?

Un proxy reverso o _reverse proxy_ es un componente del lado del server que actua de intermediero entre el cliente y uno o varios servidores
Se puede uitilizar para:

- Balanceo de carga
- Utilizacion de SSL
- Aislar
- Implementacion de cache
- Enrutamiento

## ¿Que es un Proxy o Fowrward Proxy?
El contrapunto de un proxy reverso es un Proxy o _Fowrward Proxy_ que cumple el rol inverso. Es un componente del lado del cliente que hace de intermedieario entre varios cliente y una conexion de internet o conexion a otra LAN o WAN.

## Proxy vs Proxy Reverso
![DIAGRAMA SIN REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/proxy_vs_reverse_proxy.jpg)

## Redes en Docker
Las redes (networks) en Docker son el mecanismo de iinterconexion de los contenedores y de conexion con componentes fuera de Docker.

### Tipos de redes
Docker permite expandir el soprote de redes con plug-ins, pero los mas relevantes para este curso son:

- `bridge`: El controlador de red predeterminado. Si no especifica un controlador, este es
  el tipo de red que está creando. Las redes puente se usan comúnmente cuando
  su aplicación se ejecuta en un contenedor que necesita comunicarse con otros
  contenedores en el mismo host. [Saber más](https://docs.docker.com/network/drivers/bridge/).

- `host`: Elimina el aislamiento de la red entre el contenedor y el host de Docker. Usa la red del host directamente.[Saber más](https://docs.docker.com/network/drivers/host/).

- `none`: Aisla el contenedor completamente tanto del host como de otros contenedores. [Saber más](https://docs.docker.com/network/drivers/host/).


## Redes en Docker-Compose
Al instanciar un set de contenedores con docker-compose este crea una red por defecto que conecta a todos los contenedores / servicios entres si. En esa red tiene un servicio de DNS que le asigna a cada servicio su nombre o el nombre especificado en el parametro `hostname`.

En el siguiente ejemplo, si creamos una carpeta llamada ejemplo y dentro creamos un archivo `docker-compose-yaml` con el siguiente contenido.

```yaml
services:
  cli:
    image: quay.io/pqsdev/mssql-tools-os
    depends_on:
      - mqtt
  mqtt:
      image: eclipse-mosquitto:latest
      container_name: mqtt
      environment:
        - TZ=America/Buenos_Aires
```

El ejecutar el comando `docker-compose up -d` se puede observar que se crea la red `ejemplo_default` junto con los contenedores `ejemplo-mqtt-1` y `ejemplo-cli-1`

```bash
$ cd ejemplo
$ docker-compose up -d
[+] Running 3/3
 - Network ejemplo_default
 - Container ejemplo-mqtt-1
 - Container ejemplo-cli-1
```
Si ingresamos a la temminarl del contenedor `ejemplo-cli-1` con el comando `docker exec -it ejemplo-cli-1  sh` y hacemops un ping a mqtt , el mismo responde.

```bash
$ docker exec -it ejemplo-cli-1  sh
sh-4.2$ ping mqtt
PING mqtt (172.18.0.2) 56(84) bytes of data.
64 bytes from ejemplo-mqtt-1.ejemplo_default (172.18.0.2): icmp_seq=1 ttl=64 time=0.644 ms
64 bytes from ejemplo-mqtt-1.ejemplo_default (172.18.0.2): icmp_seq=2 ttl=64 time=0.087 ms
64 bytes from ejemplo-mqtt-1.ejemplo_default (172.18.0.2): icmp_seq=3 ttl=64 time=0.081 ms
64 bytes from ejemplo-mqtt-1.ejemplo_default (172.18.0.2): icmp_seq=4 ttl=64 time=0.102 ms
```
## Ports y Expose
**Ports** hace una relacion, mapeo o _mapping_ entre un puerto dentro del host y un puerto dentro del contenedor. Al levantar el contenedor automaticamente el puerto del host queda abierto a la espera de conexiones (escuchando).

```yaml
services:
  tools:
    image: quay.io/pqsdev/mssql-tools-os
    depends_on:
      - nginx
  nginx:
    image: nginx
    ports:
      - 8080:80
    environment:
      - TZ=America/Buenos_Aires
```
En el ejemplo anterior si abrimos un navegador en http://localhost:8080 veremos la pagina de inicio n nginx

Y Si ejecutamos una linea de comandos dentro de tools llamando a http://nginx recibiremos la pagina de inicio en html

```bash
docker exec -it ejemplo-tools-1  sh
# deentro del cotnenedor
sh-4.2$ curl http://nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

**Expose** solo habilita el puerto que solo recibira conexiones de servicios/contenedores en la misma red.
```yaml
services:
  tools:
    image: quay.io/pqsdev/mssql-tools-os
    depends_on:
      - nginx
  nginx:
    image: nginx
    expose:
      - 80
    environment:
      - TZ=America/Buenos_Aires
```

En este ejemplo si habrimos un navegador en el hosty y navegamos a http://localhost:8080 o incluso http://localhost:80, no recibiremos ninguna respuesta. Es decir NO funciona por que el puerto no esta mapeado.

Si intentamos acceder al puerto 80, exactamente la misma prueba que hicimos antes, esto sigue funcionando.

```bash
docker exec -it ejemplo-tools-1  sh
# deentro del cotnenedor
sh-4.2$ curl http://nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


Esta distincion es importante por que la idea es que el unico camino a los servicios sea a travez del proxy. Si los puertos de los servicios quedan expuestos directamente se puede hacerle un bypass al proxy. Cuando se implementa un proxy lo unico que esta expuesto o "mapeado" es el proxy.




## PROXY + Docker-Compose
En este ejemplo usaremos [NGINX](https://www.nginx.com/) como proxy-reverso. Hay otras implementaciones como [HAProxy](https://www.haproxy.org/) o [traefik](https://github.com/traefik/traefik#overview).


![DIAGRAMA SIN REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/no-proxy.drawio.svg)

![DIAGRAMA CON REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/reverse-proxy.drawio.svg)

Siguiendo los pasos
- Cambiamos ports por expose
- Agregar el servicio nginx
- Configurar el routing de NGINX mediante el archivo de configuracion [nginx.conf](https://www.nginx.com/resources/wiki/start/topics/examples/full/)

- Revisar la configuracion de los servicios detras del proxy si requieren indicarle el path donde estan hosteados
  - Grafana: necesita una modificacion en su archivo de configuracion, `grafana.ini` para que use el subpath `/grafana/`
  ```ini
  [server]
  root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
  serve_from_sub_path = true
  ```

## PROXY + Docker-Compose +tls


## Fuentes de informacion

### Linkss
- https://httpd.apache.org/docs/2.0/mod/mod_proxy.html#forwardreverse

- [Docker: Network Drivers](https://docs.docker.com/network/drivers/)

- [Docker: Networking in Compose](https://docs.docker.com/compose/networking/)

### Videos