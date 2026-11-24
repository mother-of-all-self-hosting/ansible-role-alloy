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
SPDX-FileCopyrightText: 2026 spatterIight
SPDX-FileCopyrightText: 2026 sudo-Tiz

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Alloy

This is an [Ansible](https://www.ansible.com/) role which installs [Grafana Alloy](https://grafana.com/docs/alloy/latest/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Alloy is Grafana's OpenTelemetry Collector distribution and the successor to [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/) (which reached end-of-life status on March 2, 2026). This role currently configures Alloy to collect logs (from the systemd journal and/or plain-text log files under `/var/log`) and forward them to one or more [Grafana Loki](https://grafana.com/oss/loki/)-compatible endpoints, mirroring what the Promtail role used to do. Alloy is capable of a lot more than this (metrics, traces, and many other data sources), and you can extend the generated configuration with your own additional components via `alloy_configuration_extension_alloy`.

See the project's [documentation](https://grafana.com/docs/alloy/latest/) to learn what Alloy does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Alloy with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# alloy                                                                #
#                                                                      #
########################################################################

alloy_enabled: true

########################################################################
#                                                                      #
# /alloy                                                               #
#                                                                      #
########################################################################
```

### Choosing log sources

Because **no log sources are enabled** by default, Alloy does not do anything in its default configuration. Enable the systemd journal or `/var/log` file scraper explicitly:

```yaml
alloy_journald_scraper_enabled: true
alloy_varlog_scraper_enabled: true
```

The journal scraper keeps all systemd units by default, except Alloy itself. Set `alloy_journald_scraper_config_relabel_configs_keep_regex` to an allowlist when only selected units should be collected. The file scraper matches files under `/var/log` ending in `log` by default; adjust `alloy_varlog_scraper_config_labels_path` for a narrower or broader glob.

Collected logs can contain credentials or personal data. Limit sources and labels, protect the Loki endpoint, and choose a retention period appropriate for the data.

### Sending logs to Loki

Define at least one Loki-compatible client:

```yaml
alloy_config_clients_custom:
  - url: https://loki.example.com/loki/api/v1/push
    tenant_id: example
```

If you use the MASH playbook, enabling both Alloy and its Loki component wires the endpoint automatically. Use `alloy_config_clients_custom` for additional endpoints or a remote Loki instance.

### Exposing the web interface

By default, Alloy's UI, API and metrics endpoint are not exposed externally.

To expose them, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
alloy_hostname: "example.com"

# Expose Alloy's whole UI and API
alloy_container_labels_traefik_api_enabled: true

# Expose just its metrics endpoint
alloy_container_labels_traefik_metrics_enabled: true
```

Do not expose the UI or API publicly unless access is restricted, for example with HTTP Basic Authentication via the `alloy_container_labels_traefik_api_traefik_middleware_basic_auth_*` and `alloy_container_labels_traefik_metrics_traefik_middleware_basic_auth_*` variables.

### Extending the configuration

`alloy_configuration_extension_alloy` is appended to the role-generated configuration and can define additional Alloy components. Any Docker socket mounted for them should be treated as a highly privileged access path.

For the full set of role options, see [`defaults/main.yml`](../defaults/main.yml) and the [Alloy documentation](https://grafana.com/docs/alloy/latest/).

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Alloy becomes available.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu alloy` (or how you/your playbook named the service, e.g. `mash-alloy`).
