# Prometheus Datasource Example Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the `ubuntu24` Molecule scenario run a real Prometheus and prove Grafana can query it, and document the Prometheus-as-datasource pattern for role users.

**Architecture:** A `prom/prometheus` container is started by the scenario's `prepare` playbook on host port 9090, outside the Grafana Compose project, so `docker compose down` in the side-effect step cannot disturb it. The Grafana datasource URL becomes `http://<vm ip>:9090`, and `verify.yml` checks Prometheus directly, then Grafana's datasource health endpoint, then an actual `/api/ds/query` result. The role itself is untouched: it stays Grafana-only.

**Tech Stack:** Ansible, Molecule (`molecule_generic`/`molecule_kubevirt`), `community.docker`, Docker Compose v2, Grafana 13.3.0, Prometheus 3.16.0.

**Spec:** `docs/superpowers/specs/2026-11-06-prometheus-datasource-example-design.md`

## Global Constraints

- No role change: no new `defaults/main.yml` variables, no new `tasks/` files, no change to `templates/docker-compose.yml.j2`. The role never deploys, configures, or references Prometheus.
- Prometheus must not join the Grafana Compose project or its `grafana_default` network. `molecule/ubuntu24/side_effect.yml:24-29` removes containers with `remove_volumes: false` and must keep passing.
- The datasource must be reachable from inside the Grafana container; `localhost:9090` is Grafana itself and must not appear anywhere.
- Scenario pins `fabos_grafana_version: "13.3.0"`. The role default stays `latest`.
- Scenario pins the Prometheus image as `prom/prometheus:v3.16.0`; do not use a floating tag.
- Existing assertion `jsonData.moleculeRevision == 2` must keep passing, and the side-effect datasource and the converged datasource must resolve to the same URL.
- Datasource `uid` stays `molecule-prometheus`; `verify.yml` addresses it by that uid.

## Review Focus

- Grafana reaches Prometheus: a URL that only works from the VM shell (`127.0.0.1`, `localhost`) makes the datasource health check fail. The health assertion must fail loudly rather than the datasource silently rendering.
- Compose teardown isolation: attaching Prometheus to `grafana_default` would make the side-effect `compose down` fail or take Prometheus with it, breaking the data-persistence check.
- Fixture readiness: Prometheus needs seconds before `/api/v1/query?query=up` works, and `up` needs one scrape interval to exist. Verify must wait, and the direct Prometheus check must distinguish a broken fixture from a broken datasource.
- Both datasource definitions stay in sync: `group_vars/all.yml` (revision 1) and `side_effect.yml` (revision 2) both carry the URL; updating only one leaves verify querying a stale address.
- Grafana version pin: the health and query contracts belong to 13.3.0. Verify must assert on pinned-version response shapes, not tolerate unknown shapes.

---

### Task 1: Add the failing verification

**Files:**
- Modify: `molecule/ubuntu24/verify.yml` (insert after the revision-2 assertion, currently `verify.yml:197-201`)
- Modify: `molecule/ubuntu24/group_vars/all.yml` (add the URL variable and the Grafana pin this task's checks depend on)

**Interfaces:**
- Consumes: datasource `uid: molecule-prometheus` and `fabos_grafana_admin_password` (`Molecule-Test${MISSING}Tail`) from `group_vars/all.yml`.
- Produces: `molecule_prometheus_url` = `http://<vm ip>:9090` and `fabos_grafana_version: "13.3.0"`, both consumed by Task 3.

- [ ] **Step 1: Add the URL variable and the Grafana pin**

Add after `fabos_grafana_data_volume_name` (`all.yml:6`):

```yaml
# Prometheus runs outside the Grafana Compose project and is addressed by the
# VM address so container-local 127.0.0.1 is never used.
molecule_prometheus_url: "http://{{ ansible_default_ipv4.address }}:9090"
# Pin the Grafana release the scenario's API assertions target.
fabos_grafana_version: "13.3.0"
```

- [ ] **Step 2: Insert the three Prometheus checks**

Insert after the `Assert the test datasource was provisioned` task. Use the same auth style as the neighbouring `uri` tasks (`url_username: admin`, `url_password: "Molecule-Test${MISSING}Tail"`, `force_basic_auth: true`). Retries test only the status code, because a failed connection leaves `.json` undefined and templating it inside `until` would abort instead of retry.

```yaml
    - name: Query the Prometheus fixture directly
      ansible.builtin.uri:
        url: "{{ molecule_prometheus_url }}/api/v1/query?query=up"
        status_code: 200
        return_content: true
      register: grafana_prometheus_up
      retries: 30
      delay: 5
      until: grafana_prometheus_up.status | default(0) == 200

    - name: Assert the Prometheus fixture is scraping
      ansible.builtin.assert:
        that:
          - grafana_prometheus_up.json.data.result | length > 0
          - >-
            grafana_prometheus_up.json.data.result
            | map(attribute='value') | map('last') | select('equalto', '1') | list | length > 0

    - name: Query the datasource health endpoint
      ansible.builtin.uri:
        url: http://127.0.0.1:3000/api/datasources/uid/molecule-prometheus/health
        method: GET
        url_username: admin
        url_password: "Molecule-Test${MISSING}Tail"
        force_basic_auth: true
        status_code: 200
        return_content: true
      register: grafana_datasource_health
      retries: 20
      delay: 5
      until: grafana_datasource_health.status | default(0) == 200

    - name: Assert Grafana can reach Prometheus
      ansible.builtin.assert:
        that:
          - grafana_datasource_health.json.status == 'OK'
        fail_msg: >-
          Grafana cannot reach {{ molecule_prometheus_url }}.

    - name: Query the datasource through Grafana
      ansible.builtin.uri:
        url: http://127.0.0.1:3000/api/ds/query
        method: POST
        url_username: admin
        url_password: "Molecule-Test${MISSING}Tail"
        force_basic_auth: true
        body_format: json
        status_code: 200
        return_content: true
        body:
          from: now-5m
          to: now
          queries:
            - refId: A
              datasource:
                type: prometheus
                uid: molecule-prometheus
              expr: up
              instant: true
              range: false
              format: table
      register: grafana_datasource_query

    - name: Assert the datasource returns sample values
      ansible.builtin.assert:
        that:
          - grafana_datasource_query.json.results.A.status | int == 200
          - grafana_datasource_query.json.results.A.frames | length > 0
          - grafana_datasource_query.json.results.A.frames[0].data.values[1] | length > 0
```

- [ ] **Step 3: Run the scenario and confirm the new checks fail**

The datasource still points at `localhost:9090` and no Prometheus exists yet, so this is the red state that proves the checks are real. The full sequence destroys the VM at the end, so iterate with stages:

```sh
molecule create -s ubuntu24
molecule converge -s ubuntu24
molecule side_effect -s ubuntu24
molecule verify -s ubuntu24
```

Expected: FAIL. `Query the Prometheus fixture directly` exhausts its retries; if it somehow succeeds, `Assert Grafana can reach Prometheus` fails because the datasource points at container-local `localhost`.

- [ ] **Step 4: Tear down and commit**

```sh
molecule destroy -s ubuntu24
git add molecule/ubuntu24/verify.yml molecule/ubuntu24/group_vars/all.yml
git commit -m "test: assert Grafana can query a live Prometheus datasource"
```

---

### Task 2: Run a Prometheus fixture in the scenario

**Files:**
- Create: `molecule/ubuntu24/prepare.yml`
- Modify: `molecule/ubuntu24/molecule.yml:27-33` (add `prepare: prepare.yml` under `playbooks`)

**Interfaces:**
- Consumes: the Docker installation performed by `molecule/default/prepare.yml`.
- Produces: Prometheus listening on VM port 9090, scraped `up` series available within ~10s, and `prometheus-test` fixture directory at `/opt/prometheus-test`.

- [ ] **Step 1: Create `molecule/ubuntu24/prepare.yml`**

```yaml
---
- import_playbook: ../default/prepare.yml

- name: Prepare the Prometheus test fixture
  hosts: all
  become: true
  gather_facts: true
  tasks:
    - name: Create the Prometheus fixture directory
      ansible.builtin.file:
        path: /opt/prometheus-test
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Write the Prometheus fixture configuration
      ansible.builtin.copy:
        dest: /opt/prometheus-test/prometheus.yml
        owner: root
        group: root
        mode: "0644"
        content: |
          global:
            scrape_interval: 5s
          scrape_configs:
            - job_name: prometheus
              static_configs:
                - targets: ["localhost:9090"]

    - name: Start the Prometheus test fixture
      community.docker.docker_container:
        name: molecule-prometheus
        image: prom/prometheus:v3.16.0
        command:
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
        published_ports:
          - "9090:9090"
        volumes:
          - /opt/prometheus-test/prometheus.yml:/etc/prometheus/prometheus.yml:ro
        restart_policy: unless-stopped
        state: started

    - name: Wait for the Prometheus test fixture to become ready
      ansible.builtin.uri:
        url: http://127.0.0.1:9090/-/ready
        status_code: 200
      register: prometheus_ready
      retries: 30
      delay: 2
      until: prometheus_ready.status | default(0) == 200
```

The container has no named volume and no Compose project, so `docker compose ... state: absent` for project `grafana` cannot remove it, and the fixture is on the default bridge rather than `grafana_default`.

- [ ] **Step 2: Wire the playbook into the scenario**

In `molecule/ubuntu24/molecule.yml`, add under `ansible.playbooks` (alongside `create`, `destroy`, `converge`, `side_effect`, `verify`):

```yaml
    prepare: prepare.yml
```

- [ ] **Step 3: Syntax-check the scenario**

```sh
molecule syntax -s ubuntu24
```

Expected: PASS, with `prepare.yml` listed.

- [ ] **Step 4: Verify the fixture starts on a real VM**

```sh
molecule create -s ubuntu24
molecule prepare -s ubuntu24
molecule login -s ubuntu24 -- docker ps --filter name=molecule-prometheus
```

Expected: the container is `Up`, and from inside the VM `curl -s 127.0.0.1:9090/api/v1/query?query=up` returns a result with `"value":[<ts>,"1"]`. Run the curl via `molecule login -s ubuntu24 -- curl ...` if the login command does not take a shell command.

- [ ] **Step 5: Commit**

```sh
git add molecule/ubuntu24/prepare.yml molecule/ubuntu24/molecule.yml
git commit -m "test: run a Prometheus fixture in the ubuntu24 scenario"
```

---

### Task 3: Point the datasource at the fixture

**Files:**
- Modify: `molecule/ubuntu24/group_vars/all.yml:1-21`
- Modify: `molecule/ubuntu24/side_effect.yml:31-42`

**Interfaces:**
- Consumes: Prometheus on VM port 9090 from Task 2.
- Produces: `molecule_prometheus_url` consumed by Task 1's `Query the Prometheus fixture directly` task; both datasource definitions use it.

- [ ] **Step 1: Replace the dead URL in the converged datasource**

In `all.yml:18`, change `url: http://localhost:9090` to:

```yaml
    url: "{{ molecule_prometheus_url }}"
```

- [ ] **Step 2: Replace the dead URL in the side-effect datasource**

In `side_effect.yml:38`, change `url: http://localhost:9090` to the same expression. Both definitions must resolve to the same address; the side-effect revision-2 datasource is what verify queries.

- [ ] **Step 3: Confirm no container-local address remains**

```sh
grep -rn "localhost:9090" molecule/ubuntu24/group_vars molecule/ubuntu24/side_effect.yml
```

Expected: no output.

- [ ] **Step 4: Run the scenario end to end and confirm the checks pass**

```sh
molecule test -s ubuntu24
```

Expected: PASS. All three Prometheus checks from Task 1 pass after the side-effect step, the `moleculeRevision == 2` assertion still passes, and the container-replacement persistence check still passes.

- [ ] **Step 5: Commit**

```sh
git add molecule/ubuntu24/group_vars/all.yml molecule/ubuntu24/side_effect.yml
git commit -m "test: point the scenario datasource at the live Prometheus fixture"
```

---

### Task 4: Document Prometheus as a datasource

**Files:**
- Modify: `README.md` (new section after the existing `## Example` section, before `## Testing`)

**Interfaces:**
- Consumes: `fabos_grafana_datasources` (documented at `README.md:54`).
- Produces: user-facing guidance; `## Testing` is rewritten in Task 5.

- [ ] **Step 1: Add the section**

Cover, in this order:

1. The role provisions datasource objects only; it never deploys, configures, or health-checks Prometheus. Users run Prometheus themselves.
2. A snippet showing the datasource entry:

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

3. A table of URL choices, with when each works:

| Prometheus location | URL to use | Notes |
| --- | --- | --- |
| Separate host or service | `http://prometheus.example.org:9090` | Simplest; requires network reachability from the Grafana container. |
| Same host, published port | `http://<host-address>:9090` | Works with approach used by the test scenario; do not use `localhost`. |
| Same Compose network | `http://<service-name>:9090` | Only when both containers share a Compose network; the role does not add Prometheus to its project. |

4. The rule that `localhost` and `127.0.0.1` refer to the Grafana container, so they never reach a Prometheus on the host. Include the check:

```sh
docker compose -p grafana exec -T grafana \
  wget -qO- http://<prometheus-url>:9090/-/ready
```

5. How to confirm the datasource works through Grafana's API, using the health endpoint the scenario uses:

```sh
curl -u admin:"$GRAFANA_ADMIN_PASSWORD" \
  http://127.0.0.1:3000/api/datasources/uid/prometheus/health
```

Expected body: `{"message":"Data source is working","status":"OK"}`.

6. A closing line that `molecule/ubuntu24` is the runnable example: it starts a `prom/prometheus` fixture on port 9090 and asserts both health and query results.

- [ ] **Step 2: Check the section against the role's variable names**

Confirm every `fabos_grafana_*` name used in the new text exists in `defaults/main.yml`, and that the example does not import `prometheus_*` variables from the role.

```sh
grep -o "fabos_grafana_[a-z_]*" README.md | sort -u
```

- [ ] **Step 3: Commit**

```sh
git add README.md
git commit -m "docs: show how to use Prometheus as a Grafana datasource"
```

---

### Task 5: Correct the stale Testing section

**Files:**
- Modify: `README.md:158-175` (`## Testing` section)

**Interfaces:**
- Consumes: `scenario.name: ubuntu24` (`molecule/ubuntu24/molecule.yml:36`) and `MOLECULE_DRIVER: kubevirt` (`molecule.yml:26`).
- Produces: Testing instructions that match the repository.

- [ ] **Step 1: Fix the scenario name and command**

Replace `The Molecule \`kubevirt\` scenario` with `The Molecule \`ubuntu24\` scenario`, and `molecule test -s kubevirt` with `molecule test -s ubuntu24`. Note that the KubeVirt backend is selected by `MOLECULE_DRIVER: kubevirt` in the scenario's `ansible.env`, which is where the old `kubevirt` wording came from.

- [ ] **Step 2: Fix the namespace claim**

Delete the `GRAFANA_MOLECULE_NAMESPACE` instruction and the claim that the test uses the namespace `eclipse-slm-grafana-molecule`. Replace with: the KubeVirt backend creates the VM in the namespace `molecule` by default (`molecule_kubevirt_default_namespace`), and concurrent runs can set `molecule_namespace` to a unique namespace.

- [ ] **Step 3: Describe the test sequence and new prerequisite**

State the sequence as it exists in `molecule.yml`: converge, idempotence, side_effect, verify. Explain that side_effect removes the Grafana containers while keeping the data volume so verify can prove data survives container replacement. Add that the scenario now starts a `prom/prometheus:v3.16.0` fixture on port 9090, so the VM needs outbound access to Docker Hub on first run.

- [ ] **Step 4: Check the documented names exist**

```sh
grep -rn "molecule test -s" README.md
grep -rn "scenario:" -A2 molecule/ubuntu24/molecule.yml
grep -rn "GRAFANA_MOLECULE_NAMESPACE" README.md molecule/ || true
```

Expected: the documented scenario name matches `molecule.yml`, and the removed variable returns no matches.

- [ ] **Step 5: Commit**

```sh
git add README.md
git commit -m "docs: correct the Molecule testing instructions"
```

---

## Verification

After all tasks, the change is complete when:

```sh
molecule test -s ubuntu24
grep -rn "localhost:9090" molecule/ubuntu24/group_vars molecule/ubuntu24/side_effect.yml  # no output
git diff --stat   # no files under defaults/, tasks/, or templates/ are modified
```

The last check is the guard rail for the spec's core constraint: the role stays Grafana-only.
