
# Практика обработки и визуализации данных

**Цель задания:** Развернуть в облаке систему для визуализации и обработки данных.

## Задачи
- Средствами командной строки приготовить виртуальную машину в Яндекс Облаке, подключиться к ней по протоколу ssh.
- На виртуальной машине средствами docker-compose или k8s запустить кластер обработки данных: postgresql, kafka.
- На виртуальной машине средствами docker-compose или k8s запустить кластер визуализации данных: prometheus, grafana.
- Проверить работу дашборда в Grafana.

## Развертывание кластера средствами docker-compose
### Create VPS with users settings

```bash
yc compute instance create \
--folder-id b1gn61susoertj6sijgb \
--name dkazanskii-docker \
--zone ru-central1-a \
--platform standard-v3 \
--cores 4 \
--memory 8 \
--core-fraction 20 \
--create-boot-disk size=20,image-folder-id=standard-images,image-family=ubuntu-2204-lts \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--metadata-from-file user-data="calccloud_homework_02.yaml"
```

> листинг файла конфигурации `calccloud_homework_02.yaml`  в Приложении 1, ниже.


![alt text](assets/virtualpc.png)

Результат работы: созданы и запущены в контейнере все необходимые приложения:

```
root@fhmil63g20f59c46o15v:/usr# docker ps 
CONTAINER ID   IMAGE                                          COMMAND                  CREATED          STATUS          PORTS                                                           NAMES
a18865d44c6a   confluentinc/cp-kafka:7.5.0                    "/etc/confluent/dock…"   20 minutes ago   Up 20 minutes   0.0.0.0:9092->9092/tcp                                          kafka
37d2ea6f0548   prom/prometheus:latest                         "/bin/prometheus --c…"   20 minutes ago   Up 20 minutes   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp                       prometheus
280209f5d204   confluentinc/cp-zookeeper:7.5.0                "/etc/confluent/dock…"   20 minutes ago   Up 20 minutes   2888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 3888/tcp   zookeeper
5d69fb047f11   postgres:15                                    "docker-entrypoint.s…"   20 minutes ago   Up 20 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                       postgres
f53543e4af4e   grafana/grafana:latest                         "/run.sh"                20 minutes ago   Up 20 minutes   0.0.0.0:3000->3000/tcp                                          grafana
e08b673454b0   prometheuscommunity/postgres-exporter:latest   "/bin/postgres_expor…"   20 minutes ago   Up 20 minutes   0.0.0.0:9187->9187/tcp, :::9187->9187/tcp                       postgres_exporter
```


### Grafana Dashboard - проверка работы

Скриншот 1. Метрики `prometheus`

![alt text](assets/prometheus_metrics.png)


Скриншот 2. Подключение `postgreSQL`

![alt text](assets/postgres_metrics.png)

## Развертывание кластера средствами `k8s`
### Создание кластера Managed Service for Kubernetes

```
yc managed-kubernetes cluster create \
--network-name default \
--zone ru-central1-a \
--subnet-name default-ru-central1-a \
--public-ip \
--service-account-id aje04s2i437pqvk9ti0d \
--node-service-account-id aje04s2i437pqvk9ti0d \
--cluster-ipv4-range 10.3.0.0/20 \
--service-ipv4-range 10.0.3.0/24
```

```
done (8m26s)
id: cat3jbqrplkjp2snrfff
folder_id: b1gn61susoertj6sijgb
...
master:
  zonal_master:
    zone_id: ru-central1-a
    internal_v4_address: 10.128.0.37
    external_v4_address: 158.160.48.168
...
  version: "1.27"
...
ip_allocation_policy:
  cluster_ipv4_cidr_block: 10.3.0.0/20
  node_ipv4_cidr_mask_size: "24"
  service_ipv4_cidr_block: 10.0.3.0/24
service_account_id: aje04s2i437pqvk9ti0d
node_service_account_id: aje04s2i437pqvk9ti0d
release_channel: REGULAR
```

### Создание группы узлов Managed Service for Kubernetes (группы ВМ)

```
yc managed-kubernetes node-group create \
--name k8s-dkazanskii-ng \
--cluster-id cat3jbqrplkjp2snrfff \
--platform standard-v3 \
--cores 2 --memory 4 --core-fraction 20 \
--disk-type network-ssd --network-interface subnets=default-ru-central1-a,ipv4-address=nat \
--async \
--auto-scale max=4
```

```
yc managed-kubernetes cluster list-nodes --id cat3jbqrplkjp2snrfff
+--------------------------------+---------------------------+--------------------------------+-------------+--------+
|         CLOUD INSTANCE         |      KUBERNETES NODE      |           RESOURCES            |    DISK     | STATUS |
+--------------------------------+---------------------------+--------------------------------+-------------+--------+
| fhmiea91pfmgpd0funf7           | cl1j698riecva6j51n9h-oqip | 2 20% core(s), 4.0 GB of       | 96.0 GB ssd | READY  |
| RUNNING_ACTUAL                 |                           | memory                         |             |        |
+--------------------------------+---------------------------+--------------------------------+-------------+--------+

```



### Развертывание кластера (применение конфигурации)

```
kubectl apply -f k8s-cluster.yaml
```

```

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/grafana      NodePort    10.0.3.179   <none>        3000:32000/TCP   58m
service/kafka        ClusterIP   10.0.3.220   <none>        9092/TCP         58m
service/postgres     ClusterIP   10.0.3.200   <none>        5432/TCP         58m
service/prometheus   ClusterIP   10.0.3.203   <none>        9090/TCP         58m
service/zookeeper    ClusterIP   10.0.3.175   <none>        2181/TCP         58m

```


### Grafana Dashboard - проверка работы

```
kubectl get nodes -o wide

NAME                        STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cl1j698riecva6j51n9h-oqip   Ready    <none>   64m   v1.27.3   10.128.0.44   158.160.51.162   Ubuntu 20.04.6 LTS   5.4.0-177-generic   containerd://1.6.28

```


  ![alt text](assets/grafana_k8s.png)



## Приложения - конфигурационные файлы развертывания кластера средствами `docker-compose` и `k8s`
### Приложение 1. `calccloud_homework_02.yaml` - конфигурационный файл развертывания кластера средством `docker-compose`

Ссылка на репозиторий: [https://github.com/dk-miptedu/cloudsrv_claster/blob/dcomposer/homework03.md](https://github.com/dk-miptedu/cloudsrv_claster/blob/dcomposer/homework03.md)

```
#cloud-config
datasource:
  Ec2:
    strict_id: false
ssh_pwauth: no
users:
- name: hw2-user
  sudo: "ALL=(ALL) NOPASSWD:ALL"
  shell: /bin/bash
  ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCnu/vz4gPsfDTAtPZ3CSe1hunRDUMTFcWPKs8+5Oa/M1gAA8VyeFA9Ip8jwmyZN0KM98WYIMvDr6uLZJmNeB+QuzQfuLC6yn1AwNyFXBY8STHnYL2EAIYKgzRVFZwE09KlYW9szKrtTxGCs38ILBnWUysgC5E4kiYrQuTuvF3rnrF2qHMRRc0oSk+1CpSydhnxNJ34zQqJ3vbFPzI5E/eXUf2J1UDxolddRp/9lttgg7AJd/s65/h08cufTHakFuy8ohI4TarIGztsUXTO6MlRKwjnnjfSXQqIWu8/FwD6QlZ8mY/CCQAeHOdnaGsFxzT3H6eVBwNJbvgOEqimof8mJ8xngQpAmx/gHyue+JJXE4YTpxXI7rMCaxVe8ZneqRGLf+2CEvrBaL/sqxk6kbu7v6qDs6RvfZf/aLGfi94wjTl01w8wTQb90dBpXdhrSrvpEx7PigrKEz8jKt5sdZ9gP6upZaVUAZYz9w6RtCYkteAHfQamzcOXaRHoNmAHZPNcTV07tSca/ifSu/z3niYDhJ5M2OlBEagfr1BmT18TcSl8KoACxpvKeq08SEpipRpaMfifSyQlom9xEjQ3Ny+RC6YoZPBXsUNF7NfJaQRmBnNIY8moJcqQQXkDbS+uHeszTshMLYZw+trYso5epRuxu8gtH7mr7nwW2Z7wI+JzwQ== kazanskii.da@phystech.edu
write_files:
  - path: "/usr/local/etc/docker-install.sh"
    permissions: "755"
    content: |
      #!/bin/bash

      # Docker
      echo "Installing Docker"
      sudo apt update -y && sudo apt install -y docker.io docker-compose mc screen net-tools
      echo "Grant user access to Docker"
      sudo usermod -aG docker hw2-user
      newgrp docker
      sudo ufw allow 3000 & ufw allow 9092 & ufw allow 9093 & ufw enable

      mkdir -p /usr/local/etc/data-cluster
      mkdir -p /usr/local/etc/monitoring-cluster

    defer: true
  - path: "/usr/local/etc/data-cluster/docker-compose.yml"
    permissions: "755"
    content: |
      version: '3.8'

      services:
        # PostgreSQL Service
        postgres:
          image: postgres:15
          container_name: postgres
          environment:
            POSTGRES_USER: admin
            POSTGRES_PASSWORD: admin
            POSTGRES_DB: example_db
          ports:
            - "5432:5432"
          volumes:
            - postgres_data:/var/lib/postgresql/data
            - ./init.sql:/docker-entrypoint-initdb.d/init.sql
          networks:
            - monitoring_net

        postgres-exporter:
          image: prometheuscommunity/postgres-exporter:latest
          container_name: postgres_exporter
          environment:
            DATA_SOURCE_NAME: "postgresql://admin:admin@postgres:5432/example_db?sslmode=disable"
          ports:
            - "9187:9187"
          networks:
            - monitoring_net

        # Kafka Service
        zookeeper:
          image: confluentinc/cp-zookeeper:7.5.0
          container_name: zookeeper
          environment:
            ZOOKEEPER_CLIENT_PORT: 2181
            ZOOKEEPER_TICK_TIME: 2000
          ports:
            - "2181:2181"
          networks:
            - monitoring_net

        kafka:
          image: confluentinc/cp-kafka:7.5.0
          container_name: kafka
          environment:
            KAFKA_BROKER_ID: 1
            KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
            KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
            KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
          ports:
            - "9092:9092"
          depends_on:
            - zookeeper
          networks:
            - monitoring_net
         

        # Prometheus Service
        prometheus:
          image: prom/prometheus:latest
          container_name: prometheus
          volumes:
            - ./prometheus.yml:/etc/prometheus/prometheus.yml
          ports:
            - "9090:9090"
          networks:
            - monitoring_net

        # Grafana Service
        grafana:
          image: grafana/grafana:latest
          container_name: grafana
          environment:
            - GF_SECURITY_ADMIN_USER=admin
            - GF_SECURITY_ADMIN_PASSWORD=admin
          ports:
            - "0.0.0.0:3000:3000"
          volumes:
            - grafana_data:/var/lib/grafana
          networks:
            - monitoring_net

      volumes:
        postgres_data:
        grafana_data:

      networks:
        monitoring_net:

    defer: true
  
  - path: "/usr/local/etc/data-cluster/prometheus.yml"
    permissions: "755"
    content: |
      global:
        scrape_interval: 15s

      scrape_configs:
        - job_name: 'postgres-exporter'
          static_configs:
            - targets: ['postgres_exporter:9187']

        - job_name: 'prometheus'
          static_configs:
            - targets: ['localhost:9090']

    defer: true    
  - path: "/usr/local/etc/data-cluster/init.sql"
    permissions: "755"
    content: |
      CREATE TABLE IF NOT EXISTS users (
          id SERIAL PRIMARY KEY,
          username VARCHAR(50) NOT NULL,
          email VARCHAR(100) NOT NULL UNIQUE,
          created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      INSERT INTO users (username, email)
      SELECT 
          'user_' || generate_series(1, 100) AS username,
          'user_' || generate_series(1, 100) || '@example.com' AS email;

    defer: true    
runcmd:
  - [su, hw2-user, -c, "/usr/local/etc/docker-install.sh"]
  - [su, hw2-user, -c, "docker-compose -f /usr/local/etc/data-cluster/docker-compose.yml up -d"]

```

### Приложение 2. `k8s-cluster.yaml` - конфигурационный файл развертывания кластера средствами `k8s`

```
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
# ConfigMap для SQL-инициализации PostgreSQL
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init-sql
  namespace: monitoring
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        username VARCHAR(50) NOT NULL,
        email VARCHAR(100) NOT NULL UNIQUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    INSERT INTO users (username, email)
    SELECT 
        'user_' || generate_series(1, 100) AS username,
        'user_' || generate_series(1, 100) || '@example.com' AS email;
---
# StatefulSet для PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: monitoring
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          value: admin
        - name: POSTGRES_DB
          value: example_db
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        - name: init-sql
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: init-sql
        configMap:
          name: postgres-init-sql
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# Service для PostgreSQL
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: monitoring
spec:
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: postgres
---
# Deployment для Zookeeper
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zookeeper
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - name: zookeeper
        image: confluentinc/cp-zookeeper:7.5.0
        ports:
        - containerPort: 2181
        env:
        - name: ZOOKEEPER_CLIENT_PORT
          value: "2181"
        - name: ZOOKEEPER_TICK_TIME
          value: "2000"
---
# Service для Zookeeper
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: monitoring
spec:
  ports:
    - port: 2181
      targetPort: 2181
  selector:
    app: zookeeper
---
# Deployment для Kafka
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        ports:
        - containerPort: 9092
        env:
        - name: KAFKA_BROKER_ID
          value: "1"
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: zookeeper:2181
        - name: KAFKA_ADVERTISED_LISTENERS
          value: PLAINTEXT://kafka:9092
        - name: KAFKA_LISTENERS
          value: PLAINTEXT://0.0.0.0:9092
---
# Service для Kafka
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: monitoring
spec:
  ports:
    - port: 9092
      targetPort: 9092
  selector:
    app: kafka
---
# ConfigMap для Prometheus
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'postgres-exporter'
        static_configs:
          - targets: ['postgres:9187']

      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']
---
# Deployment для Prometheus
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config
          mountPath: /etc/prometheus
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config
---
# Service для Prometheus
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
spec:
  ports:
    - port: 9090
      targetPort: 9090
  selector:
    app: prometheus
---
# Deployment для Grafana
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin
---
# Service для Grafана с NodePort
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 32000
  selector:
    app: grafana
```