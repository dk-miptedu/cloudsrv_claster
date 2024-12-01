
# Практика обработки и визуализации данных

**Цель задания:** Развернуть в облаке систему для визуализации и обработки данных.

## Задачи
- Средствами командной строки приготовить виртуальную машину в Яндекс Облаке, подключиться к ней по протоколу ssh.
- На виртуальной машине средствами docker-compose или k8s запустить кластер обработки данных: postgresql, kafka.
- На виртуальной машине средствами docker-compose или k8s запустить кластер визуализации данных: prometheus, grafana.
- Проверить работу дашборда в Grafana.
- 
## Create VPS with users settings

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


## Grafana Dashboard - проверка работы

Скриншот 1. Метрики `prometheus`

![alt text](assets/prometheus_metrics.png)


Скриншот 2. Подключение `postgreSQL`

![alt text](assets/postgres_metrics.png)

## Приложение 1. Конфигурационный файл развертывания кластера

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