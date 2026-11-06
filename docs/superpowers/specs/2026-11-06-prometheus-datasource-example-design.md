# Prometheus Datasource Example for the ubuntu24 Scenario

## Intent

Make the `ubuntu24` Molecule scenario prove that Grafana can actually query a
Prometheus server, and document for role users how to connect Grafana to a
Prometheus they already run.

The role stays Grafana-only: it does not deploy, configure, or manage
Prometheus. Prometheus exists only in the test scenario, and the README shows
the datasource configuration users write in their own inventory.

## Why now

Today the scenario declares a `Molecule Prometheus` datasource pointing at
`http://localhost:9090` (`molecule/ubuntu24/group_vars/all.yml:16-19`), but
nothing listens on that address. Inside the Grafana container `localhost` is
Grafana itself, so the datasource is dead. `verify.yml` only asserts that the
provisioning object exists with `moleculeRevision: 2`; it never proves
connectivity or data flow. A regression in datasource rendering, container
networking, or the URL would pass unnoticed.

## Goals

- A real Prometheus server runs in the scenario and is reachable from the
  Grafana container.
- Verification proves the datasource is healthy and returns query results, not
  merely that a YAML file was rendered.
- The README documents the datasource pattern, including the network
  reachability rules that determine the URL.
- No role change: no new defaults, tasks, templates, or variables.

## Non-goals

- Deploying Prometheus in production, or adding Prometheus to the role's
  Compose project.
- Scraping real targets, Grafana→Prometheus alerting, or dashboards that query
  Prometheus data.
- Testing Prometheus upgrades, persistence, or configuration management.
- Authentication between Grafana and Prometheus.

## Anti-goals

- The role gaining a `prometheus_*` variable, a second Compose service, or an
  `extra_hosts` entry.
- Verification that only re-reads the provisioning file the role just wrote.
- A scenario that breaks the existing container-replacement persistence check
  in `side_effect.yml`.

## Constraints

- `molecule/ubuntu24/side_effect.yml:24-29` runs `docker compose ... state:
  absent` for the Grafana project, which removes the project's default network.
  Prometheus must not depend on that network.
- `side_effect.yml` recreates the Grafana container between converge and
  verify, so Prometheus must outlive the Compose teardown.
- The Grafana container must reach Prometheus over a real address; the URL must
  be reachable from inside the container, not from the VM shell.
- The existing assertion that the side-effect datasource has
  `jsonData.moleculeRevision == 2` must keep working.

## Approaches considered

1. **Independent Prometheus container with a published host port, addressed by
   the VM IP (chosen).** The scenario starts `prom/prometheus` on host port
   9090 outside the Grafana Compose project. The datasource URL is
   `http://<vm default IPv4>:9090`. Nothing is shared with the Compose project,
   so the side-effect teardown cannot disturb it, and the README can document
   the same pattern for a Prometheus on another host or on the same host with a
   published port.
2. **Prometheus attached to `grafana_default`, addressed by container name.**
   Cleaner URL, but `compose down` in the side-effect step removes the network
   while an external container is attached, so the scenario would have to
   diverge from the current persistence test or drop it.
3. **`extra_hosts: host.docker.internal:host-gateway` in the role template.**
   Lets users write `http://host.docker.internal:9090`, but that is a role
   change and only helps for same-host Prometheus. Rejected because the role
   must stay Grafana-only.

Approach 1 trades a slightly less pretty URL for zero coupling to the role and
a documented pattern that matches how users usually run Prometheus.

## Decisions

### Scenario layout

- Add `molecule/ubuntu24/prepare.yml` and point
  `molecule.yml:ansible.playbooks.prepare` at it. It starts with
  `import_playbook: ../default/prepare.yml` so the Docker installation stays in
  one place, then adds a play that installs and starts the Prometheus fixture.
- Fixture location: `/opt/prometheus-test` on the VM, holding a minimal
  `prometheus.yml` with a 5-second scrape interval scraping Prometheus itself
  (`localhost:9090`), so `up` exists within seconds.
- Container: `prom/prometheus:v3.16.0`, `restart: unless-stopped`, published
  `9090:9090`, config bind-mounted read-only. No named volume and no
  `docker_compose_v2`; `community.docker.docker_container` plus a readiness
  wait on `http://127.0.0.1:9090/-/ready`.
- The scenario's destroy step removes the VM, so no explicit Prometheus cleanup
  is added.

### Datasource wiring

- `group_vars/all.yml` gains `molecule_prometheus_url: "http://{{
  ansible_default_ipv4.address }}:9090"` and uses it for the datasource `url`.
- `side_effect.yml` uses the same expression for the revision-2 datasource so
  both converges agree on the address.
- The datasource keeps `uid: molecule-prometheus`, `access: proxy`,
  `isDefault: true`, and `jsonData.moleculeRevision`.

### Pin the Grafana version in the scenario

The new assertions depend on version-sensitive contracts:
`GET /api/datasources/uid/{uid}/health` returning `OK`, `/api/ds/query`
returning `results.A.status == 200`, and the Prometheus datasource plugin's
frame layout. Running `latest` would let an upstream release break the scenario
with no repo change.

- `molecule/ubuntu24/group_vars/all.yml` gains
  `fabos_grafana_version: "13.3.0"`, matching the version in the README example
  and the newest release present on Docker Hub at the time of writing.
- The role default stays `latest`; only the scenario pins.
- Assertions target the pinned version's behaviour and do not need tolerance
  shims.

### Verification

`verify.yml` gains, after the revision-2 assertion:

1. A direct check that Prometheus answers `/api/v1/query?query=up` from the VM,
   giving a clear failure when the fixture itself is broken.
2. `GET /api/datasources/uid/molecule-prometheus/health` and an assertion that
   the reported status is `OK`, proving the container can reach the URL.
3. `POST /api/ds/query` with a `prometheus` query for `up` (instant, range
   `now-5m` to `now`), asserting `results.A.status == 200` and that the frame
   contains sample values, proving data actually flows through Grafana.

The existing `Molecule Prometheus` existence assertion stays.

### Documentation

`README.md` gains a "Prometheus as a datasource" section covering:

- The role provisions datasource objects only; it never runs Prometheus.
- A datasource snippet using `http://prometheus.example.org:9090`.
- A table of URL choices and when each works: service name on a shared Compose
  network, host/gateway address with a published port, or a remote hostname.
- The requirement that the URL resolves inside the Grafana container, with the
  `docker compose exec grafana ...` style check and the health endpoint used by
  the scenario.
- A pointer that the `ubuntu24` scenario is the runnable example.

### Correct the stale Testing section

`README.md:158-175` predates the current scenario layout and is wrong
independently of this work. Fix it in the same change, because the new
"runnable example" pointer would otherwise contradict it two paragraphs later:

- The section calls the scenario `kubevirt` and tells users to run
  `molecule test -s kubevirt`. The only scenario directory is
  `molecule/ubuntu24`, and `scenario.name` is `ubuntu24`
  (`molecule/ubuntu24/molecule.yml:36`), so the documented command fails.
  Replace with `molecule test -s ubuntu24`. Mention that the KubeVirt backend
  is selected by `MOLECULE_DRIVER: kubevirt` in the scenario env, which is
  where the `kubevirt` name in the old text came from.
- The section states the test uses the reserved namespace
  `eclipse-slm-grafana-molecule` and that `GRAFANA_MOLECULE_NAMESPACE` should be
  set for concurrent runs. Neither exists: `GRAFANA_MOLECULE_NAMESPACE` appears
  nowhere in the repo, and `molecule_kubevirt` reads
  `molecule_namespace`/`molecule_kubevirt_default_namespace`, defaulting to
  `molecule`. Document the namespace as the backend default `molecule` and note
  that it can be overridden with `molecule_namespace` for concurrent runs.
- Keep the existing statements that a KubeVirt cluster, working kubeconfig, and
  the Docker role are required, and that the controller needs Python 3, pip,
  and the Kubernetes client.
- Add the scenario's new prerequisite: it runs a `prom/prometheus` container as
  a test fixture, so the VM needs outbound access to Docker Hub on first run.
- Describe the test sequence as it exists: converge, idempotence, side_effect,
  verify, with side_effect testing data persistence across container
  replacement. Use the `ubuntu24` scenario directory when referencing the
  persistence check.

## Left to the builder

- Exact Prometheus image tag pin and config file contents.
- Whether the readiness wait uses `uri` retries or `wait_for`.
- Wording and placement of the README sections.
- Whether to add a local `tests/` play for the datasource rendering; not
  required, since provisioning rendering is already covered.
