# «Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni»

## 1. Инфраструктура.

Кластер будет развернут на базе  виртуальных машин Oracle Virtual box. Создано 3 VM:
- Node1 c Ip  - 192.168.31.171
- Node2 c Ip  - 192.168.31.172
- Node3 c Ip  - 192.168.31.173


## 2.  Настройка host на всех трех машинах.

- настройка host на всех трех машинах.
sudo tee /etc/hosts <<EOF
127.0.0.1       localhost
192.168.31.171  node1
192.168.31.172  node2
192.168.31.173  node3
EOF

## 3.  Установка и настрока Etcd на всех трех VM.

- sudo apt install etcd-server
- Конфигурация etcd для node1, для остальных node такая же только отредактировать ETCD_NAME и IP адреса.

```
sudo nano /etc/default/etcd

ETCD_NAME=node1
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER="node1=http://192.168.31.171:2380,node2=http://192.168.31.172:2380,node3=http://192.168.31.173:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="patroni-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.31.171:2379"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.31.171:2380"
```
- Установим клиента
```
sudo apt  install etcd-client 
```

- Проверяем статус  
```
kop@node1:~$ ETCDCTL_API=3 etcdctl \
  --endpoints=http://192.168.31.171:2379,http://192.168.31.172:2379,http://192.1                                                           68.31.173:2379 \
  endpoint health --cluster
http://192.168.31.172:2379 is healthy: successfully committed proposal: took = 2                                                           7.137449ms
http://192.168.31.173:2379 is healthy: successfully committed proposal: took = 3                                                           1.856763ms
http://192.168.31.171:2379 is healthy: successfully committed proposal: took = 3                                                           8.379372ms
```

## 5. Установка PostgreSQL 17
- На всех узлах
```
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt install postgresql-17 postgresql-client-17 -y
```

- Остановите и отключите службу:
```
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

## 5. Установка Patroni
```
При попытки установки получил ошибку:
kop@node1:~$ pip3 install patroni[etcd3]
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.

    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.

    If you wish to install a non-Debian packaged Python application,
    it may be easiest to use pipx install xyz, which will manage a
    virtual environment for you. Make sure you have pipx installed.

    See /usr/share/doc/python3.12/README.venv for more information.

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```
- Столкнулся с новым механизмом безопасности в Ubuntu 22.04+ (и Debian 12+), который блокирует глобальную установку Python-пакетов через pip, чтобы не повредить системные пакеты.

- Установки провел через создание виртуального окружения.

- Ставим модуль для создания виртуальных окружений
```
sudo apt install python3.12-venv
```
- Создаём каталог для Patroni.
```
sudo mkdir -p /opt/patroni 
```
- Передаём владение каталогом пользователю postgres
```
sudo chown postgres:postgres /opt/patroni
```
- Создаём виртуальное окружение от имени postgres.
```
sudo -u postgres python3 -m venv /opt/patroni/venv 
```
- Устанавливаем Patroni с поддержкой etcd3.
```
sudo -u postgres /opt/patroni/venv/bin/pip install 'patroni[etcd3]'
 ```
- Устанавливаем драйвер для работы Patroni с PostgreSQL.
```
sudo -u postgres /opt/patroni/venv/bin/pip install 'psycopg2-binary' 
```
- Создание конфигураций Patroni
- На Node1:
```
sudo mkdir -p /etc/patroni
sudo nano /etc/patroni/config.yml 
scope: postgres_cluster
namespace: /service/
name: node1

restapi:
  listen: 192.168.31.171:8008
  connect_address: 192.168.31.171:8008

etcd3:
  hosts: ["192.168.31.171:2379", "192.168.31.172:2379", "192.168.31.173:2379"]

bootstrap:
  method: initdb
  initdb:
  - encoding: UTF8
  - locale: en_US.UTF-8
  - data-checksums
  pg_hba:
  - host replication replicator 192.168.31.0/24 md5
  - host all all 192.168.31.0/24 md5
  - host all all 0.0.0.0/0 md5
  - host replication replicator 192.168.31.0/24 scram-sha-256
  - host postgres postgres 192.168.31.0/24 md5

  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: logical
        hot_standby: "on"
        wal_keep_size: 128MB
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        track_commit_timestamp: "off"
        archive_mode: "on"
        archive_timeout: 1800s
        archive_command: '/bin/true'
        shared_preload_libraries: 'pg_stat_statements'
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all

postgresql:
  listen: 192.168.31.171,127.0.0.1
  connect_address: 192.168.31.171:5432
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  authentication:
    replication:
      username: replicator
      password: replication_pass
    superuser:
      username: postgres
      password: postgres_pass
  parameters:
    unix_socket_directories: '/var/run/postgresql'

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: true
  nosync: false
```

- Удаляем папки с данными postgres: 
```
sudo rm -rf /var/lib/postgresql/17/main
sudo rm -rf /var/lib/postgresql/data
```
- Настройка прав доступа
```
sudo chown -R postgres:postgres /etc/patroni
sudo chmod 750 /etc/patroni
sudo chmod 640 /etc/patroni/config.yml
```

- Создание systemd сервиса для Patroni
```
sudo nano /etc/systemd/system/patroni.service

[Unit]
Description=High availability PostgreSQL Cluster - Patroni
After=syslog.target network.target etcd.service
Wants=etcd.service
ConditionPathExists=/etc/patroni/config.yml

[Service]
Type=simple
User=postgres
Group=postgres
Environment=PATH=/opt/patroni/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=PYTHONPATH=/opt/patroni/venv/lib/python3.12/site-packages
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/config.yml
ExecReload=/bin/kill -HUP \$MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30
LimitNOFILE=65536

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ReadWritePaths=/var/lib/postgresql /var/run/postgresql /var/log/postgresql

[Install]
WantedBy=multi-user.target
```

- Запуск Patroni кластера
```
sudo systemctl enable patroni
sudo systemctl start patroni
```

- Были проблемы почему-то не выбирался лидер, пришлось очистить ключи
```
Dec 04 16:47:02 node1 patroni[1623]: 2025-12-04 16:47:02,059 INFO: waiting for leader to bootstrap

etcdctl --endpoints=http://192.168.31.171:2379,http://192.168.31.172:2379,http://192.168.31.173:2379 \
  del /service/postgres_cluster --prefix
```

- На второй Node пошли ошибки
```
FATAL: no pg_hba.conf entry for replication connection from host "192.168.31.172", user "replicator", no encryption
означает, что PostgreSQL на node1 отклоняет попытку подключения для репликации, потому что в pg_hba.conf нет соответствующей строки, разрешающей подключение.
```
- Проблема была в том, что не прописал patroni.yml
```
  - host replication replicator 192.168.31.0/24 scram-sha-256
```
-и в  /etc/postgresql/17/main/pg_hba.conf это строчка соотвественно не прописалась

- Проверяем журнал
```
sudo journalctl -u patroni -n 50 --no-pager

2025-12-06 14:40:50,347 INFO: no action. I am (node2), a secondary, and following a leader (node1)
```

## 6. Установка PgBouncer
```
sudo apt install pgbouncer -y
```
- sudo nano [`/etc/pgbouncer/pgbouncer.ini`](pgbouncer/pgbouncer_node1.conf)
- sudo nano  [`/etc/pgbouncer/userlist.txt`](pgbouncer/userlist.txt)


## 7. Установка HAProxy
- sudo apt install -y haproxy
- sudo nano /etc/haproxy/haproxy.cfg
- sudo nano [`/etc/haproxy/haproxy.cfg`](haproxy/haproxy.cfg)

systemctl enable --now haproxy
sudo journalctl -u haproxy -n 20 --no-pager

