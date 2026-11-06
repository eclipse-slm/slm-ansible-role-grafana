# Grafana Ansible Role Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an Eclipse SLM Ansible role that deploys Grafana with Docker Compose and structured provisioning, with Molecule coverage on Ubuntu 24.04.

**Architecture:** Use a conventional role layout with defaults, validation, templates, and `community.docker.docker_compose_v2`. A KubeVirt Molecule scenario calls `molecule_generic` for VM lifecycle, prepares Ubuntu with the Eclipse SLM Docker role, and verifies Grafana health and rendered provisioning.

**Tech Stack:** Ansible, `community.docker`, Docker Compose v2, Molecule, `molecule_generic`, `molecule_kubevirt`, Ubuntu 24.04.

**Spec:** `docs/superpowers/specs/2026-11-04-grafana-role-design.md`

## Global Constraints

- Production targets already have Docker Engine and the Compose v2 plugin; this role does not install them.
- The deployment uses the official Grafana container and Docker Compose.
- Grafana data persists when the container is replaced.
- Provisioning is driven by structured role variables for datasources, dashboard providers/models, alert rules, contact points, notification policies, templates, and mute timings.
- Admin credentials are supplied through initial-startup environment settings only; changing them after database initialization does not rotate existing credentials.
- Plugins are installed from a caller-supplied plugin list; plugin-specific advanced configuration is out of scope.
- Molecule creates and destroys a KubeVirt VM using `molecule_generic`; the guest OS is Ubuntu 24.04.

## Review Focus

- Empty admin password: role validation must fail before writing or starting the Compose project; cover in Task 2.
- Empty provisioning categories: no invalid or unnecessary provisioning files should be rendered; cover in Task 3.
- Special characters in admin credentials or environment values: values must survive YAML rendering without corrupting Compose configuration; cover in Task 2.
- Reapplying the same deployment: Compose and rendered files must be idempotent and preserve the data volume; cover in Task 2 and the Molecule idempotence step.
- Grafana readiness delay: verification must retry the health endpoint rather than fail on the first startup response; cover in Task 2.

---

## File Structure

- Create `defaults/main.yml` — role defaults, using the `fabos_grafana_` prefix to align with Eclipse SLM role conventions.
- Create `meta/main.yml` — role metadata and supported platform declarations.
- Create `tasks/main.yml` — validation and ordered task includes.
- Create `tasks/validate.yml` — required values and supported configuration validation.
- Create `tasks/configure.yml` — project directories and generated provisioning content.
- Create `tasks/deploy.yml` — Compose project lifecycle.
- Create `templates/docker-compose.yml.j2` — Grafana service and persistent volume definition.
- Create focused provisioning templates under `templates/provisioning/` — datasources, dashboard providers, dashboard JSON, and unified-alerting resources.
- Create `requirements.yml` — `molecule_generic`, `molecule_kubevirt`, Eclipse SLM Docker role, and `community.docker` / `kubernetes.core` collections.
- Create `molecule/kubevirt/molecule.yml` — Ubuntu 24.04 KubeVirt platform and Molecule sequence.
- Create `molecule/kubevirt/create.yml` and `destroy.yml` — `molecule_generic` lifecycle wrappers.
- Create `molecule/kubevirt/prepare.yml` — install Docker and Compose on the test VM using the Eclipse SLM Docker role.
- Create `molecule/kubevirt/converge.yml` and `verify.yml` — apply the role and assert health and provisioning.
- Update `README.md` — requirements, variables, use, security notes, and test commands.

### Task 1: Add role metadata, defaults, and Molecule KubeVirt scaffold

**Files:**
- Create: `meta/main.yml`
- Create: `defaults/main.yml`
- Create: `requirements.yml`
- Create: `molecule/kubevirt/molecule.yml`
- Create: `molecule/kubevirt/create.yml`
- Create: `molecule/kubevirt/destroy.yml`
- Create: `molecule/kubevirt/prepare.yml`
- Create: `molecule/kubevirt/converge.yml`
- Create: `molecule/kubevirt/verify.yml`

**Interfaces:**
- Produces a root role loaded by `MOLECULE_PROJECT_DIRECTORY`, variable prefix `fabos_grafana_`, and a Molecule scenario named `kubevirt`.

- [x] **Step 1: Add the minimal KubeVirt Molecule scenario and lifecycle wrappers**

Use `quay.io/containerdisks/ubuntu:24.04` as the platform image, allocate a 15Gi volume, set `MOLECULE_DRIVER: kubevirt`, and configure the sequence to run dependency, destroy, syntax, create, prepare, converge, idempotence, verify, destroy. The create and destroy wrappers must call `molecule_generic` with `tasks_from: create.yml` and `tasks_from: destroy.yml` respectively. The prepare play must install Docker Engine and the Compose v2 plugin on the guest using the `eclipse-slm.docker` role.

- [x] **Step 2: Add role defaults and metadata**

Define these defaults in `defaults/main.yml`: `fabos_grafana_image: grafana/grafana`, `fabos_grafana_version: latest`, `fabos_grafana_project_name: grafana`, `fabos_grafana_project_dir: /opt/grafana`, `fabos_grafana_http_port: 3000`, `fabos_grafana_admin_user: admin`, empty `fabos_grafana_admin_password`, empty `fabos_grafana_plugins`, empty `fabos_grafana_environment`, `fabos_grafana_datasources: []`, `fabos_grafana_dashboard_providers: []`, `fabos_grafana_dashboards: {}`, and `fabos_grafana_alerting` with empty `alert_rules`, `contact_points`, `notification_policies`, `templates`, and `mute_timings` lists. Declare Ubuntu 24.04 support in role metadata.

- [x] **Step 3: Declare dependencies and run syntax validation**

Declare the `molecule_generic`, `molecule_kubevirt`, and `eclipse-slm.docker` roles plus `community.docker` and `kubernetes.core` collections in `requirements.yml`.

Run: `ansible-galaxy install -r requirements.yml` and `molecule syntax -s kubevirt`
Expected: dependencies resolve and all scenario playbooks pass syntax validation.

- [x] **Step 4: Validate role metadata and YAML**

Run: `ansible-lint meta/main.yml defaults/main.yml requirements.yml molecule/kubevirt`
Expected: no lint errors.

### Task 2: Implement Compose deployment and test lifecycle behavior

**Files:**
- Create: `tasks/main.yml`
- Create: `tasks/validate.yml`
- Create: `tasks/configure.yml`
- Create: `tasks/deploy.yml`
- Create: `templates/docker-compose.yml.j2`
- Create: `tests/validate-admin-password.yml`
- Modify: `molecule/kubevirt/converge.yml`
- Modify: `molecule/kubevirt/verify.yml`
- Modify: `molecule/kubevirt/destroy.yml`

**Interfaces:**
- Consumes: `fabos_grafana_*` defaults from Task 1.
- Produces: a Compose project under `fabos_grafana_project_dir`, using a persistent named volume and the `community.docker.docker_compose_v2` module.

- [x] **Step 1: Write the failing Molecule assertions for required settings and health**

Set `fabos_grafana_admin_password` in the scenario to `Molecule-Test$p@ss:word`, and configure one test environment setting containing YAML-special characters. In `verify.yml`, assert the test host returns HTTP 200 from `http://127.0.0.1:3000/api/health`, retrying until the service is ready. Assert that the Compose project and persistent volume exist.

- [x] **Step 2: Add validation and confirm missing password fails**

Implement `tasks/validate.yml` to fail when `fabos_grafana_admin_password` is empty. Add a focused Molecule assertion/play invocation with the password unset.

Run the local focused test: `ansible-playbook -i localhost, tests/validate-admin-password.yml`
Expected: the playbook succeeds only when role validation rejects an empty password with the required-password message.

- [x] **Step 3: Implement generated Compose config and deployment**

Implement `tasks/configure.yml` to create the project directory and write Compose configuration. Implement `templates/docker-compose.yml.j2` with the configurable Grafana image tag, host-to-container HTTP port mapping, environment, plugin installation list, restart policy, and named data volume mounted at `/var/lib/grafana`. Mount generated provisioning read-only at `/etc/grafana/provisioning` and dashboard JSON read-only at `/etc/grafana/dashboards`. Quote/serialize values as YAML rather than interpolating unescaped strings.

Implement `tasks/deploy.yml` with `community.docker.docker_compose_v2`, `project_src: fabos_grafana_project_dir`, and `state: present`. `tasks/main.yml` must run validation before configuring files and deploying.

- [x] **Step 4: Run Molecule and verify health, persistence, idempotence, and quoting**

Run: `molecule test -s kubevirt`
Expected: the VM is created, Grafana becomes healthy, the persistent volume exists, the second converge reports no unintended changes, and Molecule destroys the VM. Assert the special-character environment setting survives Compose rendering and that the configured password authenticates to Grafana's API.

### Task 3: Add structured Grafana provisioning

**Files:**
- Modify: `defaults/main.yml`
- Modify: `tasks/validate.yml`
- Modify: `tasks/configure.yml`
- Create: `templates/provisioning/datasources.yml.j2`
- Create: `templates/provisioning/dashboard-providers.yml.j2`
- Create: `templates/provisioning/dashboard.json.j2`
- Create: focused unified-alerting templates under `templates/provisioning/`
- Create: `tests/empty-provisioning.yml`
- Modify: `molecule/kubevirt/converge.yml`
- Modify: `molecule/kubevirt/verify.yml`

**Interfaces:**
- Consumes: Compose deployment from Task 2.
- Produces: `fabos_grafana_datasources` (list), `fabos_grafana_dashboard_providers` (list), `fabos_grafana_dashboards` (filename-to-dashboard-model mapping), and `fabos_grafana_alerting` with `alert_rules`, `contact_points`, `notification_policies`, `templates`, and `mute_timings` lists.

- [x] **Step 1: Add test provisioning inputs and failing verification assertions**

In the scenario, configure one datasource, one dashboard provider, one dashboard model with title `Molecule test dashboard`, and at least one entry under each alerting category. In `verify.yml`, assert all expected non-empty categories produce files and the dashboard is registered. Add `tests/empty-provisioning.yml` to run the role's configuration tasks in a temporary project directory with default-empty provisioning inputs and assert that it produces no provisioning YAML or dashboard JSON.

- [x] **Step 2: Implement structured provisioning rendering**

Render Grafana's expected provisioning YAML/JSON layouts from the variables in this task's interface. Write dashboard models as JSON, keep all generated provisioning read-only mounted into the Grafana service, and omit files for empty categories. Keep each template focused on one Grafana provisioning file type.

- [x] **Step 3: Verify Grafana receives the provisioning and empty input remains valid**

Extend verification to query `/api/datasources` with the configured test admin credentials and assert the datasource name is present. Assert the dashboard JSON title and the alerting provisioning files contain the configured test values.

Run: `MOLECULE_PROJECT_DIRECTORY="$PWD" ansible-playbook -i localhost, tests/empty-provisioning.yml` and `molecule test -s kubevirt`
Expected: health and provisioning assertions pass; re-running converge is idempotent.

### Task 4: Document role usage and validate the repository

**Files:**
- Modify: `README.md`

**Interfaces:**
- Documents the variable names and structures implemented in Tasks 1–3, the Docker/Compose prerequisite, first-startup-only admin credential behavior, and the KubeVirt Molecule setup.

- [x] **Step 1: Replace the placeholder README with user documentation**

Document requirements, defaults and required password, persistent storage, environment/plugin configuration, datasource/dashboard/alerting examples, a role invocation playbook, secret handling guidance, test dependencies, `molecule test -s kubevirt`, and the focused empty-provisioning test.

- [ ] **Step 2: Run final static and integration verification**

Run: `ansible-lint .`, `molecule syntax -s kubevirt`, and `molecule test -s kubevirt`.
Expected: lint and syntax pass; the full scenario passes when KubeVirt credentials and cluster access are available. If no KubeVirt cluster is available in the execution environment, report the full test as blocked and retain static checks as verified results.

- [ ] **Step 3: Review changes against the design**

Check that production tasks never install Docker, persistent data survives container replacement, all chosen provisioning categories map to Grafana-supported formats, and Molecule creates/destroys its VM through `molecule_generic`.
