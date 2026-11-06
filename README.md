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
| `fabos_grafana_dashboard_dir` | `<project_dir>/dashboards/role-managed` | Role-managed dashboard JSON directory. Keep this reserved for files managed by the role. |
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

The same call with a `background` query is what proves data flows, and it is how the test suite verifies the wiring.

For a complete runnable example, see the `ubuntu24` Molecule scenario. It starts a `prom/prometheus` container on port 9090 outside the Grafana Compose project, provisions the datasource against the host address, and asserts the health endpoint plus a real query result.

## Testing

The Molecule `kubevirt` scenario creates and destroys an Ubuntu 24.04 VM through [`molecule_generic`](https://github.com/eclipse-slm/molecule_generic) and its KubeVirt backend. The test VM is prepared with Docker using the Eclipse SLM Docker role. A KubeVirt cluster and working kubeconfig are required. The test uses the reserved namespace `eclipse-slm-grafana-molecule` and a randomly generated, ephemeral VM SSH password; set `GRAFANA_MOLECULE_NAMESPACE` to a unique namespace for concurrent runs.

Install the scenario's role and collection dependencies, then run the full scenario:

```sh
ansible-galaxy install -r requirements.yml
molecule test -s kubevirt
```

For a local check that empty provisioning inputs generate no files:

```sh
MOLECULE_PROJECT_DIRECTORY="$PWD" ansible-playbook -i localhost, tests/empty-provisioning.yml
```

The KubeVirt test controller needs Python 3 and pip; the scenario installs the Kubernetes Python client and its `jsonpatch` dependency before creating the VM.
