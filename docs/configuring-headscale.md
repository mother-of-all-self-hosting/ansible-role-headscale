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

# Setting up Headscale

This is an [Ansible](https://www.ansible.com/) role which installs [Headscale](https://headscale.net/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

Headscale is an open-source, self-hosted implementation of the [Tailscale](https://tailscale.com/) control server.

Refer to the project's [documentation](https://headscale.net/stable/usage/getting-started/) to learn what Headscale does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Headscale with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# headscale                                                            #
#                                                                      #
########################################################################

headscale_enabled: true

########################################################################
#                                                                      #
# /headscale                                                           #
#                                                                      #
########################################################################
```

### Set the hostname

To enable Headscale you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
headscale_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

**Note**: hosting Headscale under a subpath (by configuring the `headscale_path_prefix` variable) does not seem to be possible due to Headscale's technical limitations.

### Configuring ports

#### Configuring embedded DERP server

Headscale serves the embedded DERP relay through its normal HTTPS endpoint, but its STUN listener uses a separate UDP port. To publish the default UDP port from the container, add the following configuration to your `vars.yml` file:

```yaml
headscale_config_derp_server_enabled: true

headscale_container_derp_stun_bind_port: 3478
```

Publication is opt-in to preserve existing deployments. The host firewall and any external router or firewall must also allow and, when applicable, forward UDP 3478. Headscale advertises the port from `headscale_config_derp_server_stun_listen_addr`, so when publishing it that listener port must match `headscale_container_derp_stun_port` and be reachable externally under the same public port number.

For direct Docker publication, the host port in `headscale_container_derp_stun_bind_port` should also match. A different Docker host port requires an external router or firewall to forward the advertised public UDP port to it. Headscale does not currently support configuring a separate advertised STUN port.

##### Avoiding port conflicts

Publishing UDP 3478 conflicts with another service already bound to that port on an overlapping host IP address.

If UDP 3478 is occupied, choose a free port and change both the container listener and host publication by adding the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
headscale_config_derp_server_enabled: true

headscale_container_derp_stun_port: 3480
headscale_container_derp_stun_bind_port: 3480
```

It is also necessary to remove or update any existing listener override. Please note that changing only the host publication does not change the port advertised to clients.

##### Advertised addresses

By default, `headscale_config_derp_server_ipv4` and `headscale_config_derp_server_ipv6` are empty, so clients resolve the hostname from `headscale_config_server_url` (normally derived from `headscale_hostname`). Headscale recommends setting the embedded DERP server's actual public IPv4 and IPv6 addresses for better connection stability, especially when DNS is unavailable.

##### Updating an existing deployment

If `headscale_container_extra_arguments_custom` already contains a manual STUN `-p` mapping, remove that mapping before setting `headscale_container_derp_stun_bind_port` to avoid publishing the same container port twice.

Existing explicit STUN listener overrides must use a valid `host:port` value and match `headscale_container_derp_stun_port` when the role publishes the listener. If you override the configuration or extend it with `headscale_configuration_extension_yaml`, keep the resulting listener aligned with the container publication as well.

The address defaults no longer use documentation-only IP addresses. Explicit `headscale_config_derp_server_ipv4` and `headscale_config_derp_server_ipv6` overrides are preserved; replace any copied example addresses with your server's actual public addresses, or remove the overrides to use DNS.

#### Other container port publications

HTTP API and metrics publications use `headscale_container_http_api_port` and `headscale_container_http_metrics_port` as their container targets. If you override `headscale_config_listen_addr` or `headscale_config_metrics_listen_addr`, keep its port aligned with the corresponding container port.

`headscale_container_grpc_bind_port` publishes the configured `headscale_container_grpc_port` over TCP. **Please review existing nonempty gRPC bindings before upgrading:** previously ignored values now take effect and may expose the listener on the host. Leave the bind variable empty if direct host publication is not wanted, or select the intended host IP and port. Please note that publication alone does not configure gRPC TLS or authentication.

### Configuring Single-Sign-On (SSO) integration

Headscale supports Single-Sign-On (SSO) via OIDC. To make use of it, an Identity Provider (IdP) like [authentik](https://goauthentik.io/), [Authelia](https://www.authelia.com/), [Keycloak](https://www.keycloak.org/) or [Tinyauth](https://tinyauth.app) needs to be set up.

As Headscale's built-in authentication is somewhat manual, setting up OIDC can provide a smoother user experience.

For example, you can enable SSO with authentik via OIDC by adding the following configuration to your `vars.yml` file (adapt to your needs). Here Ansible Vault is used to supply both our `domain` as well as `client_id` and `client_secret`.

```yaml
headscale_config_oidc_enabled: true
headscale_config_oidc_issuer: "https://authentik.{{ domain }}/application/o/headscale/"
headscale_config_oidc_client_id: "{{ vault_headscale_client_id }}"
headscale_config_oidc_client_secret: "{{ vault_headscale_client_secret }}"
headscale_config_oidc_pkce_enabled: true

# You can add custom scopes on top of the defaults (openid, profile, email) with `headscale_config_oidc_scope_custom`. For example:
# headscale_config_oidc_scope_custom:
#   - groups
```

> [!NOTE]
>
> - This assumes that you picked the slug `headscale` in authentik when adding Headscale as an application. If not, replace `headscale` in the `headscale_config_oidc_issuer` value.
> - The `headscale_config_oidc_email_verified_required` variable defaults to `true`, meaning only verified email addresses can authenticate via OIDC. If your Identity Provider does not send the `email_verified: true` claim, you may need to set `headscale_config_oidc_email_verified_required: false`.

You can find more details about configuring OIDC by referring to the documentation at both [Headscale](https://headscale.net/stable/ref/oidc/?h=oidc) and [authentik](https://integrations.goauthentik.io/networking/headscale/).

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `headscale_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, Headscale becomes available at the specified hostname like `https://example.com`.

After installation, you would normally:

1. Create users
2. Connect devices with official Tailscale applications, configured to talk to your own Headscale server

### Creating users (optional)

💡 Creating users is not strictly required. You can also connect devices using [pre-auth keys](#connecting-linux-devices-with-a-preshared-key) without creating users first.

After logging in with SSH to the server where Headscale is installed, you can create a user by running a command like this (if Headscale is managed with the MASH playbook):

```sh
/usr/bin/env docker exec -it mash-headscale \
headscale users create \
john.doe \
--display-name "John Doe" \
--email "john.doe@example.com"
```

Refer to [this page](https://headscale.net/stable/usage/getting-started/#create-a-user) on the Headscale's official documentation for details about the command.

>[!NOTE]
> We use `docker exec` here because the [convenience script](#convenience-script-to-call-the-binary) does not handle forwarding arguments with spaces (like `--display-name`) correctly.

If you want to [list the existing users](https://headscale.net/stable/usage/getting-started/#list-existing-users), run a command as below:

```sh
/mash/headscale/bin/headscale users list
```

### Connecting devices

Here are some quick guides for the various platforms:

- [Android devices](https://headscale.net/stable/usage/connect/android/)
- [Apple devices](https://headscale.net/stable/usage/connect/apple/)
- [Windows devices](https://headscale.net/stable/usage/connect/windows/)
- Linux: install the `tailscale` CLI. Refer to this [official documentation](https://tailscale.com/kb/1031/install-linux) about setting up Tailscale on Linux. [Archlinux Tailscale Wiki page](https://wiki.archlinux.org/title/Tailscale) (and specifically its [Third-party clients](https://wiki.archlinux.org/title/Tailscale#Third-party_clients) section for GUI clients) is available too.

All of these platforms will require confirmation after initial login, so consult the section below for details.

#### Connecting Linux devices with manual confirmation

To connect a Linux device with manual confirmation, you can run a `tailscale up` command like this:

```sh
tailscale up --login-server=https://headscale.example.com
```

>[!NOTE]
> You may wish to add additional arguments to this command, such as `--hostname`, `--advertise-exit-node`, `--advertise-routes`, etc. These settings can also be configured later using `tailscale set` (e.g. `tailscale set --hostname=custom-hostname-for-my-device`).

Running the `tailscale up` command will print a URL you need to open in your browser to complete the setup. The URL should contain a `headscale` command you need to run. It looks something like this:

```sh
headscale nodes register --user USERNAME --key mkey:....
```

Take this command and:

- replace the `headscale` prefix with `/mash/headscale/bin/headscale` (adjust the path as necessary)
- replace `USERNAME` with the username of a valid [user you created](#creating-users-optional) earlier
- run it on the Headscale server

#### Connecting Linux devices with a preshared key

Instead of following the manual back-and-forth flow, you can also use a preshared key to connect your device.

First, generate a preshared key:

```sh
/mash/headscale/bin/headscale preauthkeys create
```

>[!NOTE]
> You may optionally associate the key with a user by passing `--user=NUMERIC_USER_ID` (e.g. `--user=1`). To find a user's numeric ID, run: `/mash/headscale/bin/headscale users list --name=john.doe`

Then, connect your device with the preshared key:

```sh
tailscale up --login-server=https://headscale.example.com --auth-key=...
```

The device will be automatically connected to the Headscale server, without any additional approval steps.

### Convenience script to call the binary

The installation command sets up a `/mash/headscale/bin/headscale` script on the server. It can be used to forward commands to the `headscale` binary inside the container. Make sure to adjust the path to the binary as necessary.

Example usage: `/mash/headscale/bin/headscale version`

>[!WARNING]
> Command arguments which contain spaces may not be forwarded correctly.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu headscale` (or how you/your playbook named the service, e.g. `mash-headscale`).
