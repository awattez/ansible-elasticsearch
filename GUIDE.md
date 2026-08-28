# Guide - from zero to a running Elasticsearch 9.x

Complete walkthrough: an empty directory on your laptop, one or three Linux
hosts, and at the end a secured Elasticsearch cluster.

The pattern: **one inventory file carries everything**
(hosts + configuration), the playbooks ship with the collection. You never
write a playbook.

## 0. Prerequisites

Control machine (your laptop):

```bash
# ansible-core 2.16+ (pick one)
brew install ansible          # macOS
pipx install ansible-core     # any OS
ansible --version
```

Target hosts:

- Ubuntu 22.04+/Debian 12+ or RHEL/Rocky/Alma 8+, x86_64, systemd
- SSH reachable with a sudo-capable user (`ssh ubuntu@10.0.0.11` works)
- 4 GB RAM minimum per node (2 GB heap + OS)
- Ports open between nodes: `9200` (HTTP) and `9300` (transport)

## 1. Project layout

```bash
mkdir es-deploy && cd es-deploy
```

You will end up with exactly two files:

```
es-deploy/
├── requirements.yml   # where the collection comes from
└── inventory.yml      # your hosts AND all the configuration
```

## 2. Install the collection

`requirements.yml`:

```yaml
collections:
  - name: git+https://github.com/awattez/ansible-elasticsearch.git
    type: git
    version: next
```

```bash
ansible-galaxy collection install -r requirements.yml --force
```

`--force` re-downloads, so re-running the command picks up collection updates.
It lands in `~/.ansible/collections/ansible_collections/awattez/elastic`.

Three playbooks ship with it (component name = deploy/converge it; future
operations will get a verb prefix like `upgrade_elasticsearch`):

- `awattez.elastic.all` - every component group present in the
  inventory (`elasticsearch`, then `kibana`)
- `awattez.elastic.elasticsearch` - Elasticsearch only
- `awattez.elastic.kibana` - Kibana only

No uninstall - tear down by deleting the VMs.

## 3. Scenario A - single node

The lab entry point: security on (mandatory in 9.x), HTTP kept in clear to
avoid certificate handling on day one.

`inventory.yml`:

```yaml
---
elasticsearch:
  vars:
    es_version: "9.5.2"
    es_heap_size: 1g
    es_api_basic_auth_username: elastic
    es_api_basic_auth_password: "{{ lookup('env', 'ELASTIC_PASSWORD') }}"
    es_config:
      cluster.name: lab
  hosts:
    es-single:
      ansible_host: 10.0.0.11        # <- your VM IP
      ansible_user: ubuntu           # <- your SSH user
```

That's all: with one host in the group the collection renders
`discovery.type: single-node`, `network.host: 0.0.0.0` and
`node.name: es-single` by itself.

Run:

```bash
export ELASTIC_PASSWORD='changeme-please'
ansible-playbook -i inventory.yml awattez.elastic.all
```

Verify:

```bash
curl -u "elastic:$ELASTIC_PASSWORD" http://10.0.0.11:9200
curl -u "elastic:$ELASTIC_PASSWORD" http://10.0.0.11:9200/_cluster/health?pretty
```

Expected: `"number" : "9.5.2"` and `"status" : "green"`.

## 4. Scenario B - 3-node cluster, TLS everywhere

Enable TLS and provide no certificate: the collection generates a self-signed
CA and one certificate per node (with `elasticsearch-certutil` on the first
node) and distributes them. Transport is mTLS, HTTP is HTTPS.

`inventory.yml`:

```yaml
---
elasticsearch:
  vars:
    es_version: "9.5.2"
    es_heap_size: 2g
    es_api_basic_auth_username: elastic
    es_api_basic_auth_password: "{{ lookup('env', 'ELASTIC_PASSWORD') }}"
    es_enable_transport_ssl: true
    es_enable_http_ssl: true
    es_config:
      cluster.name: lab-tls
      node.roles: [master, data, ingest]
  hosts:
    es-1: { ansible_host: 10.0.0.11, ansible_user: ubuntu }
    es-2: { ansible_host: 10.0.0.12, ansible_user: ubuntu }
    es-3: { ansible_host: 10.0.0.13, ansible_user: ubuntu }
```

Three hosts in the group → the collection renders `discovery.seed_hosts`
(the `ansible_host` IPs) and `cluster.initial_master_nodes` (the node names)
automatically. The generated certificates include both hostnames and IPs as
SANs.

Run:

```bash
export ELASTIC_PASSWORD='changeme-please'
ansible-playbook -i inventory.yml awattez.elastic.all
```

Verify (the CA lives on the nodes in `/etc/elasticsearch/certs/ca.crt`):

```bash
scp ubuntu@10.0.0.11:/etc/elasticsearch/certs/ca.crt .
curl --cacert ca.crt --resolve es-1:9200:10.0.0.11 \
  -u "elastic:$ELASTIC_PASSWORD" https://es-1:9200/_cluster/health?pretty
```

Expected: `"number_of_nodes" : 3`, `"status" : "green"`.

Prefer your own PKI? Point `es_ssl_key` / `es_ssl_certificate` /
`es_ssl_certificate_authority` to your PEM files (plus `es_ssl_key_passphrase`
if the key is encrypted) - the generation steps aside automatically. See
"Multi-node cluster with your own certificates" in the [README](README.md).

## 4bis. Scenario C - single node with Kibana

Add the same host to a `kibana` group. The deploy playbook sets the
`kibana_system` password through the security API (ES play), then installs
Kibana against it (kibana play):

`inventory.yml`:

```yaml
---
elasticsearch:
  vars:
    es_version: "9.5.2"
    es_heap_size: 1g
    es_api_basic_auth_username: elastic
    es_api_basic_auth_password: "{{ lookup('env', 'ELASTIC_PASSWORD') }}"
    es_config:
      cluster.name: lab
    es_user_passwords:
      kibana_system: "{{ lookup('env', 'KIBANA_SYSTEM_PASSWORD') }}"
  hosts:
    es-single: { ansible_host: 10.0.0.11, ansible_user: ubuntu }

kibana:
  vars:
    kibana_config:
      elasticsearch.username: kibana_system
      elasticsearch.password: "{{ lookup('env', 'KIBANA_SYSTEM_PASSWORD') }}"
  hosts:
    es-single: { ansible_host: 10.0.0.11, ansible_user: ubuntu }
```

```bash
export ELASTIC_PASSWORD='changeme-please'
export KIBANA_SYSTEM_PASSWORD='changeme-too'
ansible-playbook -i inventory.yml awattez.elastic.all
```

Then open `http://10.0.0.11:5601` and log in with `elastic` /
`$ELASTIC_PASSWORD` (`kibana_system` is Kibana's own service user, not a
login).

## 5. Day 2

Everything is driven by the same inventory - change a var, re-run, the
deployment is idempotent:

```bash
# Nothing changed -> zero "changed" tasks
ansible-playbook -i inventory.yml awattez.elastic.all

# Patch upgrade: bump es_version (e.g. 9.5.2 -> 9.5.3), then one node at a time
ansible-playbook -i inventory.yml awattez.elastic.all --limit es-1
# wait for green, then es-2, es-3

# Add a plugin: add to the inventory vars...
#   es_plugins:
#     - plugin: analysis-icu
# ...and re-run
ansible-playbook -i inventory.yml awattez.elastic.all
```

## 6. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Role fails fast: "requires TLS on the transport layer" | Multi-node with security on but no `es_enable_transport_ssl` / `es_certificates`. That is the 9.x contract - enable one of them. |
| `curl` gets `401` | Wrong `ELASTIC_PASSWORD`; the bootstrap password is only seeded on **first** start. Reset with `elasticsearch-reset-password -u elastic` on a node. |
| `curl` gets connection refused | Check `network.host`, host firewall, cloud security group (ports 9200/9300). |
| Nodes do not form a cluster | `discovery.seed_hosts` names must resolve between nodes; check `/etc/hosts`. |
| Service fails bootstrap checks | `journalctl -u elasticsearch` on the node; typical: not enough RAM for `es_heap_size`. |
| Cert generation re-runs | It is `creates:`-guarded on the first node; wipe `/etc/elasticsearch/certs` on **all** nodes to force regeneration. |
| `awattez.elastic.all` not found | The collection is not installed (or an old copy): `ansible-galaxy collection install -r requirements.yml --force`. |
