# ansible-role-loki

Deploys [Grafana Loki](https://grafana.com/oss/loki/) on Ubuntu hosts using Docker Compose (Compose v2 plugin), driven entirely by an idempotent Ansible role.

## What it does

1. (Optionally) installs Docker Engine + the Compose plugin from Docker's official apt repo.
2. Creates a dedicated non-root system user/group (`loki`, uid/gid `10001`) matching the upstream image's default user.
3. Lays out `/opt/loki/{config,data}` with correct ownership.
4. Renders `config.yaml` (Loki's own config) and `docker-compose.yml` from Jinja2 templates.
5. Brings the stack up with `community.docker.docker_compose_v2`, pulling the pinned image tag.
6. Waits on `/ready` before declaring success.
7. Re-renders + restarts automatically (via handler) whenever the config changes — fully idempotent on reruns.

## Requirements

- Target: Ubuntu 22.04 (jammy) or 24.04 (noble)
- Control node: Ansible >= 2.14
- Collections: `community.docker` (install via `ansible-galaxy collection install -r requirements.yml`)
- If `loki_install_docker: false` (default), Docker Engine + Compose plugin + the Python `docker` SDK must already be present on the target.

## Directory layout

```
.
├── ansible.cfg
├── requirements.yml
├── playbook.yml
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── observability.yml
└── roles/
    └── loki/
        ├── defaults/main.yml
        ├── tasks/
        │   ├── main.yml
        │   └── docker.yml
        ├── templates/
        │   ├── docker-compose.yml.j2
        │   ├── loki-config.yml.j2
        │   └── env.j2
        ├── handlers/main.yml
        └── meta/main.yml
```

## Usage

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory/hosts.ini playbook.yml
```

Verify:

```bash
curl http://<host>:3100/ready
curl http://<host>:3100/metrics | head
```

## Key variables (override in `group_vars`/`host_vars`)

| Variable                | Default        | Notes                                                        |
|--------------------------|----------------|---------------------------------------------------------------|
| `loki_install_docker`    | `false`        | Set `true` to let this role install Docker itself             |
| `loki_version`           | `2.9.8`        | Always pin — never track `latest`                              |
| `loki_base_dir`          | `/opt/loki`    | Root of config + data on the host                              |
| `loki_http_port`         | `3100`         | Loki HTTP API / push endpoint                                  |
| `loki_grpc_port`         | `9096`         | gRPC port                                                       |
| `loki_bind_address`      | `0.0.0.0`      | Set to `127.0.0.1` if you front Loki with nginx/traefik         |
| `loki_retention_period`  | `744h` (31d)   | Requires `loki_retention_enabled: true` (default)               |
| `loki_memory_limit`      | `512m`         | Compose resource limit                                          |
| `loki_extra_env`         | `{}`           | Extra container env vars, e.g. for S3 credentials                |
| `loki_extra_config`      | `{}`           | Raw dict merged into `config.yaml` for advanced setups (S3 backend, ruler/alerting, multi-tenant auth, etc.) |

Full list: `roles/loki/defaults/main.yml`.

## Design notes / gotchas worth knowing for an interview

- **Non-root by default**: the role pins the container to uid/gid `10001` to match the image's built-in `loki` user, and creates a matching host user so bind-mounted directories aren't world-writable root-owned folders.
- **Idempotency**: config and compose files are templated with `notify: Restart loki`, so a no-op rerun changes nothing and a config change triggers exactly one restart via the handler — not a `docker compose up` on every run.
- **`docker_compose_v2` vs `docker_compose`**: this role uses the v2 module (Compose plugin / `docker compose`), since the standalone `docker-compose` binary (v1, Python) is deprecated upstream.
- **Compose `deploy.resources` caveat**: `deploy:` limits are a Swarm-native construct; plain `docker compose up` only partially respects them depending on Compose version. For hard limits in production, consider also setting `mem_limit`/`cpus` directly, or run under Swarm/Kubernetes.
- **Storage**: ships with local filesystem storage (`common.storage.filesystem`) suitable for single-node / small deployments. For HA or larger volumes, swap in S3/GCS via `loki_extra_config` and move to the `simple-scalable` deployment mode (separate read/write targets) instead of single-binary.
- **Retention**: handled by the `compactor` with `retention_enabled: true` — without this, `limits_config.retention_period` is a no-op.
- **Promtail/Grafana**: intentionally out of scope here so the role stays single-purpose (Loki only). Pair it with a `promtail` role (or Alloy) on log-source hosts, and point Grafana's Loki datasource at `http://<this-host>:3100`.

## Next steps to extend this for a portfolio project

- Add a `molecule` test scenario (`molecule init scenario -r loki -d docker`) to demonstrate automated testing.
- Add a companion `promtail` role + a `docker-compose` override for a local Grafana instance to make a full demo stack.
- Wire retention/storage to S3 via `loki_extra_config` and document an IAM policy for it (ties back into AWS SAA knowledge).
