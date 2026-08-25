<!--
SPDX-FileCopyrightText: 2020 - 2024 MDAD project contributors
SPDX-FileCopyrightText: 2020 - 2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 - 2025 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Mozhi

This is an [Ansible](https://www.ansible.com/) role which installs [Mozhi](https://codeberg.org/aryak/mozhi) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Mozhi is a frontend for translation engines which supports Google, Reverso, LibreTranslate, etc.

See the project's [documentation](https://codeberg.org/aryak/mozhi/src/branch/master/README.md) to learn what Mozhi does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Mozhi with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# mozhi                                                                #
#                                                                      #
########################################################################

mozhi_enabled: true

########################################################################
#                                                                      #
# /mozhi                                                               #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Mozhi you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
mozhi_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

**Note**: hosting Mozhi under a subpath (by configuring the `mozhi_path_prefix` variable) does not seem to be possible due to Mozhi's technical limitations.

### Choosing which translation engines are offered

The role exposes a switch per engine (`mozhi_environment_variables_mozhi_google_enabled`, `mozhi_environment_variables_mozhi_deepl_enabled`, and so on), each of which becomes a `MOZHI_*_ENABLED` environment variable.

**Mozhi only acts on the first of them that is switched off.** Its engine list is built by a single if/else-if chain (`EngineList` in [`utils/main.go`](https://codeberg.org/aryak/mozhi/src/branch/master/utils/main.go)), which walks the engines in the order Google, DeepL, DuckDuckGo, LibreTranslate, MyMemory, Reverso, Yandex, removes the first one it finds switched off, and then stops looking. Switching off two engines at once therefore removes only the earlier of the two, and switching off Google means no other switch has any effect at all. This is an upstream limitation rather than something this role can work around.

LibreTranslate is a further special case: Mozhi hides it unless `MOZHI_LIBRETRANSLATE_URL` is set, since it has no public instance to fall back on. This role always writes that variable into the env file, even when `mozhi_environment_variables_mozhi_libretranslate_url` is left empty, so on a default installation LibreTranslate stays in the list of offered engines and translating with it fails at request time. If you do not run a LibreTranslate instance, set:

```yaml
mozhi_environment_variables_mozhi_libretranslate_enabled: false
```

and make sure no engine earlier in the chain above is switched off, or that switch will never be reached.

### Keeping up with new Mozhi releases

Mozhi publishes no versioned container images — [its repository](https://codeberg.org/aryak/mozhi) carries no git tags and no releases, and its container registry carries only a `latest` tag. `mozhi_version` is therefore the literal `latest`, and running the playbook pulls whatever `latest` points at that day. There is no other tag to pin to.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `mozhi_environment_variables_additional_variables` variable

See [this section](https://codeberg.org/aryak/mozhi/src/branch/master/README.md#configuration) on the documentation for a complete list of Mozhi's config options that you could put in `mozhi_environment_variables_additional_variables`.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Mozhi becomes available at the specified hostname like `https://example.com`.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu mozhi` (or how you/your playbook named the service, e.g. `mash-mozhi`).
