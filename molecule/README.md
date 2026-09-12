<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there is one testing scenario available.

### `default`

Deploys Vector and then pushes an event all the way through the pipeline the role rendered.

`prepare.yml` writes a seed log file into a directory on the host. The scenario hands that directory to the role as a read-only `vector_container_additional_volumes_custom` entry, adds a `file` source reading it, a `remap` transform stamping a marker onto every event, and a `file` sink writing back into Vector's data directory. `verify.yml` then asserts that:

- the running Vector binary reports the version `vector_version` asks for — this is what makes a Renovate version bump a testable change rather than a string edit
- the seeded event came out of the sink carrying the marker only the role-rendered transform could add
- a log file written *after* Vector started also came out of the sink, so the pipeline is live rather than having read a file once
- Vector's API answers `/health` on `vector_container_api_port`, and nothing answers on Vector's own default port (8686), which the scenario deliberately does not use
- the container runs non-root, with all capabilities dropped and a read-only root filesystem, carries the additional volume read-only, carries `vector_container_extra_arguments`, and carries no Traefik labels while Traefik support is off
- systemd has not restarted the unit — `Restart=always` would otherwise report a crash-looping container as `active`

An unconfigured `timberio/vector` image exits immediately with code 78 (`Config file not found in path`) and ships no fallback configuration, so none of the above can pass against an instance the role did not configure.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
