# AE6 - DNS y servicio de directorio para el despliegue

# Índice

- [DNS y resolución de nombres](#dns-y-resolución-de-nombres)
- [Servicio de directorio](#servicio-de-directorio)
- [Verificables](#verificables)
  - [1. named está activo y validación de la zona](#1-named-está-activo-y-validación-de-la-zona)
  - [2. Estructura y registros de la zona DNS (SOA, NS y registros A)](#2-estructura-y-registros-de-la-zona-dns-soa-ns-y-registros-a)
  - [3. Resolución de nombres y conectividad desde el cliente sin /etc/hosts](#3-resolución-de-nombres-y-conectividad-desde-el-cliente-sin-etchosts)
  - [4. Validación de peticiones HTTP por nombre de dominio](#4-validación-de-peticiones-http-por-nombre-de-dominio)
  - [5. Servicio OpenLDAP activo y estructura base del directorio LDAP](#5-servicio-openldap-activo-y-estructura-base-del-directorio-ldap)
  - [6. Validación de credenciales y control de errores con ldapwhoami](#6-validación-de-credenciales-y-control-de-errores-con-ldapwhoami)
  - [7. Verificación del contenido del directorio con slapcat y ldapsearch](#7-verificación-del-contenido-del-directorio-con-slapcat-y-ldapsearch)
  - [8. Integración de la aplicación web con el servicio de directorio](#8-integración-de-la-aplicación-web-con-el-servicio-de-directorio)
  - [9. El informe de diagnóstico](#9-el-informe-de-diagnóstico)
  


# DNS y resolución de nombres
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

salida de terminal:
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

## Verificación - DNS

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
Name:	www.dpl.local# Servicio y puerto (sin systemd)
docker exec dpl-lab pgrep -a slapd
docker exec dpl-lab ss -lntp | grep ':389'

# Volcado directo del directorio y árbol visto por red desde el cliente
docker exec dpl-lab slapcat
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://172.20.0.2 -b "dc=dpl,dc=local" "objectClass=*" dn

# Autenticación real (debe confirmar el DN)
docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.20.0.2 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -W

# Autenticación desde una aplicación (ldap3): correcta e incorrecta
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno CAMBIAR_ESTO
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno contrasena_incorrecta

# Autenticación integrada en la aplicación Web (con ~/dpl/ae4 desplegado y LDAP_HOST exportado)
curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://localhost:8001/api/login
curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://localhost:8001/api/login
99 slapd -d 0 -h ldap:///
LISTEN 0      2048         0.0.0.0:389        0.0.0.0:*    users:(("slapd",pid=99,fd=6))
LISTEN 0      2048            [::]:389           [::]:*    users:(("slapd",pid=99,fd=7))
dn: dc=dpl,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: DPL
dc: dpl
structuralObjectClass: organization
entryUUID: 4774a6d0-6ef9-1041-91e6-ed43b28b1020
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107131708Z
entryCSN: 20261107131708.593179Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107131708Z

dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios
structuralObjectClass: organizationalUnit
entryUUID: e415c8c2-6efc-1041-9db2-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.863868Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local
userPassword:: Q0FNQklBUl9FU1RP
structuralObjectClass: inetOrgPerson
entryUUID: e4165454-6efc-1041-9db3-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.867516Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local
userPassword:: Q0FNQklBUl9FU1RP
structuralObjectClass: inetOrgPerson
entryUUID: e416b08e-6efc-1041-9db4-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.869878Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: dc=dpl,dc=local

dn: ou=usuarios,dc=dpl,dc=local

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local

dn: uid=maria,ou=usuarios,dc=dpl,dc=local

Enter LDAP Password: 
dn:uid=alumno,ou=usuarios,dc=dpl,dc=local

Usuario alumno autenticado
cn: alumno
mail: alumno@dpl.local
Acceso denegado: automatic bind not successful - invalidCredentials
{"autenticado": true, "usuario": "alumno", "perfil": {"cn": "alumno", "mail": "alumno@dpl.local"}}401v
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

# Servicio de directorio

## Paso 1 — Instalar el directorio dentro de dpl-lab

Dentro de `dpl-lab`:

```bash
docker exec dpl-lab bash -c "
  echo 'slapd slapd/domain string dpl.local' | debconf-set-selections &&
  echo 'slapd shared/organization string DPL' | debconf-set-selections &&
  echo 'slapd slapd/password1 password CAMBIAR_ESTO' | debconf-set-selections &&
  echo 'slapd slapd/password2 password CAMBIAR_ESTO' | debconf-set-selections &&
  echo 'slapd slapd/backend select MDB' | debconf-set-selections &&
  echo 'slapd slapd/purge_database boolean false' | debconf-set-selections &&
  echo 'slapd slapd/move_old_database boolean true' | debconf-set-selections &&
  apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y slapd ldap-utils"
```

## Paso 2 — Lanzar slapd sin systemd y verificar la base

```bash
docker exec -d dpl-lab slapd -d 0 -h ldap:///
docker exec dpl-lab pgrep -a slapd
docker exec dpl-lab ss -lntp | grep ':389'
```

Comprueba que la base se creó:

```bash
docker exec dpl-lab ldapsearch -x -LLL -b "dc=dpl,dc=local" "objectClass=*" dn
```

> Si se reinicia `dpl-lab`, hay que relanzar `docker exec -d dpl-lab slapd -d 0 -h ldap:///`. Ya que el arranque automático con systemd no es comprobable en el contenedor (no probado, sin systemd en contenedor).

## Paso 3 — Preparar el cliente LDAP

```bash
docker exec dpl-cliente bash -c "apt-get update && apt-get install -y ldap-utils"
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://IP_DPL_LAB -b "dc=dpl,dc=local" "objectClass=*" dn
```

## Paso 4 — Crear la unidad organizativa y los usuarios (LDIF)

```bash
mkdir -p ~/dpl/ae6/ldif
cd ~/dpl/ae6
git init
cat > ldif/usuarios.ldif <<'EOF'
dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local
userPassword: CAMBIAR_ESTO

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local
userPassword: CAMBIAR_ESTO
EOF
git add ldif/usuarios.ldif
git commit -m "Crear el LDIF de usuarios" -m "ou=usuarios y los usuarios alumno y maria con contrasenas de laboratorio (CAMBIAR_ESTO)."
```

Se copia a `dpl-cliente` y se carga con el administrador:

```bash
docker cp ~/dpl/directorio-ut6/ldif/usuarios.ldif dpl-cliente:/tmp/usuarios.ldif
docker exec -it dpl-cliente ldapadd -x -H ldap://IP_DPL_LAB -D "cn=admin,dc=dpl,dc=local" -W -f /tmp/usuarios.ldif
```

## Paso 5 — Autenticar contra el directorio

Desde `dpl-cliente`, comprueba las credenciales de `alumno` y `maria`:

```bash
docker exec -it dpl-cliente ldapwhoami -x -H ldap://IP_DPL_LAB -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -W
docker exec -it dpl-cliente ldapwhoami -x -H ldap://IP_DPL_LAB -D "uid=maria,ou=usuarios,dc=dpl,dc=local" -W
```

## Paso 6 — Consultar la entrada (equivalente a "buscar quién es")

```bash
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://IP_DPL_LAB -b "ou=usuarios,dc=dpl,dc=local" "(uid=alumno)" cn mail
```

## Paso 7 — Autenticar la aplicación Web contra el directorio

La aplicación Web de ejemplo (la de la UT4) trae el endpoint /api/login, que hace el bind LDAP con el usuario y la contraseña recibidos. Se activa con variables de entorno, sin tocar el código:

```bash
export LDAP_HOST="ldap://IP_DPL_LAB"     # URL del directorio (obligatoria)
export LDAP_BASE_DN="ou=usuarios,dc=dpl,dc=local"
export LDAP_USER_FILTER="(uid=%s)"
```

* `LDAP_HOST`: dónde está el directorio (URL; el puerto 389 es el de LDAP). Si no se define, /api/login responde 503 y la app funciona sin directorio.
* `LDAP_BASE_DN`: base de búsqueda en el árbol.
* `LDAP_USER_FILTER`: filtro que localiza al usuario por su uid (el %s se sustituye por el nombre que escribe la persona).

Para que el contenedor de la aplicación alcance a `dpl-lab`, ambos deben compartir red; se conecta dpl-lab a la red del proyecto Compose (o la app a dpl-net):

```bash
docker network ls                         
docker network connect dpl-app_default dpl-lab
cd ~/dpl/ae4 && docker compose up -d     
```

Arranca el despliegue con la variable exportada y prueba el login (correcto → 200; incorrecto → 401):

```bash
curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://localhost:8001/api/login
curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://localhost:8001/api/login
```

## Paso 8 — Volcar el directorio con slapcat

`slapcat` exporta la base directamente desde el servidor (sin pasar por la red) en formato LDIF; sirve para verificar la configuración y como copia de seguridad:

```bash
docker exec dpl-lab slapcat
# dn: dc=dpl,dc=local
# objectClass: top
# objectClass: dcObject
# objectClass: organization
# o: DPL
# dc: dpl
```

## Paso 9 — Validar la autenticación con un cliente Python (ldap3)

```bash
docker exec dpl-cliente bash -c "apt-get install -y python3-pip && pip install ldap3 --break-system-packages"
docker exec dpl-cliente bash -c "cat > /tmp/comprobar_acceso.py <<'EOF'
import sys
from ldap3 import Connection
from ldap3.core.exceptions import LDAPException

LDAP_SERVER = 'ldap://IP_DPL_LAB'
BASE_DN = 'ou=usuarios,dc=dpl,dc=local'
USER_ATTRIBUTES = ['cn', 'mail']

username, password = sys.argv[1], sys.argv[2]
user_dn = f'uid={username},{BASE_DN}'

try:
    conn = Connection(LDAP_SERVER, user=user_dn, password=password, auto_bind=True)
except LDAPException as err:
    print(f'Acceso denegado: {err}')
else:
    print(f'Usuario {username} autenticado')
    conn.search(BASE_DN, f'(uid={username})', attributes=USER_ATTRIBUTES)
    print(f'cn: {conn.entries[0].cn}')
    print(f'mail: {conn.entries[0].mail}')
EOF"
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno CAMBIAR_ESTO          # autentica
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno contrasena_incorrecta # deniega
```

## Verificación - Servicio de directorio

Comandos y salida de terminal:

```bash
# Servicio y puerto (sin systemd)
docker exec dpl-lab pgrep -a slapd
docker exec dpl-lab ss -lntp | grep ':389'

# Volcado directo del directorio y árbol visto por red desde el cliente
docker exec dpl-lab slapcat
docker exec dpl-cliente ldapsearch -x -LLL -H ldap://172.20.0.2 -b "dc=dpl,dc=local" "objectClass=*" dn

# Autenticación real (debe confirmar el DN)
docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.20.0.2 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -W

# Autenticación desde una aplicación (ldap3): correcta e incorrecta
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno CAMBIAR_ESTO
docker exec dpl-cliente python3 /tmp/comprobar_acceso.py alumno contrasena_incorrecta

# Autenticación integrada en la aplicación Web (con ~/dpl/ae4 desplegado y LDAP_HOST exportado)
curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://localhost:8001/api/login
curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://localhost:8001/api/login
99 slapd -d 0 -h ldap:///
LISTEN 0      2048         0.0.0.0:389        0.0.0.0:*    users:(("slapd",pid=99,fd=6))
LISTEN 0      2048            [::]:389           [::]:*    users:(("slapd",pid=99,fd=7))
dn: dc=dpl,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: DPL
dc: dpl
structuralObjectClass: organization
entryUUID: 4774a6d0-6ef9-1041-91e6-ed43b28b1020
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107131708Z
entryCSN: 20261107131708.593179Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107131708Z

dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios
structuralObjectClass: organizationalUnit
entryUUID: e415c8c2-6efc-1041-9db2-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.863868Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local
userPassword:: Q0FNQklBUl9FU1RP
structuralObjectClass: inetOrgPerson
entryUUID: e4165454-6efc-1041-9db3-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.867516Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local
userPassword:: Q0FNQklBUl9FU1RP
structuralObjectClass: inetOrgPerson
entryUUID: e416b08e-6efc-1041-9db4-ab1e5e981393
creatorsName: cn=admin,dc=dpl,dc=local
createTimestamp: 20261107134259Z
entryCSN: 20261107134259.869878Z#000000#000#000000
modifiersName: cn=admin,dc=dpl,dc=local
modifyTimestamp: 20261107134259Z

dn: dc=dpl,dc=local

dn: ou=usuarios,dc=dpl,dc=local

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local

dn: uid=maria,ou=usuarios,dc=dpl,dc=local

Enter LDAP Password: 
dn:uid=alumno,ou=usuarios,dc=dpl,dc=local

Usuario alumno autenticado
cn: alumno
mail: alumno@dpl.local
Acceso denegado: automatic bind not successful - invalidCredentials
{"autenticado": true, "usuario": "alumno", "perfil": {"cn": "alumno", "mail": "alumno@dpl.local"}}401
```

# Verificables

### 1. `named` está activo y validación de la zona

* `named` está activo (`pgrep -a named`, `ss -lunp | grep :53`) 
* la zona `dpl.local` es `type master`; `named-checkconf` y `named-checkzone dpl.local /etc/bind/db.dpl.local` terminan en OK.

Comandos y salida de terminal: 
```bash
docker exec -it dpl-lab pgrep -a named
docker exec -it dpl-lab ss -lunp | grep :53
docker exec -it dpl-lab named-checkconf
docker exec -it dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local

65 /usr/sbin/named -u bind

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

zone dpl.local/IN: loaded serial 2024110501
OK
```
* `pgrep -a named`: Busca en los procesos activos del contenedor si el demonio del servidor DNS (named) está corriendo.
* `ss -lunp | grep :53`: Muestra las estadísticas de sockets de red (ss), filtrando las conexiones UDP (-u), en escucha (-l), mostrando los números de puerto (-n) y el proceso asociado (-p), específicamente en el puerto 53 (el puerto estándar de DNS).
* `named-checkconf`: Analiza los archivos de configuración general de BIND9 para verificar que la sintaxis es correcta y no hay errores tipográficos.
* `named-checkzone` dpl.local /etc/bind/db.dpl.local: Valida que el archivo de zona específico para el dominio dpl.local esté perfectamente estructurado.

### 2. Estructura y registros de la zona DNS (SOA, NS y registros A)

LA zona incluye SOA, NS y los registros **A** de `www` (servidor Web nginx) y `app` (servidor de aplicaciones).

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
```

El comando `cat /etc/bind/db.dpl.local` muestra el contenido completo del fichero de zona de BIND9 para el dominio dpl.local, permite comprobar que están declarados los registros del inicio de autoridad (SOA), los servidores de nombres (NS) y las IPs asociadas a www y app.

### 3. Resolución de nombres y conectividad desde el cliente sin /etc/hosts

* Desde `dpl-cliente` sin `/etc/hosts`: `nslookup www.dpl.local`, `dig @IP_DPL_LAB app.dpl.local A`, `host -t a www.dpl.local` y `ping -c 3 www.dpl.local` funcionan.

Comandos y salida de terminal:
```bash
docker exec dpl-cliente nslookup www.dpl.local
docker exec dpl-cliente dig @172.20.0.2 app.dpl.local A +short
docker exec dpl-cliente host -t a www.dpl.local
docker exec dpl-cliente ping -c 3 www.dpl.local

Server:		172.20.0.2
Address:	172.20.0.2#53

Name:	www.dpl.local
Address: 172.23.0.5

172.23.0.3
www.dpl.local has address 172.23.0.5
PING www.dpl.local (172.23.0.5) 56(84) bytes of data.
64 bytes from 172.23.0.5: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 172.23.0.5: icmp_seq=2 ttl=64 time=0.169 ms
64 bytes from 172.23.0.5: icmp_seq=3 ttl=64 time=0.176 ms

--- www.dpl.local ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.041/0.128/0.176/0.062 ms
```

* `nslookup www.dpl.local`: Consulta al servidor DNS configurado en el contenedor cliente para resolver el nombre a su dirección IP.
* `dig @172.20.0.2 app.dpl.local A +short`: Realiza una consulta DNS directa (dig) preguntando explícitamente al servidor DNS de laboratorio (172.20.0.2) por el registro tipo A de app.dpl.local, mostrando solo la respuesta limpia (+short).
* `host -t a www.dpl.local`: Utiliza la utilidad host para buscar específicamente registros de tipo dirección (-t a) del dominio indicado.
* `ping -c 3 www.dpl.local`: Envía 3 paquetes ICMP (-c 3) al dominio www.dpl.local para comprobar que, además de resolver el nombre por DNS, hay conectividad de red activa con la IP destino.

### 4. Validación de peticiones HTTP por nombre de dominio

* `curl -I http://www.dpl.local` y una petición a la aplicación por nombre llegan a los servicios.

Comandos y salida de terminal:
```bash
docker exec dpl-cliente curl -I http://www.dpl.local
docker exec dpl-cliente curl -s http://app.dpl.local/

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.27.5
Date: Sun, 08 Nov 2026 09:39:42 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/login?next=/">/login?next=/</a>. If not, click the link.
```

* `curl -I http://www.dpl.local`: Solicita únicamente las cabeceras HTTP (-I) al servidor web usando su nombre de dominio, verificando que Nginx responde correctamente (devolviendo el estado 200 OK).
* `curl -s http://app.dpl.local/` Realiza una petición silenciosa (-s) al subdominio de la aplicación web, comprobando que el tráfico cruza el proxy inverso y alcanza el backend.


### 5. Servicio OpenLDAP activo y estructura base del directorio LDAP

* `slapd` está activo (`pgrep -a slapd`, `ss -lntp | grep :389`) 
* la base `dc=dpl,dc=local` existe con `ou=usuarios` y las entradas del LDIF entregado.

Comandos y salida de terminal:

```bash
docker exec dpl-lab pgrep -a slapd
docker exec dpl-lab ss -lntp | grep ':389'

99 slapd -d 0 -h ldap:///
LISTEN 0      2048         0.0.0.0:389        0.0.0.0:*    users:(("slapd",pid=99,fd=6))
LISTEN 0      2048            [::]:389           [::]:*    users:(("slapd",pid=99,fd=7))
```

* `ldif`:

```bash
cat ldif/usuarios.ldif

dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios

dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local
userPassword: CAMBIAR_ESTO

dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local
userPassword: CAMBIAR_ESTO
```

### 6. Validación de credenciales y control de errores con ldapwhoami

* `ldapwhoami` con las credenciales correctas confirma el DN del usuario y con una contraseña incorrecta falla con error 49.

Comandos y salida de terminal:

```bash
docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.20.0.2 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -w "CAMBIAR_ESTO"
docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.20.0.2 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -w "contrasena_incorrecta"

dn:uid=alumno,ou=usuarios,dc=dpl,dc=local
ldap_bind: Invalid credentials (49)
```

### 7. Verificación del contenido del directorio con slapcat y ldapsearch

* `slapcat` (o `ldapsearch`) demuestra el contenido del directorio.

Comandos y salida de terminal:

```bash
docker exec -it dpl-lab ldapsearch -x -H ldap://localhost -b "ou=usuarios,dc=dpl,dc=local"

# extended LDIF
#
# LDAPv3
# base <ou=usuarios,dc=dpl,dc=local> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# usuarios, dpl.local
dn: ou=usuarios,dc=dpl,dc=local
objectClass: organizationalUnit
ou: usuarios

# maria, usuarios, dpl.local
dn: uid=maria,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: maria
cn: maria
sn: Maria DPL
mail: maria@dpl.local

# alumno, usuarios, dpl.local
dn: uid=alumno,ou=usuarios,dc=dpl,dc=local
objectClass: inetOrgPerson
uid: alumno
cn: alumno
sn: Alumno DPL
mail: alumno@dpl.local

# search result
search: 2
result: 0 Success

# numResponses: 4
# numEntries: 3
```

### 8. Integración de la aplicación web con el servicio de directorio

La aplicación está configurada para autenticar contra el directorio (host, base DN, filtro y bind) y el usuario del directorio puede iniciar sesión.

Comandos y salida de terminal:

```bash
docker exec dpl-cliente curl -s -X POST -d "usuario=alumno&password=CAMBIAR_ESTO" http://www.dpl.local/api/login
docker exec dpl-cliente curl -s -o /dev/null -w '%{http_code}\n' -X POST -d "usuario=alumno&password=incorrecta" http://www.dpl.local/api/login

{"autenticado": true, "usuario": "alumno", "perfil": {"cn": "alumno", "mail": "alumno@dpl.local"}}
401
```

### 9. El informe de diagnóstico 

**El informe de diagnóstico** localiza fallos en las cinco capas (DNS, puerto, servidor Web, servidor de aplicaciones y autenticación) con la estructura síntoma / causa / comprobación / corrección / verificación.

|Capa analizada|	Sintoma observado|Causa raíz técnica|	Comando o acción de comprobación	|Correción aplicada|Verificación del estado OK|
|--|--|--|--|--|--|

|1. DNS (Resolución de Nombres) | Error del tipo Could not resolve host o NXDOMAIN al usar dominios.|El contenedor cliente no consultaba al servidor BIND9 del laboratorio (172.20.0.2), o el archivo de zona tenía errores sintácticos.|docker exec dpl-cliente nslookup www.dpl.local
docker exec -it dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local.| Corregir registros en el fichero de zona de BIND9 y asegurar el correcto direccionamiento del nameserver. | named-checkzone termina en OK y nslookup resuelve la IP de Nginx correctamente.|

2. Red / Puerto	docker network inspect	Conectividad activa en la red puente de Docker (dpl-net).	Enrutamiento operativo entre los contenedores de la infraestructura.
3. Servidor Web	docker logs dpl-nginx	Peticiones HTTP registradas con códigos 200 OK y redirecciones a /login.	Nginx recibe el tráfico por nombre de dominio y hace proxy inverso hacia el backend.
4. Aplicación	docker logs dpl-django	Trazas de Gunicorn mostrando peticiones a /healthz (200) y procesamiento de /api/login.	El servidor de aplicaciones gestiona correctamente las rutas y se comunica con el directorio.
5. Directorio LDAP	slapd -d 0 / Logs de app	Error [Errno 111] Connection refused con pkill slapd, y éxito o error 49 en los binds.	Dependencia crítica del servicio de directorio para la validación de credenciales (inetOrgPerson).

| Capa analizada | Síntoma observado | Causa raíz técnica | Comando o acción de comprobación | Corrección aplicada | Verificación del estado OK |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. DNS** *(Resolución de Nombres)* | Error del tipo *Could not resolve host* o NXDOMAIN al usar dominios. | El contenedor cliente no consultaba al servidor BIND9 del laboratorio (`172.20.0.2`), o el archivo de zona tenía errores sintácticos. | `docker exec dpl-cliente nslookup www.dpl.local`<br>`docker exec -it dpl-lab named-checkzone dpl.local /etc/bind/db.dpl.local` | Corregir registros en el fichero de zona de BIND9 y asegurar el correcto direccionamiento del nameserver. | `named-checkzone` termina en **OK** y `nslookup` resuelve la IP de Nginx correctamente. |
| **2. Puerto / Red** *(Conectividad de Red y Sockets)* | Error de socket de red del tipo `[Errno 111] Connection refused` al intentar conectar al puerto del servicio. | El contenedor de servicio estaba detenido, el demonio `slapd` no escuchaba en el puerto, o la red Docker (`dpl-net`) estaba desconectada. | `docker exec dpl-lab pgrep -a slapd`<br>`docker exec dpl-lab ss -lntp \| grep :389` | Iniciar de nuevo el demonio OpenLDAP (`slapd -d 0 ...`) o revisar el enrutamiento de la red puente. | El comando `ss` confirma que el servicio escucha de nuevo en el puerto TCP `389` y la app restablece la conexión. |
| **3. Servidor Web** *(Proxy Inverso - Nginx)* | Código de estado erróneo en las peticiones HTTP al dominio, como un `502 Bad Gateway` o un `404 Not Found`. | Nginx no enrutaba correctamente el tráfico hacia el *upstream* del backend, o el proxy carecía de conectividad con la aplicación. | `docker logs dpl-nginx` | Revisar los bloques `proxy_pass` en el archivo de configuración de Nginx (`default.conf`) y reiniciar el contenedor proxy. | `docker exec dpl-cliente curl -I http://www.dpl.local` devuelve **`HTTP/1.1 200 OK`**. |
| **4. Servidor de Aplicaciones** *(Backend - Django / Gunicorn)* | Error interno de servidor (`HTTP 500 Internal Server Error`) o la API no responde a las peticiones POST de login. | Excepción interna en el código, variables de entorno de conexión LDAP desconfiguradas o caída de Gunicorn. | `docker logs dpl-django` | Ajustar las variables de entorno (`LDAP_HOST`, `LDAP_BASE_DN`) en la configuración del servicio y reiniciar el backend. | El contenedor responde correctamente a las peticiones de salud (`/healthz` con código `200`) y procesa la API. |
| **5. Autenticación** *(Directorio OpenLDAP)* | Rechazo del inicio de sesión devolviendo estado `autenticado: false`, código HTTP `401` o error LDAP `49`. | Servicio LDAP caído o discrepancia en las credenciales proporcionadas frente a los registros del esquema `inetOrgPerson`. | `docker exec -it dpl-cliente ldapwhoami -x -H ldap://172.20.0.2 -D "uid=alumno,ou=usuarios,dc=dpl,dc=local" -w "..."` | Comprobar el estado activo de `slapd` y alinear las credenciales válidas en los ficheros LDIF. | `ldapwhoami` con contraseña correcta devuelve el DN del usuario y con contraseña errónea devuelve el error **`Invalid credentials (49)`**. |
