---

title: Docker Compose Reverse Proxy
date: 2023-05-29 00:00:00 -300
published: false
categories: [ docker,docker-compose,nginx, proxy]
tags: []
mermaid: true

---

Este documento explica la importancia de implementar un proxy reverso en los despliegues con `docker-compose`.

## ¿Que es un Proxy Reverso?

Un proxy reverso o _reverse proxy_ es un componente del lado del server que actúa de intermediario entre el cliente y uno o varios servidores
Se puede utilizar para:

- Balanceo de carga
- Utilización de SSL
- Aislar
- Implementación de cache
- Enrutamiento

## ¿Que es un Proxy o Fowrward Proxy?

El contrapunto de un proxy reverso es un Proxy o _Fowrward Proxy_ que cumple el rol inverso. Es un componente del lado del cliente que hace de intermedieario entre varios cliente y una conexión de internet o conexión a otra LAN o WAN.

## Proxy vs Proxy Reverso

![DIAGRAMA SIN REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/proxy_vs_reverse_proxy.jpg)

## Redes en Docker

Las redes (networks) en Docker son el mecanismo de interconexión de los contenedores y de conexión con componentes fuera de Docker.

### Tipos de redes

Docker permite expandir el soporté de redes con plug-ins, pero los mas relevantes para este curso son:

- `bridge`: El controlador de red predeterminado. Si no especifica un controlador, este es
  el tipo de red que está creando. Las redes puente se usan comúnmente cuando
  su aplicación se ejecuta en un contenedor que necesita comunicarse con otros
  contenedores en el mismo host. [Saber más](https://docs.docker.com/network/drivers/bridge/).

- `host`: Elimina el aislamiento de la red entre el contenedor y el host de Docker. Usa la red del host directamente. [Saber más](https://docs.docker.com/network/drivers/host/).

- `none`: Aísla el contenedor completamente tanto del host como de otros contenedores. [Saber más](https://docs.docker.com/network/drivers/host/).

## Redes en Docker-Compose

Al instanciar un set de contenedores con docker-compose este crea una red por defecto que conecta a todos los contenedores / servicios entres si. En esa red tiene un servicio de DNS que le asigna a cada servicio su nombre o el nombre especificado en el parámetro `hostname`.

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

Si ingresamos a la terminal del contenedor `ejemplo-cli-1` con el comando `docker exec -it ejemplo-cli-1  sh` y hacemos un ping a mqtt , el mismo responde.

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

**Ports** hace una relación, mapeo o _mapping_ entre un puerto dentro del host y un puerto dentro del contenedor. Al levantar el contenedor automaticamente el puerto del host queda abierto a la espera de conexiones (escuchando).

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

Y Si ejecutamos una línea de comandos dentro de tools llamando a http://nginx recibiremos la pagina de inicio en html

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

**Expose** solo habilita el puerto que solo recibirá conexiones de servicios/contenedores en la misma red.

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

En este ejemplo si abrimos un navegador en el host y navegamos a http://localhost:8080 o incluso http://localhost:80, no recibiremos ninguna respuesta. Es decir NO funciona por que el puerto no esta mapeado.

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

> Esta distinción es importante por que la idea es que el único camino a los servicios sea a través del proxy. Si los puertos de los servicios quedan expuestos directamente se puede hacerle un bypass al proxy. Cuando se implementa un proxy lo único que esta expuesto o "mapeado" es el proxy.
> {: .prompt-info }

## PROXY + Docker-Compose

En este ejemplo usaremos [NGINX](https://www.nginx.com/) como proxy-reverso. Hay otras implementaciones como [HAProxy](https://www.haproxy.org/) o [traefik](https://github.com/traefik/traefik#overview).

![DIAGRAMA SIN REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/no-proxy.drawio.svg)

![DIAGRAMA CON REVERSE PROXY](/assets/img/2023/docker-reverse-proxy/http-reverse-proxy.drawio.svg)

Siguiendo los pasos

- Cambiamos `Ports` por `Expose`

- Agregar el servicio nginx

- Configurar el routing de NGINX mediante el archivo de configuracion [nginx.conf](https://www.nginx.com/resources/wiki/start/topics/examples/full/)

- Revisar la configuración de los servicios detras del proxy si requieren indicarle el path donde estan hosteados
  
  - Grafana: necesita una modificacion en su archivo de configuracion, `grafana.ini` para que use el subpath `/grafana/`
    
    ```ini
    [server]
    root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
    serve_from_sub_path = true
    ```

> Este ejemplo utiliza los dominios `iot.localdev.me` e `influxdb.localdev.me`. `*iot*.localdev.me` Es un dominio especial creado por Amazon y apunta a las IP `127.0.0.1` , también conocida como localhost. Apunta a nuestra propia PC, por algo denominado loopback adapter, una placa de red virtual en nuestra PC, muy útil para comunicación entre procesos. Esto nos da lugar para jugar con la configuración de NGINX y hacer que dependiendo del domino entremos a Grafana o Influxdb. Otra herramienta es usar el sub-path, pero requiere que el software detrás del proxy lo soporte.
> {: .prompt-info }

**EJEMPLO DE GITHUB:** El ejemplo IOT tiene una rama con las modificaciones para llegar a este punto https://github.com/UNAHUR-OPE1/iot/tree/feature/proxy-http

## PROXY + Docker-Compose + TLS

![DIAGRAMA CON REVERSE PROXY + TLS](/assets/img/2023/docker-reverse-proxy/https-reverse-proxy.drawio.svg)

Como vimos en clase para TLS requiere la generación de una clave publica y una clave privada por cada dominio. Dichas claves son certificados.

En nuestro caso los dominios son `iot.localdev.me` e `influxdb.localdev.me`. Para este ejemplo usaremos certificados auto-firmados o _self-signed_ que son útiles para propósitos de desarrollo. Con un programa llamado `openssl` se puede emitir el certificado. En este ejemplo para `iot.localdev.me.crt`

```bash
openssl req -x509 -newkey rsa:2048  -days 3650 -nodes \
    -keyout iot.localdev.me.key -out iot.localdev.me.crt -subj "/CN=iot.localdev.me" \
    -addext "subjectAltName=DNS:iot.localdev.me"
```

> En ambientes productivos se debe adquirir un certificado en una entidad certificante (CA) valida. Dicho certificado tiene un costo que puede variar según sus características. Incluso puede incluir una verificación por parte de la entidad
> {: .prompt-warn }

Pero el hecho de poner un certificado en el repositorio de codigo NO ES UNA BUENA PRACTICA. Asi que procederemos a empaquetar la generacion del certificado en un contenedor que genera los certificados en un volumen compartido con NGINX.

A fines prácticos los pasos realizados son

1. Generar una imagen llamada `script-runner` que ejecute script de shell. [ver](https://github.com/UNAHUR-OPE1/iot/commit/576e369ab19e8c629fce5456d679f9f23c5bf927)
2. Hacer un shell script que genere los certificados necesarios, dicho script debe se idempoente [commit](https://github.com/UNAHUR-OPE1/iot/commit/ae292e21a59cb6c4cfe55cb56a6b2dc9a4053f20#diff-1ef0f326b3c5ec2b3181fa3e90e4f15dfe778e28935114f735d110ab73fd2f65)
3. Modificar el docker-compose [ver commit](https://github.com/UNAHUR-OPE1/iot/commit/ae292e21a59cb6c4cfe55cb56a6b2dc9a4053f20#diff-3fde9d1a396e140fefc7676e1bd237d67b6864552b6f45af1ebcc27bcd0bb6e9):
   - Incorporar un contenedor de script-runner que ejecute el script del paso anterior
   - Incorporar un volumen donde generar los certificados

Hasta este punto los scripts están generado pero nginx. Debemos seguir realizando cambios

1. Modificar el docker compose para exponer el puerto 443 (utilizado en HTTPS), conectar el contenedor al volumen que con los certificados generados y una dependencia del servicio que genera los certificados para que nginx solo inicie cuando los certificados están listos. [ver commit](https://github.com/UNAHUR-OPE1/iot/commit/31528b74d146a9dc48c901927ad4c693b5d3655f#diff-3fde9d1a396e140fefc7676e1bd237d67b6864552b6f45af1ebcc27bcd0bb6e9)

2. Modificar la configuracion de nginx que utilice ssl, el puerto 443 , los certificados e incluye un redirect por si el usuario ingresa por http [ver commit](https://github.com/UNAHUR-OPE1/iot/commit/31528b74d146a9dc48c901927ad4c693b5d3655f#diff-eb810c54827c4ed2c2e3c2ca55d380882f21dda070517e73a95231a084344f5b)
   
   
   
   

> Se puede observar que la implementación de TLS solo modifica al proxy. Esto es una gran ventaja per recuerden que un proxy es un único punto de falla. Falla el proxy, falla todo lo que esta detrás.
> {: .prompt-info }

## Fuentes de información

- [Apache: Forward and reverse proxy](https://httpd.apache.org/docs/2.0/mod/mod_proxy.html#forwardreverse)

- [Docker: Network Drivers](https://docs.docker.com/network/drivers/)

- [Docker: Networking in Compose](https://docs.docker.com/compose/networking/)

- [NGINX Begginers Guide](http://nginx.org/en/docs/beginners_guide.html)