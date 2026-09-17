<!--
SPDX-FileCopyrightText: 2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Headscale Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [Headscale](https://headscale.net) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Embedded DERP server

Headscale serves the embedded DERP relay through its normal HTTPS endpoint, but its STUN listener uses a separate UDP port. To publish the default UDP port from the container, use:

```yaml
headscale_config_derp_server_enabled: true
headscale_container_derp_stun_bind_port: 3478
```

Publication is opt-in to preserve existing deployments. The host firewall and any external router or firewall must also allow and, when applicable, forward UDP 3478. Headscale advertises the port from `headscale_config_derp_server_stun_listen_addr`, so when publishing it that listener port must match `headscale_container_derp_stun_port` and be reachable externally under the same public port number. For direct Docker publication, the host port in `headscale_container_derp_stun_bind_port` should also match. A different Docker host port requires an external router or firewall to forward the advertised public UDP port to it. Headscale does not currently support configuring a separate advertised STUN port.

### Avoiding port conflicts

Publishing UDP 3478 conflicts with another service already bound to that port on an overlapping host IP address. For example, Coturn uses UDP 3478 by default, including when enabled by [matrix-docker-ansible-deploy (MDAD)](https://github.com/spantaleev/matrix-docker-ansible-deploy/blob/master/docs/configuring-playbook-turn.md). This is Coturn's STUN/TURN listener, not Synapse itself.

If UDP 3478 is occupied, choose a free port and change both the container listener and host publication. For example:

```yaml
headscale_config_derp_server_enabled: true
headscale_container_derp_stun_port: 3480
headscale_container_derp_stun_bind_port: 3480
```

Allow and, where necessary, forward UDP 3480 instead. The default `headscale_config_derp_server_stun_listen_addr` follows `headscale_container_derp_stun_port`; no separate override is needed. Remove or update any existing listener override so it uses the same port. Changing only the host publication does not change the port advertised to clients.

### Advertised addresses

By default, `headscale_config_derp_server_ipv4` and `headscale_config_derp_server_ipv6` are empty, so clients resolve the hostname from `headscale_config_server_url` (normally derived from `headscale_hostname`). Headscale recommends setting the embedded DERP server's actual public IPv4 and IPv6 addresses for better connection stability, especially when DNS is unavailable.

### Updating an existing deployment

If `headscale_container_extra_arguments_custom` already contains a manual STUN `-p` mapping, remove that mapping before setting `headscale_container_derp_stun_bind_port` to avoid publishing the same container port twice.

Existing explicit STUN listener overrides must use a valid `host:port` value and match `headscale_container_derp_stun_port` when the role publishes the listener. If you override the complete configuration or extend it with `headscale_configuration_extension_yaml`, keep the resulting listener aligned with the container publication as well.

The address defaults no longer use documentation-only IP addresses. Explicit `headscale_config_derp_server_ipv4` and `headscale_config_derp_server_ipv6` overrides are preserved; replace any copied example addresses with your server's actual public addresses, or remove the overrides to use DNS.

## Other container port publications

HTTP API and metrics publications use `headscale_container_http_api_port` and `headscale_container_http_metrics_port` as their container targets, rather than fixed ports 8080 and 9090. Their default configuration listeners follow these variables. If you override `headscale_config_listen_addr` or `headscale_config_metrics_listen_addr`, keep its port aligned with the corresponding container port.

`headscale_container_grpc_bind_port` now publishes the configured `headscale_container_grpc_port` over TCP. **Review existing nonempty gRPC bindings before upgrading:** previously ignored values now take effect and may expose the listener on the host. Leave the bind variable empty if direct host publication is not wanted, or select the intended host IP and port. Publication alone does not configure gRPC TLS or authentication.

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

Refer to [this page](./molecule/README.md) for details about how to utilize it.
