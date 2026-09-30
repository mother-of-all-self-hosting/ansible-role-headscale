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

See the project's [documentation](https://headscale.net/stable/usage/getting-started/) to learn what Headscale does and why it might be useful to you.

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

### Single-Sign-On (SSO) integration

Headscale supports Single-Sign-On (SSO) via OIDC. To make use of it, an Identity Provider (IdP) like [authentik](authentik.md), [Authelia](https://www.authelia.com/), [Keycloak](keycloak.md) or [Tinyauth](tinyauth.md) needs to be set up.

As Headscale's built-in authentication is somewhat manual, setting up OIDC can provide a smoother user experience.

For example, you can enable SSO with authentik via OIDC by following the steps below.

Here, we are using Ansible Vault to supply both our `domain` as well as `client_id` and `client_secret`. Add the following configuration to your `vars.yml` file. This assumes that you picked the slug `headscale` in authentik when adding Headscale as an application. If not, replace `headscale` in the `headscale_config_oidc_issuer` value.

```yaml
headscale_config_oidc_enabled: true
headscale_config_oidc_issuer: "https://authentik.{{ domain }}/application/o/headscale/"
headscale_config_oidc_client_id: "{{ vault_headscale_client_id }}"
headscale_config_oidc_client_secret: "{{ vault_headscale_client_secret }}"
headscale_config_oidc_pkce_enabled: true

# To add custom scopes on top of the defaults (openid, profile, email),
# use headscale_config_oidc_scope_custom. For example:
# headscale_config_oidc_scope_custom:
#   - groups
```

> [!NOTE]
> The `headscale_config_oidc_email_verified_required` variable defaults to `true`, meaning only verified email addresses can authenticate via OIDC. If your Identity Provider does not send the `email_verified: true` claim, you may need to set `headscale_config_oidc_email_verified_required: false`.

You can find more details about configuring OIDC by referring to the documentation at both [Headscale](https://headscale.net/stable/ref/oidc/?h=oidc) and [authentik](https://integrations.goauthentik.io/networking/headscale/). Note that Headscale's documentation doesn't explicitly cover authentik.

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

- first, [create some users](#creating-users)
- then, [connect some devices](#connecting-devices) by the official Tailscale applications, configured to talk to your own Headscale server

### Convenience script to call the binary

We provide a `/mash/headscale/bin/headscale` script on the server, which forwards commands to the `headscale` binary inside the container (`mash-headscale`).

Example usage: `/mash/headscale/bin/headscale version`

> [!WARNING]
> Command arguments which contain spaces may not be forwarded correctly.

### Creating users

To [create a user](https://headscale.net/stable/usage/getting-started/#create-a-user), run a command like this:

```sh
/usr/bin/env docker exec -it mash-headscale \
headscale users create \
john.doe \
--display-name "John Doe" \
--email "john.doe@example.com"
```

> [!WARNING]
> We use `docker exec` here because the [convenience script](#convenience-script-to-call-the-binary) does not handle forwarding arguments with spaces (like `--display-name`) correctly.

💡 You can [list the existing users](https://headscale.net/stable/usage/getting-started/#list-existing-users) with a command like this: `/mash/headscale/bin/headscale users list`

💡 Creating users is not strictly required. You can also connect devices using [pre-auth keys](#connecting-linux-devices-with-a-preshared-key) without creating users first.

### Connecting devices

Here are some quick guides for the various platforms:

- [Android devices](https://headscale.net/stable/usage/connect/android/)
- [Apple devices](https://headscale.net/stable/usage/connect/apple/)
- [Windows devices](https://headscale.net/stable/usage/connect/windows/)
- Linux: install the `tailscale` CLI. See the official [Setting up Tailscale on Linux](https://tailscale.com/kb/1031/install-linux) documentation, or the [Archlinux Tailscale Wiki page](https://wiki.archlinux.org/title/Tailscale) (and specifically its [Third-party clients](https://wiki.archlinux.org/title/Tailscale#Third-party_clients) section for GUI clients).

All of these platforms would require confirmation after initial login, so consult the [Connecting Linux devices with manual confirmation](#connecting-linux-devices-with-manual-confirmation) section below for details on how to do it.

#### Connecting Linux devices with manual confirmation

To connect a Linux device, you can use a `tailscale up` command like this:

```sh
tailscale up --login-server=https://headscale.example.com
```

💡 You may wish to add additional arguments to this command, such as `--hostname`, `--advertise-exit-node`, `--advertise-routes`, etc. These settings can also be configured later using `tailscale set` (e.g. `tailscale set --hostname=custom-hostname-for-my-device`).

Running the `tailscale up` command will print a URL you need to open in your browser to complete the setup.

The URL would contain a `headscale` command you need to run. It looks something like this:

```sh
headscale nodes register --user USERNAME --key mkey:....
```

Take this command and:

- replace the `headscale` prefix with `/mash/headscale/bin/headscale`
- replace `USERNAME` with the username of a valid [user you created](#creating-users) earlier
- run it on the Headscale server

#### Connecting Linux devices with a preshared key

Instead of following the manual back-and-forth flow as specified in [Connecting Linux devices with manual confirmation](#connecting-linux-devices-with-manual-confirmation), you can also use a preshared key to connect your device.

**First**, generate a preshared key:

```sh
/mash/headscale/bin/headscale preauthkeys create
```

> [!TIP]
> You may optionally associate the key with a user by passing `--user=NUMERIC_USER_ID` (e.g. `--user=1`). To find a user's numeric ID, run: `/mash/headscale/bin/headscale users list --name=john.doe`

**Then**, connect your device with the preshared key:

```sh
tailscale up --login-server=https://headscale.example.com --auth-key=...
```

The device will be automatically connected to the Headscale server, without any additional approval steps.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu headscale` (or how you/your playbook named the service, e.g. `mash-headscale`).
