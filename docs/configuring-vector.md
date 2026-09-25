<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Seerr

This is an [Ansible](https://www.ansible.com/) role which installs [Seerr](https://github.com/seerr-team/seerr) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Seerr is a media request and discovery manager with support for [Jellyfin](https://jellyfin.org/), [Plex](https://plex.tv/), and [Emby](https://emby.media/).

See the project's [documentation](https://docs.seerr.dev/) to learn what Seerr does and why it might be useful to you.

>[!NOTE]
> If you are looking for an Ansible role for Jellyfin and Plex, you can check out [ansible-role-jellyfin](https://github.com/spatterIight/ansible-role-jellyfin) and [ansible-role-plex](https://github.com/spatterIight/ansible-role-plex), both of which are maintained by me.

## Adjusting the playbook configuration

To enable Seerr with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# seerr                                                                #
#                                                                      #
########################################################################

seerr_enabled: true

########################################################################
#                                                                      #
# /seerr                                                               #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Seerr you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
seerr_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

>[!NOTE]
> The `seerr_path_prefix` variable can be adjusted to host under a subpath (e.g. `seerr_path_prefix: /seerr`), but this hasn't been tested yet.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `seerr_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Seerr becomes available at the specified hostname like `https://example.com`.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu seerr` (or how you/your playbook named the service, e.g. `mash-seerr`).
