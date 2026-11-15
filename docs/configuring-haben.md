<!--
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 Bergruebe

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Haben

This is an [Ansible](https://www.ansible.com/) role which installs [Haben](https://haben.app), a self-hosted bookkeeping application for German freelancers and small businesses, to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

See the project's [documentation](https://haben.app/) (in German) to learn what Haben does and why it might be useful to you.

## Prerequisites

To run a Haben instance it is necessary to prepare a [Postgres](https://www.postgresql.org/) database. Haben is developed against Postgres 18 and needs the `pg_trgm` extension, which the database owner can create by itself.

If you are looking for an Ansible role for Postgres, you can check out [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

## Adjusting the playbook configuration

To enable Haben with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                                                                                            #
# haben                                                                                                                                 #
#                                                                                                                                            #
########################################################################

haben_enabled: true

########################################################################
#                                                                                                                                            #
# /haben                                                                                                                                #
#                                                                                                                                            #
########################################################################
```

### Set the hostname

To serve Haben you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
haben_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the Haben domain to your server.

Passkeys are bound to the hostname, so it should not be changed later. Hosting under a subpath is not supported.

### Configure the database

To connect Haben to Postgres, add the following configuration to your `vars.yml` file:

```yaml
haben_database_hostname: YOUR_DATABASE_HOSTNAME_HERE
haben_database_username: YOUR_DATABASE_USERNAME_HERE
haben_database_password: YOUR_DATABASE_PASSWORD_HERE
```

The user must own the database (`haben` by default). Haben runs its migrations on startup.

#### Use a separate application role (optional)

Haben can use a restricted role for the running application, and the owner only for migrations. See [Getrennte Datenbankrollen](https://haben.app/installation-konfiguration.html#getrennte-datenbankrollen) for details.

Create the role once on the Postgres server, for example:

```sh
createuser --login --no-superuser --no-createdb --no-createrole --pwprompt haben_app
```

Then add the following configuration to your `vars.yml` file:

```yaml
haben_database_app_username: haben_app
haben_database_app_password: YOUR_APP_ROLE_PASSWORD_HERE
```

### Set the secrets

Haben requires two secrets. Add the following configuration to your `vars.yml` file:

```yaml
# Signs the sessions, at least 32 characters
haben_environment_variables_better_auth_secret: YOUR_SECRET_HERE

# Generate it with: openssl rand -base64 32
haben_environment_variables_haben_encryption_key: YOUR_ENCRYPTION_KEY_HERE
```

>[!WARNING]
> The encryption key protects the ELSTER certificate and stored API keys. Keep a copy in a safe place and do not change it later, or the encrypted data cannot be restored.

### Set a fixed setup code (optional)

On the first start Haben prints a setup code to the log, which is needed to create the first account. To set a fixed one instead, add the following configuration to your `vars.yml` file:

```yaml
haben_environment_variables_haben_setup_code: YOUR_SETUP_CODE_HERE
```

### Configure Single Sign-On (optional)

Haben supports exclusive login via OpenID Connect, with roles taken from groups. To enable it, add the following configuration to your `vars.yml` file (adapt to your needs):

```yaml
haben_environment_variables_oidc_only: true
haben_environment_variables_oidc_issuer: https://auth.example.com
haben_environment_variables_oidc_client_id: haben
haben_environment_variables_oidc_client_secret: YOUR_CLIENT_SECRET_HERE
haben_environment_variables_oidc_group_inhaber: haben-inhaber
haben_environment_variables_oidc_group_steuerberatung: haben-steuerberatung
```

See [Anmeldung per SSO](https://haben.app/installation-sso.html) for details.

### Configure ELSTER (optional)

To let Haben download ERiC from the tax administration on startup, add the following configuration to your `vars.yml` file. Enabling it means accepting ERiC's terms of use.

```yaml
haben_environment_variables_eric_auto_install_enabled: true

# Your own manufacturer ID, once the test transmission succeeded
haben_environment_variables_elster_hersteller_id: "12345"
```

ERiC is only available for Linux x86_64. See [ELSTER-Anbindung](https://haben.app/installation-anbindungen.html) for details.

### Expose Haben via TSDProxy (optional)

[TSDProxy](https://almeidapaulopt.github.io/tsdproxy/) can make Haben available in your Tailscale network. As Haben accepts a single address only, set the tailnet address as the hostname and turn off the Traefik labels:

```yaml
haben_hostname: haben.YOUR-TAILNET.ts.net

haben_container_labels_traefik_enabled: false
haben_container_labels_tsdproxy_enabled: true
```

The container needs to share a network with TSDProxy. The MASH playbook connects them automatically.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file
- [the upstream documentation](https://haben.app/installation-konfiguration.html#umgebungsvariablen) for all environment variables, which you can add with `haben_environment_variables_additional_variables`

```yaml
haben_environment_variables_additional_variables: |
  ANTHROPIC_API_KEY=YOUR_API_KEY_HERE
  HABEN_FIRMEN=Studio Nord|https://nord.example.com;Werkstatt|https://werkstatt.example.com
```

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After installation, Haben becomes available at the specified hostname like `https://example.com`.

To create the first account, open the page and enter the setup code. Unless you have set one, you can find it in the logs (see below).

## Backup

The bookkeeping data is stored in the Postgres database and in the receipts directory (`haben_documents_path`). Both need to be included in your backups.

The directories for ERiC (`haben_eric_path`) and its logs (`haben_eric_log_path`) do not need to be backed up.

## Troubleshooting

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu haben` (or how you/your playbook named the service, e.g. `mash-haben`).
