# AE6 - DNS y servicio de directorio para el despliegue

# Índice

## Paso 1 - Comprobar el entorno y la aplicación

```bash
cd ~/dpl/ae4 && docker compose up -d
docker compose ps 
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' dpl-nginx dpl-django
```

```bash
docker start dpl-lab
docker ps                     # dpl-lab, dpl-cliente y los contenedores de la app (dpl-nginx, dpl-django)
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dpl-lab
```

Posibles errores:
* posible conflicto con puertos, tal y como estaba configurada dpl-lab y compose.yml, el contenedor dpl-lab mapea el puerto 8000, y en el compose.yml, dpl-nginx mapea el puerto 8000. Para cambiarlo, cambia el puerto 8000 en el compose.yml por ejemplo por el 8001. Y ejecuta los siguientes comandos:

    ```bash
    docker compose down
    docker compose up -d
    ```
* si se ejecuta `docker compose down` es probable que tengas que volver a conectar `dpl-lab` a la red `dpl-net`:
    ```bash
    docker network connect dpl-net dpl-lab
    ```

## Paso 2 — Instalar BIND9 en dpl-lab

La instalación se hace dentro del contenedor:
```bash
docker exec dpl-lab bash -c "apt-get update && apt-get install -y bind9"
docker exec dpl-lab named -v
```

## Paso 3 — Crear el fichero de zona dpl.local

Una vez preparado el entorno e instalado BIND9 en el contenedor, crea `/etc/bind/db.dpl.local` dentro de `dpl-lab` con el contenido siguiente (sustituye las IPs por las anotadas en el paso 1):

```
$TTL 3600
$ORIGIN dpl.local.
@       IN  SOA ns1.dpl.local. admin.dpl.local. (
        2024110501 ; serial
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400 )    ; minimum TTL
@       IN  NS  ns1.dpl.local.
ns1     IN  A   IP_DPL_LAB
www     IN  A   IP_NGINX
app     IN  A   IP_DJANGO
```

# Paso 4 — Declarar la zona y validar

En `/etc/bind/named.conf.local` añade:

```conf
zone "dpl.local" {
    type master;
    file "/etc/bind/db.dpl.local";
};
```

Valida la configuración y el fichero de zona:

```bash
docker exec dpl-lab named-checkconf
docker exec dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local
```
El resultado esperado: `named-checkconf` sin salida (no hay errores) y `named-checkzone` muestra `OK`. Si aparece un error de sintaxis, revisa los puntos finales de los nombres absolutos.

Salida:
```bash
docker exec -it dpl-lab named-checkconf


docker exec -it dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local
zone dpl.local/IN: loaded serial 2024110501
OK
```

## Paso 5 — Lanzar named sin systemd

Los contenedores no ejecutan systemd: named se lanza en foreground:

```bash
docker exec -d dpl-lab named -g
docker exec dpl-lab pgrep -a named
docker exec dpl-lab ss -lunp | grep ':53'
```

El resultado esperado: pgrep muestra el proceso named y ss el puerto 53 (UDP y TCP) escuchando.

Salidas:
```bash
docker exec -it dpl-lab pgrep -a named

65 /usr/sbin/named -u bind

docker exec -it dpl-lab ss -lunp | grep :53

UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
```

> Si reinicias `dpl-lab`, relanza `docker exec -d dpl-lab named -g`.

## Paso 6 — Apuntar el cliente a tu DNS y a la red de la aplicación

En `dpl-cliente`:

```bash
# Herramientas de consulta (si no están): dnsutils (dig/nslookup/host) e iputils-ping (ping)
docker exec dpl-cliente bash -c "apt-get update && apt-get install -y dnsutils iputils-ping"

# Resolvedor: apuntar a tu BIND9 y anexar el dominio en búsquedas cortas.
# El segundo nameserver es un respaldo para nombres de Internet: al sustituir
# el DNS interno de Docker (127.0.0.11), si tu BIND9 no resuelve nombres
# externos (p. ej. en Docker Desktop devuelve SERVFAIL), sin respaldo se
# pierden apt/curl contra Internet.
docker exec dpl-cliente bash -c "echo 'nameserver IP_DPL_LAB' > /etc/resolv.conf && echo 'nameserver 1.1.1.1' >> /etc/resolv.conf && echo 'search dpl.local' >> /etc/resolv.conf"
docker exec dpl-cliente cat /etc/resolv.conf
```

Para que `dpl-cliente` alcance por red a los contenedores de la aplicación (que están en la red del proyecto Compose, no en dpl-net), conéctalo a esa red:

```bash
docker network ls                    # busca la red del proyecto (p. ej. dpl-app_default)
docker network connect dpl-app_default dpl-cliente
```

## Paso 7 — Comprobar la resolución desde el cliente

```bash
docker exec dpl-cliente nslookup www.dpl.local
docker exec dpl-cliente dig @IP_DPL_LAB app.dpl.local A +noall +answer
docker exec dpl-cliente host -t a www.dpl.local
docker exec dpl-cliente host -t ns dpl.local
docker exec dpl-cliente ping -c 3 www.dpl.local
docker exec dpl-cliente curl -I http://www.dpl.local
docker exec dpl-cliente curl -s http://app.dpl.local:8000/api/mensaje
```

Salida:

```bash
Server:		172.20.0.2
Address:	172.20.0.2#53

Name:	www.dpl.local
Address: 172.23.0.5

app.dpl.local.		3600	IN	A	172.23.0.3
www.dpl.local has address 172.23.0.5
dpl.local name server ns1.dpl.local.
PING www.dpl.local (172.23.0.5) 56(84) bytes of data.
64 bytes from 172.23.0.5: icmp_seq=1 ttl=64 time=0.030 ms
64 bytes from 172.23.0.5: icmp_seq=2 ttl=64 time=0.172 ms
64 bytes from 172.23.0.5: icmp_seq=3 ttl=64 time=0.179 ms

--- www.dpl.local ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2091ms
rtt min/avg/max/mdev = 0.030/0.127/0.179/0.068 ms
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.27.5
Date: Fri, 06 Nov 2026 12:12:06 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive

{"mensaje": "Hola desde el servidor de aplicaciones DPL (UT4)", "hostname": "3890eb1899ce", "fecha": "2026-11-06T13:12:06"}%                     
```

## Verificación

```bash
# Resolución directa contra tu DNS (siempre, para saltar la caché)
docker exec dpl-cliente dig @IP_DPL_LAB www.dpl.local A +noall +answer
docker exec dpl-cliente dig @IP_DPL_LAB app.dpl.local A +noall +answer

# El cliente usa tu DNS por defecto (no /etc/hosts)
docker exec dpl-cliente nslookup -debug www.dpl.local | head -20
docker exec dpl-cliente cat /etc/hosts        # no debe contener dpl.local

# Servicios por nombre
docker exec dpl-cliente ping -c 2 www.dpl.local
docker exec dpl-cliente curl -I http://www.dpl.local
docker exec dpl-cliente curl -s http://www.dpl.local/api/mensaje    # vía proxy nginx -> django
```

Salida:

```
www.dpl.local.		3600	IN	A	172.23.0.5
app.dpl.local.		3600	IN	A	172.23.0.3
Server:		172.20.0.2
Address:	172.20.0.2#53

------------
    QUESTIONS:
	www.dpl.local, type = A, class = IN
    ANSWERS:
    ->  www.dpl.local
	internet address = 172.23.0.5
	ttl = 3600
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
Name:	www.dpl.local
Address: 172.23.0.5
------------
    QUESTIONS:
	www.dpl.local, type = AAAA, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::	ip6-localnet
ff00::	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.20.0.3	ae47235b11dd
172.17.0.2	ae47235b11dd
172.23.0.6	ae47235b11dd
PING www.dpl.local (172.23.0.5) 56(84) bytes of data.
64 bytes from 172.23.0.5: icmp_seq=1 ttl=64 time=0.034 ms
64 bytes from 172.23.0.5: icmp_seq=2 ttl=64 time=0.184 ms

--- www.dpl.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1015ms
rtt min/avg/max/mdev = 0.034/0.109/0.184/0.075 ms
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.27.5
Date: Fri, 06 Nov 2026 12:13:07 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive

{"mensaje": "Hola desde el servidor de aplicaciones DPL (UT4)", "hostname": "3890eb1899ce", "fecha": "2026-11-06T13:13:07"}%                     
```

# Verificables

* `named` está activo (`pgrep -a named`, `ss -lunp | grep :53`) y la zona `dpl.local` es `type master`; `named-checkconf` y `named-checkzone dpl.local /etc/bind/db.dpl.local` terminan en OK.

Comandos y salidas: 
```bash
docker exec -it dpl-lab pgrep -a named
docker exec -it dpl-lab ss -lunp | grep :53
docker exec -it dpl-lab named-checkconf
docker exec -it dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local

65 /usr/sbin/named -u bind

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug dpl-lab
    Learn more at https://docs.docker.com/go/debug-cli/

UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.23.0.6:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                              172.20.0.2:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                               127.0.0.1:53                               0.0.0.0:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                
UNCONN           0                0                                   [::1]:53                                  [::]:*                                

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug dpl-lab
    Learn more at https://docs.docker.com/go/debug-cli/
zone dpl.local/IN: loaded serial 2024110501
OK

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug dpl-lab
    Learn more at https://docs.docker.com/go/debug-cli/
```


* La zona incluye SOA, NS y los registros **A** de `www` (servidor Web nginx) y `app` (servidor de aplicaciones).

```bash
docker exec -it dpl-lab cat /etc/bind/db.dpl.local

$TTL 3600
$ORIGIN dpl.local.
@       IN  SOA ns1.dpl.local. admin.dpl.local. (
        2024110501 ; serial
        3600       ; refresh
        900        ; retry
        604800     ; expire
        86400 )    ; minimum TTL
@       IN  NS  ns1.dpl.local.
ns1     IN  A   172.20.0.2
www     IN  A   172.23.0.5
app     IN  A   172.23.0.3

What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug dpl-lab
    Learn more at https://docs.docker.com/go/debug-cli/
```

