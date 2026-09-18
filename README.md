# Elasticsearch Ansible Cluster

Ansible playbook for deploying a secure 3-node Elasticsearch 8.x cluster with TLS, authentication, RBAC, ILM and snapshots.

## Architecture

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
              Shared volume /mnt/backups

- Cluster name: `junior-devops-cluster`
- Version: Elasticsearch 8.13.4
- Security: TLS on transport (node-to-node) and HTTP (client-to-node), Basic Auth
- Administration: RBAC, ILM policy, index template, snapshot repository

## Project structure

    .
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    │   └── all/
    │       ├── vars.yml
    │       └── vault.yml
    ├── roles/
    │   ├── common/
    │   ├── elasticsearch/
    │   ├── certificates/
    │   ├── security/
    │   └── es_admin/
    └── README.md

## Roles

| Role | Responsibility |
|------|----------------|
| common | Java 17, `vm.max_map_count`, file limits, swap off |
| elasticsearch | Elastic repo, install ES 8.13.4, config templates, `bootstrap.password` |
| certificates | Generate CA and SAN certificate on first node, distribute to others |
| security | Start cluster, wait for HTTP, verify cluster health |
| es_admin | RBAC, ILM policy, index template, snapshot repository |

## Quick start

### Requirements

- Linux/macOS or WSL2 on Windows
- Ansible >= 2.15
- Docker (for test environment) or 3 VMs with SSH
- Git

### 1. Clone

    git clone git@github.com:karinakliuchuk/elasticsearch-ansible.git
    cd elasticsearch-ansible

### 2. Start test nodes (Docker)

    docker volume create es-backups
    for i in 1 2 3; do
      docker volume create es-data-$i
      docker run -d --name es-node$i --hostname es-node$i \
        --privileged --cgroupns=host \
        --memory=2g --memory-swap=2g \
        -v es-data-$i:/var/lib/elasticsearch \
        -v es-backups:/mnt/backups \
        -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
        jrei/systemd-ubuntu:22.04
    done

    for node in es-node1 es-node2 es-node3; do
      docker exec $node bash -c "apt update && apt install -y python3 python3-apt sudo curl"
    done

### 3. Configure inventory

Check container IPs:

    docker inspect -f '{{.Name}} -> {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' es-node1 es-node2 es-node3

Update `inventory.ini` accordingly.

### 4. Vault password

    echo "your-vault-password" > .vault_pass
    chmod 600 .vault_pass

### 5. Run

    ansible all -m ping
    ansible-playbook site.yml

## Security

- CA and node certificates generated via `elasticsearch-certutil` on the first node
- Certificate SAN includes DNS names (`es-node1/2/3`, `localhost`) and IPs (`172.17.0.2-4`)
- TLS enabled for:
  - `xpack.security.transport.ssl` - node-to-node traffic (port 9300)
  - `xpack.security.http.ssl` - client HTTPS (port 9200)
- Passwords stored in `group_vars/all/vault.yml` (ansible-vault, AES256)
- `elastic` user password set via `bootstrap.password` in ES keystore
- ES auto-configuration `secure_password` entries removed to allow empty keystore password

## Administration

### RBAC

Application role `app_writer` grants write access to `logs-*` indices only. User `app_user` (password in vault) is assigned this role.

    # Test write access
    curl -sk -u app_user:<password> -X POST "https://localhost:9200/logs-test/_doc" \
      -H 'Content-Type: application/json' \
      -d '{"message": "test", "level": "info"}'

    # Delete is forbidden -> 403
    curl -sk -u app_user:<password> -X DELETE "https://localhost:9200/logs-test"

### Index Lifecycle Management

Policy `logs_policy` manages `logs-*` indices:

| Phase | Condition | Action |
|-------|-----------|--------|
| hot | - | rollover at 5GB or 1 day |
| warm | 7 days | reduce replicas to 1, forcemerge |
| delete | 30 days | delete index |

### Index template

Template `logs_template` applies to `logs-*`:

- 1 shard, 1 replica
- Mapped fields: `@timestamp`, `message`, `level`, `service`
- ILM policy attached: `logs_policy`

### Snapshots

Repository `backup_repo` on shared volume `/mnt/backups` mounted into all nodes.

    # Create snapshot
    curl -sk -u elastic:<password> -X PUT \
      "https://localhost:9200/_snapshot/backup_repo/snapshot_$(date +%Y%m%d)?wait_for_completion=true"

    # List snapshots
    curl -sk -u elastic:<password> \
      "https://localhost:9200/_snapshot/backup_repo/_all?pretty"

    # Restore
    curl -sk -u elastic:<password> -X POST \
      "https://localhost:9200/_snapshot/backup_repo/snapshot_20260918/_restore"

## Verification

### Cluster health

    docker exec es-node1 curl -sk -u elastic:<your-password> \
      "https://localhost:9200/_cluster/health?pretty"

Expected:

    {
      "cluster_name" : "junior-devops-cluster",
      "status" : "green",
      "number_of_nodes" : 3,
      "number_of_data_nodes" : 3
    }

### Node list

    docker exec es-node1 curl -sk -u elastic:<your-password> \
      "https://localhost:9200/_cat/nodes?v"

### Security check

    # Without password -> 401 Unauthorized
    docker exec es-node1 curl -sk "https://localhost:9200/_cluster/health"

    # With password -> 200 OK
    docker exec es-node1 curl -sk -u elastic:<your-password> \
      "https://localhost:9200/_cluster/health"

## Idempotency

Re-running the playbook does not change state:

    ansible-playbook site.yml
    # PLAY RECAP: changed=0 on all hosts

## Design decisions

- Roles split by responsibility (common / elasticsearch / certificates / security / es_admin)
- Order in `site.yml` matters: certificates are generated after ES install because `elasticsearch-certutil` is part of the ES package
- `certutil` uses `--dns` and `--ip` flags instead of `instances.yml`, because `--in` with multiple instances produces a ZIP archive instead of a single p12
- `bootstrap.password` is the official ES 8.x mechanism to set the initial elastic user password
- ES auto-configuration creates `secure_password` entries in the keystore that override the empty password in `elasticsearch.yml`. These entries are removed during the ES install role
- Snapshot repository uses Docker shared volume `es-backups` because ES requires all master-eligible nodes to see the same path
- Docker connection plugin used for the test environment (no SSH overhead)

## Known limitations

- Keystore password is empty (`--pass ""`) for simplicity. In production use a real password via `elasticsearch-keystore add`
- Container IPs are dynamic (Docker bridge). When containers are recreated, update `inventory.ini`
- `apt_key` and `apt_repository` are deprecated in ansible-core 2.21 and will be removed in 2.25. Migration to `deb822_repository` is recommended for production
- Snapshot repository is on local filesystem. For production use S3 or NFS
- No monitoring stack (Metricbeat, Prometheus) or Kibana

## Stack

- Ansible 2.21
- Elasticsearch 8.13.4
- OpenJDK 17
- Ubuntu 22.04 (target)
- Docker with systemd-enabled image (test env)


