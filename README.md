# Eclipse SLM Grafana Ansible Role

Deploys and configures the official Grafana container on an existing Docker host using Docker Compose. Grafana's database is stored in a persistent Docker volume, so it survives container replacement and upgrades.

The role manages the Compose project, Grafana environment settings, plugin installation, and Grafana provisioning files. It does **not** install Docker or Docker Compose on production hosts.

## Requirements

- Ansible 2.17 or newer.
- Docker Engine and Docker Compose v2 plugin 2.18.0 or newer installed on the target host.
- The `community.docker` collection 5.4.0 or newer installed on the Ansible controller:

  ```sh
  ansible-galaxy collection install community.docker
  ```

The role is tested on Ubuntu 24.04. It runs with privilege escalation because it manages files under `/opt/grafana` by default. Docker bind-mount paths must not contain a colon; the role validates this before writing files or starting Compose.

## Role behavior

- Creates a Compose project at `/opt/grafana` and starts `grafana/grafana:latest` by default.
- Publishes host port `3000` to Grafana's container port `3000`.
- Stores Grafana data in a named Docker volume (`grafana-data` by default).
- Mounts generated provisioning, dashboard models, and an admin password file read-only into Grafana. Dashboards are isolated in `/opt/grafana/dashboards/role-managed`; other files in `/opt/grafana` are left untouched.
- Requires a non-empty initial admin password. The username and password file are used by Grafana at first startup. **Changing the password variable later does not rotate a password in an already-initialized database**; change it through Grafana itself.
- Writes the initial password to `/opt/grafana/secrets/admin_password`, owned by the Grafana container UID (472 by default) with mode `0400`. Compose passes Grafana's `$__file{...}` setting instead of placing the password value in Compose or the container environment.
- Installs plugins from the configured plugin list.
- Restarts Grafana when a role-managed provisioning file changes, but does not restart an unchanged deployment.
- Removes only role-reserved provisioning files and dashboard JSON files under the role-managed dashboard directory. Empty inputs do not automatically remove resources already stored in Grafana; use explicit delete/prune inputs below when deprovisioning.

Pin `fabos_grafana_version` to a specific Grafana release in production rather than relying on the `latest` default.
The default image pull policy is `missing`; set `fabos_grafana_pull_policy: always` to refresh a mutable tag such as `latest` on each role run.

## Variables

All role variables use the `fabos_grafana_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `fabos_grafana_image` | `grafana/grafana` | Grafana container image. |
| `fabos_grafana_version` | `latest` | Image tag; pin this in production. |
| `fabos_grafana_project_name` | `grafana` | Docker Compose project name and prefix for the persistent volume. |
| `fabos_grafana_project_dir` | `/opt/grafana` | Host directory containing Compose and provisioning files. |
| `fabos_grafana_compose_file` | `docker-compose.yml` | Compose file name written inside `fabos_grafana_project_dir` and used to deploy the project. Must be a plain `.yml`/`.yaml` filename. |
| `fabos_grafana_config_dir` | `<project_dir>` | Host base directory for the `provisioning`, `dashboards` and `secrets` bind-mount sources. |
| `fabos_grafana_dashboard_dir` | `<config_dir>/dashboards/role-managed` | Role-managed dashboard JSON directory. Keep this reserved for files managed by the role. |
| `fabos_grafana_data_volume_name` | `<project_name>-data` | Persistent Docker volume name. Set explicitly to retain the same data volume if changing the Compose project name. |
| `fabos_grafana_http_port` | `3000` | Host port mapped to Grafana's port 3000. Must be an integer from 1 to 65535. |
| `fabos_grafana_restart_policy` | `unless-stopped` | Docker restart policy (`no`, `always`, `on-failure`, or `unless-stopped`). |
| `fabos_grafana_pull_policy` | `missing` | Compose image pull policy (`always`, `missing`, `never`, or `policy`). |
| `fabos_grafana_container_uid` / `_gid` | `472` / `472` | Owner of the host-side initial-password file, matching the official Grafana container user. |
| `fabos_grafana_admin_user` | `admin` | Initial Grafana administrator username. |
| `fabos_grafana_admin_password` | `""` (required) | Initial administrator password. The role fails if empty. Store the value in Ansible Vault. |
| `fabos_grafana_plugins` | `[]` | List of plugin IDs passed to Grafana's plugin installer. |
| `fabos_grafana_environment` | `{}` | Additional Grafana container environment variables, for example `GF_SERVER_ROOT_URL`. Dollar signs are escaped for both Compose and Grafana's configuration expansion so they remain literal. |
| `fabos_grafana_datasources` | `[]` | Grafana datasource objects, written under `provisioning/datasources`. |
| `fabos_grafana_delete_datasources` | `[]` | Datasource deletion objects written to Grafana's `deleteDatasources` provisioning key. |
| `fabos_grafana_datasource_prune` | `false` | Emit `prune: true`; Grafana removes role-provisioned datasources omitted from the file. Use with care if multiple provisioning sources exist. |
| `fabos_grafana_dashboard_providers` | `[]` | Grafana dashboard provider objects, written under `provisioning/dashboards`. |
| `fabos_grafana_dashboards` | `{}` | Mapping of simple `.json` filenames to Grafana dashboard model objects. |
| `fabos_grafana_alerting` | See below | Lists of alert rules, contact points, notification policies, templates, and mute timings. |
| `fabos_grafana_loki_enabled` | `false` | Deploy Loki in the Grafana Compose project. See [Loki and Alloy](#loki-and-alloy). |
| `fabos_grafana_alloy_enabled` | `false` | Deploy Grafana Alloy in the Grafana Compose project to collect host logs. |
| `fabos_grafana_loki_*`, `fabos_grafana_alloy_*` | See below | Image, version, port, volume, and configuration settings for Loki and Alloy. |

`fabos_grafana_alerting` defaults to:

```yaml
fabos_grafana_alerting:
  alert_rules: []
  delete_rules: []
  contact_points: []
  delete_contact_points: []
  notification_policies: []
  reset_policies: []
  templates: []
  delete_templates: []
  mute_timings: []
  delete_mute_timings: []
```

Each provisioning variable accepts Grafana's native object structure. The role adds the required provisioning `apiVersion` and root keys, writes each alerting category to its own file, and mounts the files read-only. Grafana may interpolate `$ENV_VAR` references in provisioning values; use `$$` for a literal dollar sign as described in the [Grafana provisioning documentation](https://grafana.com/docs/grafana/latest/administration/provisioning/).

When removing configured Grafana resources, specify the corresponding delete inputs (for example `delete_rules`, `delete_contact_points`, or `delete_mute_timings`) rather than relying on an empty list to delete database resources. To provision a native Grafana notification template containing Go `{{ ... }}` expressions, mark the Ansible value `!unsafe` so Ansible does not interpret it as Jinja:

```yaml
fabos_grafana_alerting:
  templates:
    - orgId: 1
      name: notifications
      template: !unsafe |
        {{ define "notifications.message" }}{{ .Status }}{{ end }}
```

`notification_policies` represents the complete notification policy tree; Grafana replaces the existing tree when that file is provisioned. For detailed object fields, see Grafana's [alerting file provisioning documentation](https://grafana.com/docs/grafana/latest/alerting/set-up/provision-alerting-resources/file-provisioning/).

## Example

```yaml
---
- name: Deploy Grafana
  hosts: grafana_hosts
  become: true
  vars:
    fabos_grafana_version: "13.3.0"
    fabos_grafana_http_port: 3000
    fabos_grafana_admin_password: "{{ vault_grafana_admin_password }}"
    fabos_grafana_plugins:
      - grafana-clock-panel
    fabos_grafana_environment:
      GF_SERVER_ROOT_URL: https://grafana.example.org/
      GF_USERS_DEFAULT_THEME: light
    fabos_grafana_datasources:
      - name: Prometheus
        uid: prometheus
        type: prometheus
        access: proxy
        url: http://prometheus.example.org:9090
        isDefault: true
        editable: false
    fabos_grafana_dashboard_providers:
      - name: provisioned
        orgId: 1
        folder: Provisioned
        type: file
        disableDeletion: false
        updateIntervalSeconds: 30
        allowUiUpdates: false
        options:
          path: /etc/grafana/dashboards
    fabos_grafana_dashboards:
      overview.json:
        id: null
        uid: overview
        title: Service overview
        tags: [provisioned]
        timezone: browser
        schemaVersion: 39
        version: 0
        panels: []
    fabos_grafana_alerting:
      alert_rules: []
      contact_points:
        - orgId: 1
          name: Operations email
          receivers:
            - uid: operations-email
              type: email
              settings:
                addresses: ops@example.org
      notification_policies:
        - orgId: 1
          receiver: Operations email
          group_by: ['...']
      templates: []
      mute_timings: []
  roles:
    - eclipse_slm.grafana
```

Keep `vault_grafana_admin_password` in an encrypted Ansible Vault file; do not commit real credentials in playbooks or inventory.

## Prometheus as a datasource

The role provisions Grafana datasource objects only. It does not deploy, configure, or health-check Prometheus: run Prometheus yourself, then point a datasource at it. The role never places Prometheus in its Compose project and adds no Prometheus variables.

```yaml
fabos_grafana_datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus.example.org:9090
    isDefault: true
    editable: false
```

### Choosing the URL

Grafana resolves the URL from inside its own container, so the address must be reachable from that container, not just from the Docker host.

| Prometheus location | URL to use | Notes |
| --- | --- | --- |
| Separate host or service | `http://prometheus.example.org:9090` | Simplest. Requires network reachability from the Grafana container. |
| Same host, published port | `http://<host-address>:9090` | Publish Prometheus with a host port and use the host's real address. Do not use `localhost`. |
| Same Compose network | `http://<service-name>:9090` | Only when both containers share a Compose network. The role does not add Prometheus to its project. |

`localhost` and `127.0.0.1` always mean the Grafana container, so they never reach a Prometheus running on the host. To test a candidate URL from inside the running container:

```sh
docker compose -p grafana exec -T grafana \
  wget -qO- http://<prometheus-url>:9090/-/ready
```

A URL that returns `200` there will work as a datasource.

### Confirming the datasource works

Grafana's datasource health endpoint reports whether the server can actually reach Prometheus:

```sh
curl -u admin:"$GRAFANA_ADMIN_PASSWORD" \
  http://127.0.0.1:3000/api/datasources/uid/prometheus/health
```

A working datasource returns `"status":"OK"`, for example:

```json
{"details":{"application":"Prometheus"},"message":"Successfully queried the Prometheus API.","status":"OK"}
```

Grafana reports an unreachable or misconfigured Prometheus from this endpoint with a non-200 status, so a failing check surfaces as an HTTP error rather than a JSON body.

For a complete runnable example, see the `ubuntu24` Molecule scenario. It starts a `prom/prometheus` container on port 9090 outside the Grafana Compose project, provisions the datasource against the host address, and asserts the health endpoint plus a real query result: it POSTs an instant `up` query to Grafana's `/api/ds/query` endpoint and checks that samples come back.

## Loki and Alloy

The role can optionally deploy [Loki](https://grafana.com/docs/loki/latest/) and [Grafana Alloy](https://grafana.com/docs/alloy/latest/) as additional services in the same Compose project as Grafana. Both are disabled by default.

```yaml
fabos_grafana_loki_enabled: true
fabos_grafana_loki_version: "3.6.0"
fabos_grafana_alloy_enabled: true
fabos_grafana_alloy_version: "v1.22.0"
```

When either service is enabled, the role adds a Loki datasource to the Grafana datasource provisioning file (alongside `fabos_grafana_datasources`):

- With `fabos_grafana_loki_enabled: true`, the datasource points at `http://loki:3100` over the Compose network, and Alloy pushes there.
- With only `fabos_grafana_alloy_enabled: true`, set `fabos_grafana_loki_url` to the base URL of an existing Loki (for example `http://loki.example.org:3100`). Alloy pushes to `<url>/loki/api/v1/push` and the datasource points at `<url>`. The role fails validation if this URL is missing.

The automatic datasource is skipped if `fabos_grafana_loki_datasource_enabled` is `false`, or if `fabos_grafana_datasources` already contains an entry with the same `uid` or `name`, so you can supply your own Loki datasource definition.

Loki runs as a single-binary instance with filesystem storage (TSDB, schema v13) in a persistent named volume. The role creates that volume before starting Compose and makes it writable by the Loki container user, because the distroless Loki image does not provide a pre-owned `/loki` directory. Alloy keeps its read positions and state in a named volume and mounts each host directory listed in `fabos_grafana_alloy_host_log_dirs` read-only at the same path. Configuration files are written to `<config_dir>/loki/loki-config.yml` and `<config_dir>/alloy/config.alloy`; a change restarts only the affected service.

### Alloy configuration

By default the role generates an Alloy pipeline (`local.file_match` → `loki.source.file` → `loki.write "default"`) from `fabos_grafana_alloy_file_targets`. Each target is a mapping: `__path__` is a file glob and every other key becomes a Loki label.

```yaml
fabos_grafana_alloy_host_log_dirs:
  - /var/log
  - /srv/app/logs
fabos_grafana_alloy_file_targets:
  - __path__: /var/log/*log
    job: varlogs
    host: "{{ inventory_hostname }}"
  - __path__: /srv/app/logs/*.log
    job: app
```

To add your own components, put [Alloy configuration syntax](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/) in `fabos_grafana_alloy_extra_config`; it is appended to the generated pipeline and can forward to `loki.write.default.receiver`. To take full control, set `fabos_grafana_alloy_config`; it replaces the generated file entirely (and `fabos_grafana_alloy_extra_config` is ignored). Mark values containing `{{ ... }}` as `!unsafe` so Ansible does not template them.

| Variable | Default | Description |
| --- | --- | --- |
| `fabos_grafana_loki_enabled` | `false` | Deploy Loki. |
| `fabos_grafana_loki_image` / `_version` | `grafana/loki` / `latest` | Loki image and tag; pin the tag in production. |
| `fabos_grafana_loki_publish_port` | `true` | Publish Loki's HTTP port on the host, for example so Alloy on other hosts can push logs. Grafana and the bundled Alloy use the Compose network and do not need it. |
| `fabos_grafana_loki_http_port` | `3100` | Host port for Loki's HTTP API when published. |
| `fabos_grafana_loki_data_volume_name` | `<project_name>-loki-data` | Persistent Loki data volume. |
| `fabos_grafana_loki_container_uid` / `_gid` | `10001` / `10001` | Owner the role sets on the Loki data volume, matching the official Loki container user. |
| `fabos_grafana_loki_retention_period` | `744h` | Log retention enforced by the Loki compactor. |
| `fabos_grafana_loki_config_overrides` | `{}` | Mapping recursively merged into the generated Loki configuration. |
| `fabos_grafana_loki_url` | `""` | Base URL of an external Loki. Required when only Alloy is enabled; ignored when Loki is deployed by the role. |
| `fabos_grafana_alloy_enabled` | `false` | Deploy Alloy. |
| `fabos_grafana_alloy_image` / `_version` | `grafana/alloy` / `latest` | Alloy image and tag; pin the tag in production. |
| `fabos_grafana_alloy_data_volume_name` | `<project_name>-alloy-data` | Volume for Alloy's storage path (read positions, state). |
| `fabos_grafana_alloy_host_log_dirs` | `['/var/log']` | Host directories mounted read-only into Alloy at the same path. |
| `fabos_grafana_alloy_file_targets` | `/var/log/*log`, job `varlogs` | File targets for the generated pipeline. Each needs `__path__`. |
| `fabos_grafana_alloy_extra_config` | `""` | Raw Alloy configuration appended to the generated pipeline. |
| `fabos_grafana_alloy_config` | `""` | Raw Alloy configuration that replaces the generated file when non-empty. |
| `fabos_grafana_loki_datasource_enabled` | `true` | Provision the Loki datasource automatically when Loki or Alloy is enabled. |
| `fabos_grafana_loki_datasource_name` / `_uid` | `Loki` / `loki` | Name and UID of the automatic Loki datasource. |
| `fabos_grafana_alloy_docker_logs_enabled` | `false` | Collect the logs of all Docker containers on the host. See [Docker container logs](#docker-container-logs). |
| `fabos_grafana_alloy_docker_socket` | `/var/run/docker.sock` | Host path of the Docker socket mounted read-only into Alloy. |
| `fabos_grafana_alloy_docker_logs_host` | `{{ inventory_hostname }}` | Value of the `host` label on container log streams. |
| `fabos_grafana_docker_logs_dashboard_enabled` | `<alloy_enabled and alloy_docker_logs_enabled>` | Provision the built-in "Docker container logs" dashboard. |
| `fabos_grafana_builtin_dashboard_folder` | `Docker` | Grafana folder for the role's built-in dashboards. |

Disabling Loki or Alloy later removes its container (the role deploys with `remove_orphans`) and its config file, but keeps its data volume. The Loki datasource is removed from the provisioning file, but Grafana keeps it in its database unless you list it in `fabos_grafana_delete_datasources` or enable `fabos_grafana_datasource_prune`.

Run the local rendering test (requires Docker, used to validate the generated Alloy syntax) with:

```sh
MOLECULE_PROJECT_DIRECTORY="$PWD" ansible-playbook -i localhost, tests/loki-alloy.yml
```

### Docker container logs

With `fabos_grafana_alloy_docker_logs_enabled: true`, Alloy discovers every container on the host through the Docker socket (`discovery.docker` → `loki.source.docker`) and ships its stdout/stderr to the same `loki.write "default"` as the file targets. Each stream gets these labels:

| Label | Value |
| --- | --- |
| `container_name` | Container name without the leading `/`, for example `grafana-grafana-1`. |
| `service_name` | Compose service name, or the container name for containers not started by Compose. |
| `compose_project` | Compose project name; absent for containers not started by Compose. |
| `host` | `fabos_grafana_alloy_docker_logs_host` (default: `inventory_hostname`). |

**Security:** the Docker socket is mounted read-only, but read-only only protects the socket file. Any process that can talk to the Docker API can start privileged containers, so Alloy effectively has root access to the host. Enable this only where that is acceptable.

The socket is mounted even when `fabos_grafana_alloy_config` replaces the generated configuration, so a custom pipeline can use `unix:///var/run/docker.sock`. The role adds no Docker components to a replaced configuration.

The role also provisions a **Docker container logs** dashboard (uid `docker-container-logs`) in the `fabos_grafana_builtin_dashboard_folder` folder, bound to the Loki datasource UID `fabos_grafana_loki_datasource_uid`. It has filters for host, Compose project, container, and a case-insensitive regex search. Its panels show total, error/warning, and per-container log volume, plus a logs panel. Built-in dashboards are kept apart from `fabos_grafana_dashboards`: they live in `<config_dir>/builtin-dashboards`, mounted at `/etc/grafana/builtin-dashboards`, with their own provider file `provisioning/dashboards/eclipse-slm-grafana-builtin-dashboard-providers.yml`. Set `fabos_grafana_docker_logs_dashboard_enabled: true` to get the dashboard when the logs are collected by another Alloy with the same labels, or `false` to skip it. Disabling it removes both files.

```sh
MOLECULE_PROJECT_DIRECTORY="$PWD" ansible-playbook -i localhost, tests/docker-logs.yml
```

## Testing

The Molecule `ubuntu24` scenario creates and destroys an Ubuntu 24.04 VM through [`molecule_generic`](https://github.com/eclipse-slm/molecule_generic) and its KubeVirt backend. The KubeVirt driver is selected by `MOLECULE_DRIVER: kubevirt` in the scenario's `ansible.env`. The test VM is prepared with Docker using the Eclipse SLM Docker role. A KubeVirt cluster and working kubeconfig are required.

The KubeVirt backend creates the VM in the `molecule` namespace by default (`molecule_kubevirt_default_namespace`). For concurrent runs, set `molecule_namespace` to a unique namespace per run.

The scenario provisions one of every supported resource category, then runs `side_effect` to remove the Grafana containers while keeping the data volume, so `verify` can prove data survives container replacement. It also starts a `prom/prometheus:v3.16.0` fixture on port 9090 outside the Grafana Compose project, and asserts that the provisioned Prometheus datasource is healthy and returns query results. The VM therefore needs outbound access to Docker Hub on first run.

Install the scenario's role and collection dependencies, then run the full scenario:

```sh
ansible-galaxy install -r requirements.yml
molecule test -s ubuntu24
```

The test sequence is `converge`, `idempotence`, `side_effect`, `verify`, surrounded by `dependency`, `destroy`, `syntax`, `create`, `prepare`, and a final `destroy`. To iterate without destroying the VM between steps, use `molecule create -s ubuntu24`, then `converge`, `side-effect`, and `verify`.

For a local check that empty provisioning inputs generate no files:

```sh
MOLECULE_PROJECT_DIRECTORY="$PWD" ansible-playbook -i localhost, tests/empty-provisioning.yml
```

The KubeVirt test controller needs Python 3 and pip; the scenario installs the Kubernetes Python client and its `jsonpatch` dependency before creating the VM.
