# Configuración de Patroni con etcd y HAProxy para cuatro VMs con Ubuntu.
#### Probado con Ubuntu 18 y Ubuntu 20, ediciones de servidor.

En las instrucciones siguientes, sustituir los elementos entre < > por los valores apropiados para el servidor en el que se está trabajando. Lo mismo se debe hacer con los nombres y dominios de DNS, si no corresponden con los sugeridos. El _cluster_ se llamará `cluster1`. Partir de una instalación de cuatro VMs Ubuntu 18 o 20.
Este documento asume que existen las siguientes cuatro máquinas dentro del dominio de DNS `lab.local`, sobre el que se debe tener control.

Servidor | Propósito | DNS | Dirección IP
---------|-----------|-----|--------
pg01 | Primer nodo de Patroni | pg01.lab.local | 10.100.100.11/24
pg02 | Segundo nodo de Patroni | pg02.lab.local | 10.100.100.12/24
etcd | Único nodo del administrador de configuraciones | etcd.lab.local | 10.100.100.41/24
haproxy | Balanceador de cargas sobre TCP | haproxy.lab.local | 10.100.100.51/24

### En haproxy, como el usuario root:
1. Instalar el balanceador de cargas.
```bash
apt install haproxy
```

### En pg01 y pg02, como el usuario root:
1. Instalar el motor de base de datos.
```bash
apt install postgresql
```
2. Detener el _cluster_ de BDD que se levanta por omisión.
```bash
systemctl stop postgresql@<version>-<cluster>
```
2. Eliminar el _cluster_, puesto que se va a usar Patroni para controlar la instancia local.
```bash
pg_dropcluster <version> main
```
3. Recargar la configuración de los servicios supervisados por systemd
```bash
systemctl daemon-reload
```
4. Instalar el machote de orquestación Patroni.
```bash
apt install patroni
```

### En etcd, como el usuario root:
1. Instalar el administrador de configuraciones.
```bash
apt install etcd
```
2. Respaldar la configuración inicial de etcd y sustituirla por la que se detalla. Si no se está utilizando DNS, sustituir 0.0.0.0 y etcd.lab.local por <10.100.100.41>
```bash
cp -v /etc/default/etcd /etc/default/etcd.bkup
cat > /etc/default/etcd <<FinArchivo
ETCD_NAME="etcd"
ETCD_DISCOVERY_SRV="lab.local"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380,http://127.0.0.1:7001"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd.lab.local:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd.lab.local:2379"
ETCD_INITIAL_CLUSTER_TOKEN="cluster1"
ETCD_INITIAL_CLUSTER_STATE="new"
FinArchivo

```
3. Reiniciar el servicio de etcd y confirmar que haya levantado correctamente.
```bash
systemctl restart etcd
systemctl status etcd
```
5. Comprobar con `etcdctl` el estado del _cluster_ de un solo miembro de etcd.
```bash
etcdctl member list
etcdctl cluster-health
etcdctl ls /
```

### En pg01, como el usuario root:
1. Crear la configuración de Patroni. Si no se está utilizando DNS, sustituir etcd.lab.local por <10.100.100.41>, pg01.lab.local por <10.100.100.11> y pg02.lab.local por <10.100.100.12>.
```bash
cat > /etc/patroni/config.yml <<FinArchivo
scope: cluster_1
namespace: /db/
name: pg01

restapi:
    listen: pg01.lab.local:8008
    connect_address: pg01.lab.local:8008

etcd:
    host: etcd.lab.local:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true

    initdb:
    - encoding: UTF8
    - data-checksums

    pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator pg01.lab.local/0 md5
    - host replication replicator pg02.lab.local/0 md5
    - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: pg01.lab.local:5432
    connect_address: pg01.lab.local:5432
    data_dir: /var/lib/postgresql/cluster_1/pg01/data
    bin_dir: /usr/lib/postgresql/<version>/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: <password>
        superuser:
            username: postgres
            password: <password>
    parameters:
        unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
FinArchivo

```
2. Arrancar el servicio de Patroni y confirmar su estado.
```bash
systemctl start patroni
systemctl status patroni
```
4. Agregar las siguientes variables al ambiente, para no tener que especificarlas cada vez que se ejecute patronictl. Si no se está utilizando DNS, sustituir etcd.lab.local por <10.100.100.41>
```bash
PATRONI_NAMESPACE=/db/
PATRONI_SCOPE=cluster_1
PATRONI_ETCD_URL="http://etcd.lab.local:2379/"
```
5. Confirmar el estado del _cluster_ de Patroni, actualmente solo con el nodo actual, como líder.
```bash
patronictl list
```

### En pg02, como el usuario root:
1. Crear la configuración de Patroni. Si no se está utilizando DNS, sustituir etcd.lab.local por <10.100.100.41>, pg01.lab.local por <10.100.100.11> y pg02.lab.local por <10.100.100.12>.
```bash
cat > /etc/patroni/config.yml <<FinArchivo
scope: cluster_1
namespace: /db/
name: pg01

restapi:
    listen: pg02.lab.local:8008
    connect_address: pg02.lab.local:8008

etcd:
    host: etcd.lab.local:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
            use_pg_rewind: true

    initdb:
    - encoding: UTF8
    - data-checksums

    pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator pg02.lab.local/0 md5
    - host replication replicator pg01.lab.local/0 md5
    - host all all 0.0.0.0/0 md5

    users:
        admin:
            password: admin
            options:
                - createrole
                - createdb

postgresql:
    listen: pg02.lab.local:5432
    connect_address: pg02.lab.local:5432
    data_dir: /var/lib/postgresql/cluster_1/pg02/data
    bin_dir: /usr/lib/postgresql/<version>/bin
    pgpass: /tmp/pgpass
    authentication:
        replication:
            username: replicator
            password: <password>
        superuser:
            username: postgres
            password: <password>
    parameters:
        unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
FinArchivo

```
2. Arrancar el servicio de Patroni y confirmar su estado.
```bash
systemctl start patroni
systemctl status patroni
```
4. Agregar las siguientes variables al ambiente, para no tener que especificarlas cada vez que se ejecute patronictl. Si no se está utilizando DNS, sustituir etcd.lab.local por <10.100.100.41>
```bash
PATRONI_NAMESPACE=/db/
PATRONI_SCOPE=cluster_1
PATRONI_ETCD_URL="http://etcd.lab.local:2379/"
```
5. Confirmar el estado del _cluster_ de Patroni, actualmente con dos nodos. El nodo presente es la réplica.
```bash
patronictl list
```

### En haproxy, como el usuario root:
1. Detener el balanceador de cargas.
```bash
systemctl stop haproxy
```
3. Respaldar la configuración inicial de haproxy y sustituirla por la que se detalla. Si no se está utilizando DNS, sustituir pg01.lab.local por <10.100.100.11> y pg02.lab.local por <10.100.100.12>.
```bash
cp -v /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bkup
cat > /etc/haproxy/haproxy.cfg <<FinArchivo
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg01 pg01.lab.local:5432 maxconn 100 check port 8008
    server pg02 pg02.lab.local:5432 maxconn 100 check port 8008

FinArchivo

```
4. Arrancar el servicio de haproxy y confirmar su estado.
```bash
systemctl start haproxy
systemctl status haproxy
```

#### Ejercicios sugeridos:
Una vez realizada la configuración según los pasos anteriores, se tiene un _cluster_ de PostgreSQL de dos nodos con un servidor de balanceo al frente. Lo siguiente sirve como práctica del funcionamiento y uso.
* Crear un usuario y una base de datos en haproxy.lab.local. Crear tablas y cargarle datos.
* Leer datos de los nodos pg01.lab.local y pg02.lab.local. (Debe funcionar)
* Insertar datos en el nodo réplica. (Debe fallar)
* Bajar el nodo líder, bajando el servicio de Patroni o reiniciando el nodo.
* Confirmar estado del clúster. (Usando `patronictl`).
* Insertar datos usando haproxy.lab.local.
* Subir el primer Patroni de nuevo.
* Confirmar estado y funcionamiento del _cluster_.
* Utilizar `etcdctl ls` y `etcdctl get` para entender el funcionamiento subyacente.
* Usar `patronictl` para realizar un cambio inmediato de maestro y confirmar que sirve.
* Usar `patronictl` para realizar un cambio calendarizado de maestro y confirmar que sirve.
