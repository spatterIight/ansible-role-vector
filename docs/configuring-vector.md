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
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Vector

This is an [Ansible](https://www.ansible.com/) role which installs [Vector](https://vector.dev/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Vector is a high-performance observability data pipeline that lets you collect, transform, and route logs and metrics from many sources to many destinations (sinks).

See the project's [documentation](https://vector.dev/docs/) to learn what Vector does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Vector with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# vector                                                               #
#                                                                      #
########################################################################

vector_enabled: true

########################################################################
#                                                                      #
# /vector                                                              #
#                                                                      #
########################################################################
```

To extend the default configuration we show you a few built-in log sources you can easily enable below. To build your own extend the default sources, transforms, and sinks through the `vector_sources_custom`, `vector_transforms_custom`, and `vector_sinks_custom` variables.

### Collecting system logs (journald and `/var/log`)

To collect the host's systemd journal and/or textual log files under `/var/log`, enable the built-in log sources:

```yaml
# Collect the host's systemd journal
vector_journald_source_enabled: true

# Collect textual log files found under /var/log
vector_varlog_source_enabled: true
```

Each enabled source becomes a stream you reference in a sink's `inputs`:

- `journald` — carries a `service_name` field (derived from the systemd unit, falling back to the syslog identifier).
- `varlog` — carries a `file` field with the source file's path.

### Shipping logs to Grafana Loki

If [Grafana Loki](grafana-loki.md) runs on the same server, Vector automatically joins Loki's container network when `loki_enabled` is set, so you can ship logs to it directly over the container network.

Add `loki` sinks with the following additional configuration. Use a separate sink per stream, since a sink applies one `labels` block to all its inputs and `journald`/`varlog` carry different fields:

```yaml
vector_sinks_custom:
  loki_journald:
    type: loki
    inputs:
      - journald
    endpoint: "{{ loki_scheme }}://{{ loki_identifier }}:{{ loki_server_http_listen_port }}"
    tenant_id: mash
    encoding:
      codec: text
    labels:
      source: vector
      service_name: "{{ '{{ service_name }}' }}"
      host: "{{ '{{ host }}' }}"

  loki_varlog:
    type: loki
    inputs:
      - varlog
    endpoint: "{{ loki_scheme }}://{{ loki_identifier }}:{{ loki_server_http_listen_port }}"
    tenant_id: mash
    encoding:
      codec: text
    labels:
      source: vector
      filename: "{{ '{{ file }}' }}"
      host: "{{ ansible_hostname | default(inventory_hostname) }}"
```

> [!WARNING]
> A `{{ '{{ field }}' }}` label must reference a field present and non-empty on **every** event the sink receives, otherwise Vector drops the event and logs a warning. Never route `internal_logs` into such a sink: it carries those warnings, so a failed render feeds back into the sink and can pin the CPU. Only label on guaranteed fields (`service_name` on `journald`, `file` on `varlog`); `unit` and `syslog_identifier` are empty for unit-less entries like kernel messages, so they are not safe to label on.

For connecting to a remote Loki instance, set `endpoint` to the public hostname (e.g. `https://mash.example.com/loki`) and adjust authentication as needed.

You can then add Loki as a datasource in Grafana — refer to [Integrating with a local Loki instance](grafana.md#integrating-with-a-local-loki-instance) on the Grafana documentation page.

### Exposing metrics to Prometheus

To let [Prometheus](prometheus.md) collect Vector's metrics, add a `prometheus_exporter` sink that exposes them on a port:

```yaml
vector_sinks_custom:
  prometheus:
    type: prometheus_exporter
    inputs:
      - internal_metrics
    address: 0.0.0.0:9598
```

Then, on the Prometheus side, add a scrape job targeting Vector over the container network (Prometheus needs to share Vector's network, so this is configured on the Prometheus side):

```yaml
prometheus_config_scrape_configs_additional:
  - job_name: vector
    metrics_path: /metrics
    static_configs:
      - targets:
          - "{{ vector_identifier }}:9598"
```

Refer to [Scraping other exporter services](prometheus.md#scraping-other-exporter-services) on the Prometheus documentation page for more details.

### Exposing the API (optional)

Vector ships a GraphQL API (with a `/health` endpoint and an interactive `/playground`) that powers `vector top` / `vector tap`. It is disabled by default. To enable it and expose it publicly through [Traefik](traefik.md), set a hostname:

```yaml
vector_api_enabled: true
vector_hostname: mash.example.com
vector_path_prefix: /vector
```

>[!WARNING]
> Vector's API has no authentication of its own. Whenever you expose it publicly, protect it with HTTP Basic Authentication:
>
> ```yaml
> vector_container_labels_api_middleware_basic_auth_enabled: true
> # See https://doc.traefik.io/traefik/middlewares/http/basicauth/#users for the format.
> vector_container_labels_api_middleware_basic_auth_users: ""
> ```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `vector_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

By default Vector is configured to output its own internal logs and write them as JSON to the console. After running the command for installation you can observe its output with:

```sh
journalctl -fu mash-vector
```

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu vector` (or how you/your playbook named the service, e.g. `mash-vector`).
