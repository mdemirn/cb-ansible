# cb-ansible

Ansible role and playbook that create a Couchbase bucket through the REST API or
`couchbase-cli`, with the settings you pass or sensible defaults.

## What it does

The `couchbase_bucket` role runs four steps:

1. **Health check:** asks every node of the `couchbase_cluster` inventory group for
   `/pools` and picks the first one that answers. Stops if none does.
2. **Validate:** checks the bucket name (1 to 100 characters, letters, digits and
   `._%-`) and whether the bucket already exists.
3. **Create:** creates the bucket if it does not exist yet and waits until it is ready
   (REST API: healthy on every node; CLI: `--wait`). An existing bucket is left alone.
4. **Info:** prints the bucket's type, RAM quota, replicas, eviction policy, storage
   backend, durability, compression, TTL, node health, item count and memory and disk
   use.

Both modes are supported: the REST API (default) and `couchbase-cli bucket-create` on
the target host. Passwords are never printed (`no_log`), and with TLS the role talks to
port 18091 and can use a CA file and a client certificate.

## Requirements

- Ansible (ansible-core 2.14 or later)
- Couchbase Server reachable from the target host
- For CLI mode: `couchbase-cli` on the target host (default `/opt/couchbase/bin`)

## Usage

```bash
git clone https://github.com/mdemirn/cb-ansible.git
cd cb-ansible
```

Put your Couchbase nodes into `inventory/hosts.yml`:

```yaml
---
all:
  children:
    couchbase_cluster:
      hosts:
        node-01:
          ansible_host: node-01.example.internal
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

Set the admin password in `group_vars/vault.yml` and encrypt the file:

```bash
ansible-vault encrypt group_vars/vault.yml
```

Create a bucket:

```bash
ansible-playbook playbooks/create_bucket.yml --ask-vault-pass \
  -e target_host=node-01 -e bucket_name=orders -e bucket_ram_quota=512 -e bucket_replica_count=2
```

The playbook runs on `target_host`. Without it, it runs on the machine's own short
hostname over a local connection, which is handy when you start it on a Couchbase
node. Without `bucket_name`, the bucket is named after the target host.

To avoid the vault prompt, store the vault password in `.vault_pass` (ignored by git) and
uncomment `vault_password_file` in `ansible.cfg`.

## Bucket settings

| Variable | Default | Couchbase setting |
|---|---|---|
| `bucket_name` | target host name | Bucket name |
| `bucket_type` | `couchbase` | `couchbase` or `ephemeral` |
| `bucket_ram_quota` | `256` | RAM quota per node in MB |
| `bucket_replica_count` | `1` | Number of replicas |
| `bucket_eviction_policy` | `valueOnly` | `valueOnly` or `fullEviction` |
| `bucket_storage_backend` | `couchstore` | `couchstore` or `magma` |
| `bucket_conflict_resolution` | `seqno` | `seqno` or `lww` |
| `bucket_durability_min_level` | `none` | Minimum durability level |
| `bucket_compression_mode` | `passive` | `off`, `passive` or `active` |
| `bucket_max_ttl` | `0` | Maximum document TTL in seconds |
| `bucket_flush_enabled` | `0` | Allow flush |
| `bucket_enable_index_replica` | `0` | Replicate view indexes |
| `bucket_priority` | `low` | Bucket priority (CLI mode) |

## Connection settings

| Variable | Default | Meaning |
|---|---|---|
| `use_rest_api` | `true` | `false` uses `couchbase-cli` instead |
| `couchbase_admin_user` | `Administrator` | Cluster admin user |
| `couchbase_admin_password` | from `vault_couchbase_admin_password` | Cluster admin password |
| `use_tls` | `false` | HTTPS on port 18091 and `couchbases://` for the CLI |
| `tls_verify_certs` | `true` | Verify the server certificate |
| `tls_ca_cert_path` | empty | CA file |
| `tls_client_cert_path`, `tls_client_key_path` | empty | Client certificate and key (set both or neither) |
| `tls_client_key_password` | from `vault_tls_client_key_password` | Password of the client key |
| `couchbase_cli_path` | `/opt/couchbase/bin` | Location of `couchbase-cli` |
| `couchbase_api_timeout`, `couchbase_api_retries`, `couchbase_api_retry_delay` | `10`, `3`, `5` | REST call timeout and retries |
| `couchbase_bucket_ready_retries`, `couchbase_bucket_ready_delay` | `30`, `10` | How long to wait for the new bucket to become healthy |

## Tags

`health_check`, `validate`, `create`, `info` and `tls` select the matching steps. To
print an existing bucket's details without creating anything:

```bash
ansible-playbook playbooks/create_bucket.yml --ask-vault-pass \
  -e target_host=node-01 -e bucket_name=orders --tags health_check,validate,info
```

## Layout

```
playbooks/create_bucket.yml        playbook
inventory/hosts.yml                Couchbase nodes (couchbase_cluster group)
group_vars/all.yml                 node list and password variables
group_vars/vault.yml               passwords (encrypt it)
roles/couchbase_bucket/            the role: health_check, validate, create, info
```
