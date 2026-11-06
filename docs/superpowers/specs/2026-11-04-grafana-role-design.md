# Grafana Ansible Role Design

## Intent

Build the `eclipse-slm/slm-ansible-role-grafana` repository into a reusable Ansible role for deploying Grafana on an existing Docker host. The role should follow conventions from other Eclipse SLM Ansible roles and be tested with Molecule.

## Goals

- Deploy and update the official Grafana container through Docker Compose.
- Preserve Grafana data across container replacement using persistent storage.
- Provide configurable container settings, admin settings, and plugin configuration.
- Support Grafana provisioning through structured role variables, including datasources, dashboard providers/dashboard models, and all core unified-alerting resources (alert rules, contact points, notification policies, templates, and mute timings).
- Test deployment on Ubuntu 24.04 using `molecule_generic` to create and destroy a KubeVirt VM.
- Document requirements, role variables, usage, and how to run tests.

## Non-goals

- Installing or configuring Docker Engine or the Docker Compose plugin on production hosts.
- Managing Grafana dashboards or integrations specific to one organization.
- Supporting multiple Grafana instances or container orchestration beyond Docker Compose.

## Constraints

- The repository currently has only a minimal README and no existing role implementation.
- Production targets are expected to have Docker Engine and the Compose v2 plugin available.
- Molecule test infrastructure must have access to a KubeVirt cluster; `molecule_generic`'s KubeVirt backend is selected because a Docker-backed Ubuntu container is not a real Docker host.
- Follow Eclipse SLM role conventions where applicable, including role defaults, metadata, dependency declarations, and Molecule lifecycle integration.

## Approaches

1. **Manage a single container directly with `community.docker.docker_container`.** This is the simplest option for a one-container service, but does not use a Compose project.
2. **Manage Grafana through Docker Compose (chosen).** This supports a conventional, reusable Compose project and fits the requested Docker-host deployment style. The role will manage the Compose definition and its lifecycle.

The Compose approach was selected by the user. For Molecule, the user selected `molecule_generic` with its KubeVirt backend and an Ubuntu 24.04 VM, rather than a Docker-backed Ubuntu container with access to the runner's Docker socket.

## Scope and behavior

- The role assumes Docker and Compose are already installed on the target.
- Manage a configurable Compose project for the official Grafana image, with configurable version, ports, environment, restart behavior, and persistent data.
- Render Grafana settings and provisioning from structured Ansible variables. Keep credentials overridable and do not include real secrets in repository defaults or examples.
- Provisioning should cover datasource definitions, dashboard providers and caller-supplied dashboard model data, plus alert rules, contact points, notification policies, templates, and mute timings. The role should generate the corresponding Grafana provisioning files and mount them read-only where appropriate.
- Configure the admin username/password through Grafana's initial-startup environment only. Changing the password variable after the persistent database has initialized does not rotate the existing Grafana password; document that password changes must be performed in Grafana.
- Support plugin installation through a caller-supplied plugin list; advanced plugin-specific configuration is out of scope.
- Make container lifecycle idempotent and allow callers to override the Compose project and storage locations.
- Verify the deployed service and representative provisioning behavior in Molecule; destroy test instances through `molecule_generic`.

## Left to the builder

- Choose a clear, consistent variable prefix and detailed variable schema, aligned with Eclipse SLM role conventions.
- Select a configurable Grafana image version and a secure development/test admin-password strategy; verify the default version in Molecule.
- Decide the precise persistent storage implementation (named volume or host bind mount) and Compose project layout.
- Choose appropriate `community.docker` collection modules and test verification mechanisms.
- Confirm whether KubeVirt-specific role dependencies and credentials can be documented and installed through the scenario's `requirements.yml` without embedding environment-specific cluster settings.
