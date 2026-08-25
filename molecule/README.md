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

Currently these testing scenarios are available:

### `default`

Tests a standard Mozhi installation, against the released `codeberg.org/aryak/mozhi:latest` image. Runs on every push.

### `default-selfbuild`

Tests a Mozhi installation that builds the container image on the machine, from Mozhi's source at the ref `mozhi_container_image_self_build_repo_version` pins. Runs on demand (`workflow_dispatch`) and whenever a `*_version:` line in `defaults/main.yml` changes, since the build is slow and its result only moves when upstream or that pin does.

## What the scenarios actually check

Mozhi is a front-end for third-party translation engines, so a test suite for it must be careful about what it can honestly assert. **Nothing here depends on Google, DeepL, Reverso or any other engine answering**, and nothing here makes a translation request. It does not have to: Mozhi's language lists are compiled into the binary (they are generated into libmozhi's source as Go literals) and its engine list is computed from environment variables alone, so the whole served UI renders with no network access at all.

What is checked instead is that the role's configuration reaches the running process, and that is only evidence if an unconfigured Mozhi would not serve the same thing. So the `default` scenario starts a **negative control** first: the same image, with none of the role's configuration, and records what it serves. An unconfigured Mozhi answers `200` on `/` too, listens on port 3000, offers Google (and hides LibreTranslate, which it will not offer without a URL), and preselects `auto` and `en` on its front page. The scenario is configured to differ from all of that in every one of those respects, so that:

- getting an answer on the port the role configured at all means `MOZHI_PORT` reached the process
- `/api/engines` returning a set that has LibreTranslate and is missing Google is the reverse of what the stock image returns
- the front page preselecting a third engine, a source language Mozhi's own fallback (`auto`) is not even offered by that engine, and a target language other than `en`, cannot have come from anywhere but the role's env file

On top of that: the labels file the role renders is checked on the container, the volume from `mozhi_container_additional_volumes` is checked to be mounted, the container is checked to be read-only with all capabilities dropped and running as the configured user, the container is checked to have been created from the image the tag points at right now, and the restart counter is sampled twice 45 seconds apart — because `Restart=always` makes `systemctl is-active` say `active` for a container that is crash-looping.

Mozhi has no honest version surface to assert against: `/api/version` reports the version of the Fiber framework and of the Go toolchain it was built with, not a Mozhi version, and the image carries no OCI labels at all. Upstream publishes only a `latest` tag, so there is no version for the role to name either. See [`.github/renovate.json`](../.github/renovate.json) for what that means for automerging.

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
