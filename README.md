# Elasticsearch Ansible Cluster

Ansible-автоматизація для розгортання secure 3-node Elasticsearch 8.x кластера з TLS, автентифікацією та автоматичною генерацією сертифікатів.

## Архітектура

    +---------------------------------------------+
    |           Ansible Control Node              |
    |                (WSL Ubuntu)                 |
    +---------------------+-----------------------+
                          |
            +-------------+-------------+
            |             |             |
            v             v             v
    +-----------+ +-----------+ +-----------+
    | es-node1  | | es-node2  | | es-node3  |
    |172.17.0.2 | |172.17.0.3 | |172.17.0.4 |
    |master+data| |master+data| |master+data|
    +-----------+ +-----------+ +-----------+
            |             |             |
            +-------------+-------------+
              TLS transport (9300)
              HTTPS HTTP (9200)

Кластер: junior-devops-cluster
Версія: Elasticsearch 8.13.4
Безпека: TLS на transport (нода-нода) і HTTP (клієнт-нода), Basic Auth

## Структура проєкту

    .
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    │   └── all/
    │       ├── vars.yml          # Загальні змінні
    │       └── vault.yml         # Зашифровані паролі (ansible-vault)
    ├── roles/
    │   ├── common/               # Java 17, sysctl, ліміти, swap off
    │   ├── elasticsearch/        # Встановлення ES, конфіг, JVM heap
    │   ├── certificates/         # CA + node certs з SAN через certutil
    │   └── security/             # Старт кластера, health check
    └── README.md

## Ролі

| Роль | Що робить |
|------|-----------|
| common | Java 17, vm.max_map_count=262144, ліміти nofile/memlock, swap off |
| elasticsearch | Репозиторій Elastic, встановлення ES 8.13.4, шаблони elasticsearch.yml і jvm.options, enable service |
| certificates | Генерація CA і сертифіката з SAN (DNS+IP) на першій ноді, fetch і distribute на решту |
| security | Старт ES, очікування HTTP, перевірка _cluster/health |

## Швидкий старт

### Передумови

- Linux/macOS або WSL2 на Windows
- Ansible >= 2.15
- Docker (для тестового середовища) або 3 VM з SSH
- Git

### 1. Клонування

    git clone git@github.com:karinakliuchuk/elasticsearch-ansible.git
    cd elasticsearch-ansible

### 2. Запуск тестових нод (Docker)

    docker run -d --name es-node1 --hostname es-node1 \
      --privileged --cgroupns=host \
      --memory=2g --memory-swap=2g \
      -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
      jrei/systemd-ubuntu:22.04

    docker run -d --name es-node2 --hostname es-node2 \
      --privileged --cgroupns=host \
      --memory=2g --memory-swap=2g \
      -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
      jrei/systemd-ubuntu:22.04

    docker run -d --name es-node3 --hostname es-node3 \
      --privileged --cgroupns=host \
      --memory=2g --memory-swap=2g \
      -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
      jrei/systemd-ubuntu:22.04

Встановити Python у контейнерах (Ansible вимагає):

    for node in es-node1 es-node2 es-node3; do
      docker exec $node bash -c "apt update && apt install -y python3 python3-apt sudo curl"
    done

### 3. Налаштування inventory

Перевір IP контейнерів:

    docker inspect -f '{{.Name}} -> {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' es-node1 es-node2 es-node3

Онови inventory.ini відповідно.

### 4. Vault password

    echo "your-vault-password" > .vault_pass
    chmod 600 .vault_pass

### 5. Запуск

    ansible all -m ping              # перевірка з'єднання
    ansible-playbook site.yml        # повне розгортання

## Безпека

- CA і сертифікати генеруються через elasticsearch-certutil на першій ноді
- SAN включає DNS (es-node1/2/3, localhost) і IP (172.17.0.2-4)
- TLS увімкнений на:
  - xpack.security.transport.ssl — трафік між нодами (порт 9300)
  - xpack.security.http.ssl — клієнтський HTTPS (порт 9200)
- Паролі зберігаються у group_vars/all/vault.yml (ansible-vault, AES256)
- Пароль elastic встановлюється через bootstrap.password у ES keystore

## Перевірка

### Health кластера

    docker exec es-node1 curl -sk -u elastic:SuperSecretElastic2026 \
      "https://localhost:9200/_cluster/health?pretty"

Очікуваний результат:

    {
      "cluster_name" : "junior-devops-cluster",
      "status" : "green",
      "number_of_nodes" : 3,
      "number_of_data_nodes" : 3
    }

### Список нод

    docker exec es-node1 curl -sk -u elastic:SuperSecretElastic2026 \
      "https://localhost:9200/_cat/nodes?v"

### Перевірка security

    # Без пароля -> 401 Unauthorized
    docker exec es-node1 curl -sk "https://localhost:9200/_cluster/health"

    # З паролем -> 200 OK
    docker exec es-node1 curl -sk -u elastic:SuperSecretElastic2026 \
      "https://localhost:9200/_cluster/health"

## Ідемпотентність

Повторний запуск плейбука не змінює стан:

    ansible-playbook site.yml
    # PLAY RECAP: changed=0 on all hosts

## Технічні рішення

- Розділення ролей — кожна роль відповідає за одну логічну область
- Порядок у site.yml — common -> elasticsearch -> certificates -> security (серти генеруються після встановлення ES, бо потрібен elasticsearch-certutil)
- certutil з SAN — використовуються прапорці --dns і --ip замість instances.yml, бо --in з кількома instance створює ZIP замість p12
- bootstrap.password — офіційний механізм ES 8.x для встановлення пароля elastic при першому старті
- Docker-connection — Ansible підключається до контейнерів через docker exec (без SSH), швидше і простіше для тестового

## Відомі обмеження

- Пароль keystore — використовується порожній (--pass "") для простоти. У проді потрібен реальний пароль через elasticsearch-keystore add
- IP контейнерів — Docker bridge видає IP динамічно, при перестворенні контейнерів треба оновлювати inventory.ini
- apt_key і apt_repository — deprecated в ansible-core 2.21, будуть видалені у 2.25. Для прода треба перейти на deb822_repository

## Стек

- Ansible 2.21
- Elasticsearch 8.13.4
- Java 17 (OpenJDK)
- Ubuntu 22.04 (target)
- Docker з systemd-образом (тестове середовище)


