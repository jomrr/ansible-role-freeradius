# Ansible Role: freeradius

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-freeradius)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-freeradius)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-freeradius)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/main.yml?query=branch%3Amain)

Configure FreeRADIUS with local PAP authentication and optional buffered
PostgreSQL accounting.

## Purpose

Run a FreeRADIUS 3 server for local PAP authentication and optional RADIUS
accounting over IPv4 UDP.

## Scope

### Managed

- Distribution packages, the complete server configuration, local users and the
  enabled, running service.
- Local PAP authentication with Crypt-Password, PBKDF2-Password,
  SSHA2-512-Password, NT-Password or Cleartext-Password credentials.
- Per-user authentication attempt limits and Accept/Reject logging.
- Optional accounting to local detail files or asynchronously to PostgreSQL,
  including PostgreSQL connection and TLS settings.
- Optional PostgreSQL schema initialization and login limits based on open SQL
  accounting sessions.

### Not Managed

- Other authentication methods such as CHAP, MSCHAPv2 and EAP; LDAP, AD and SQL
  authentication backends.
- RADIUS over TLS, RADIUS proxying and dynamic VLAN assignment.
- PostgreSQL server, database and login creation; existing-schema migrations and
  accounting retention or backups.
- Firewall rules, network tunnels and certificate provisioning.

## Requirements

- RADIUS clients must send Message-Authenticator on every Access-Request.
- PostgreSQL accounting requires an existing database and login. Use the
  distribution FreeRADIUS schema or let the role initialize it.
- Optional schema initialization requires an empty database and CREATE
  privileges for the configured database user.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: community.postgresql
    version: '>=4.2.0'
  - name: containers.podman
    version: '>=1.20.0'
```

## Role Variables

### `freeradius_listen_address`

Type: `str`. Required: `false`.

IPv4 address on which to accept authentication and enabled accounting requests.

Default:

```yaml
freeradius_listen_address: 127.0.0.1
```

### `freeradius_port`

Type: `int`. Required: `false`.

UDP authentication port, from 1 to 65535.

Default:

```yaml
freeradius_port: 1812
```

### `freeradius_clients`

Type: `list`. Required: `false`.

Authoritative clients for authentication and accounting; Access-Request requires
Message-Authenticator.

Default:

```yaml
freeradius_clients: []
```

### `freeradius_password_attribute`

Type: `str`. Required: `false`.

Default FreeRADIUS credential attribute; PAP permits hashes without requiring
cleartext storage.

Default:

```yaml
freeradius_password_attribute: Crypt-Password
```

### `freeradius_users`

Type: `list`. Required: `false`.

Authoritative local PAP users; an empty list denies all authentication.

Default:

```yaml
freeradius_users: []
```

### `freeradius_failure_limit`

Type: `int`. Required: `false`.

Maximum admitted attempts per user in a fixed window, from 1 to 1000; success
resets the counter.

Default:

```yaml
freeradius_failure_limit: 5
```

### `freeradius_failure_window`

Type: `int`. Required: `false`.

Fixed attempt window in seconds, from 10 to 86400; blocked requests do not
extend it.

Default:

```yaml
freeradius_failure_window: 300
```

### `freeradius_backup`

Type: `bool`. Required: `false`.

Back up managed configuration before replacement; backups can contain
credentials.

Default:

```yaml
freeradius_backup: false
```

### `freeradius_accounting_enabled`

Type: `bool`. Required: `false`.

Enable accounting for the configured clients on freeradius_listen_address.

Default:

```yaml
freeradius_accounting_enabled: false
```

### `freeradius_accounting_port`

Type: `int`. Required: `false`.

UDP accounting port, from 1 to 65535; must differ from the authentication port.

Default:

```yaml
freeradius_accounting_port: 1813
```

### `freeradius_accounting_backend`

Type: `str`. Required: `false`.

Keep local detail records or drain them into PostgreSQL; postgresql requires
accounting enabled.

Default:

```yaml
freeradius_accounting_backend: detail
```

### `freeradius_postgresql_host`

Type: `str`. Required: `false`.

PostgreSQL hostname or IP address; required for the postgresql backend.

### `freeradius_postgresql_port`

Type: `int`. Required: `false`.

PostgreSQL TCP port, from 1 to 65535.

Default:

```yaml
freeradius_postgresql_port: 5432
```

### `freeradius_postgresql_database`

Type: `str`. Required: `false`.

Existing database containing the distribution FreeRADIUS schema; letters,
digits, underscores or hyphens.

Default:

```yaml
freeradius_postgresql_database: radius
```

### `freeradius_postgresql_user`

Type: `str`. Required: `false`.

Database identity; letters, digits, underscores or hyphens.

Default:

```yaml
freeradius_postgresql_user: radius
```

### `freeradius_postgresql_password`

Type: `str`. Required: `false`.

Required for postgresql; 16 to 128 characters from the same alphabet as client
secrets.

### `freeradius_postgresql_ssl_mode`

Type: `str`. Required: `false`.

PostgreSQL TLS policy; verify-full verifies the certificate and hostname.

Default:

```yaml
freeradius_postgresql_ssl_mode: verify-full
```

### `freeradius_postgresql_ssl_root_cert`

Type: `path`. Required: `false`.

Absolute CA certificate path on the RADIUS host; empty uses the libpq default.

Default:

```yaml
freeradius_postgresql_ssl_root_cert: ''
```

### `freeradius_postgresql_initialize_schema`

Type: `bool`. Required: `false`.

Import the packaged schema when public.radacct is absent; requires an existing
empty database and CREATE rights.

Default:

```yaml
freeradius_postgresql_initialize_schema: false
```

### `freeradius_simultaneous_use`

Type: `int`. Required: `false`.

Open SQL sessions allowed per local user; zero disables checking, positive
values require postgresql.

Default:

```yaml
freeradius_simultaneous_use: 0
```

## Managed Files

- `/etc/freeradius/3.0/radiusd.conf` Main configuration on Debian and Ubuntu.
- `/etc/freeradius/3.0/ansible-users` Local PAP users on Debian and Ubuntu.
- `/etc/raddb/radiusd.conf` Main configuration on AlmaLinux, Fedora and
  openSUSE.
- `/etc/raddb/ansible-users` Local PAP users on AlmaLinux, Fedora and openSUSE.
- `/usr/local/libexec/freeradius-validate-config` Adapter for native validation
  of a main configuration candidate.
- `/var/log/freeradius/ansible-accounting/` Accounting directory on Debian and
  Ubuntu; FreeRADIUS writes and processes the detail files.
- `/var/log/radius/ansible-accounting/` Accounting directory on AlmaLinux,
  Fedora and openSUSE; FreeRADIUS writes and processes the detail files.

## Check Mode

Check mode predicts changes on an already provisioned host without restarting
the daemon.

- Initial installation in check mode cannot validate or use packages and service
  accounts that have not been installed yet.
- The full installed-configuration check runs during normal convergence; check
  mode does not apply candidate user changes.

## Service Behavior

The service is enabled and started. Configuration changes are validated before
restart; an already configured and running service remains unchanged when
reapplied with the same inputs.

### Handlers

- Restart the distribution freeradius or radiusd service after successful
  configuration validation.

## Security Notes

- The default listener binds to loopback. Only configured clients are admitted;
  /0 networks and duplicate client secrets are rejected.
- RADIUS authentication and accounting traffic is not transport-encrypted;
  protect the network path between clients and server.
- Store client and database secrets and user credentials in Vault. Configuration
  files use mode 0640; accounting files use 0600 inside a 0700 directory.
  Enabling freeradius_backup retains previous credentials in configuration
  backups.
- PostgreSQL defaults to verify-full TLS verification. Provide a trusted CA
  through freeradius_postgresql_ssl_root_cert or libpq's default trust file,
  readable by the service account and by root when initializing the schema.
- Authentication logs contain user and client identities, without passwords or
  stored hashes; configure retention in the host logging service.

## Operational Notes

- The role replaces radiusd.conf and the local users file. Client and user lists
  are authoritative. An empty freeradius_users list disables PAP authentication;
  enabled accounting accepts records from configured clients independently of
  the local user list.
- User passwords must match freeradius_password_attribute, with an optional
  per-user password_attribute override. The default Crypt-Password accepts
  SHA-512 crypt or yescrypt hashes; yescrypt requires support in the target
  system's libcrypt.
- Reaching freeradius_failure_limit blocks further requests, including valid
  passwords, until the fixed freeradius_failure_window expires. Success before
  the limit resets the counter. Counters reset on restart and are not shared
  between servers.
- Enable freeradius_accounting_enabled for accounting, using UDP 1813 by
  default. The detail backend keeps daily local files; postgresql forwards them
  asynchronously. A database outage buffers accounting locally and leaves PAP
  available unless a session limit is enabled.
- SQL mode consumes existing detail files and removes processed files. Keep
  pending files unchanged on persistent local storage, outside log rotation, and
  monitor free space. Buffering covers database outages, but does not guarantee
  durability on storage failure.
- Enable freeradius_postgresql_initialize_schema to initialize an empty
  database. While enabled, every role run requires database connectivity;
  disable it after initialization if convergence must work during database
  outages.
- freeradius_simultaneous_use requires PostgreSQL; zero disables the limit. A
  positive limit rejects logins when the recorded open-session count reaches the
  limit or SQL lookup fails. Accounting delays and parallel logins prevent a
  strict concurrent-session guarantee.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Buffered PostgreSQL accounting

Use an empty database with CREATE rights for optional schema initialization.

```yaml
- name: Configure authentication and accounting
  hosts: radius_servers
  gather_facts: true
  roles:
    - role: jomrr.freeradius
      freeradius_accounting_enabled: true
      freeradius_accounting_backend: postgresql
      freeradius_postgresql_host: radius-db.example.net
      freeradius_postgresql_password: "{{ vault_radius_postgresql_password }}"
      freeradius_postgresql_ssl_root_cert: /etc/ssl/certs/radius-db-ca.pem
      freeradius_postgresql_initialize_schema: true
      freeradius_simultaneous_use: 1
      freeradius_clients:
        - name: access_switch
          address: 192.0.2.20
          secret: "{{ vault_radius_client_secret }}"
      freeradius_listen_address: 192.0.2.10
      freeradius_users:
        - name: alice
          password: "{{ vault_radius_alice_password_hash }}"
```

### Local PAP authentication

Vault holds a random client secret and a SHA-512 crypt or yescrypt user hash.

```yaml
- name: Configure RADIUS authentication
  hosts: radius_servers
  gather_facts: true
  roles:
    - role: jomrr.freeradius
      freeradius_listen_address: 192.0.2.10
      freeradius_clients:
        - name: access_switch
          address: 192.0.2.20
          secret: "{{ vault_radius_client_secret }}"
      freeradius_users:
        - name: alice
          password: "{{ vault_radius_alice_password_hash }}"
```

## References

- [Buffered SQL](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/sites-available/buffered-sql)
- [FreeRADIUS configuration](https://wiki.freeradius.org/config/Configuration-files)
- [Native configuration check](https://www.freeradius.org/radiusd/man/radiusd.html)
- [RADIUS protocol testing](https://www.freeradius.org/radiusd/man/radclient.html)
- [PAP password attributes](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/mods-available/pap)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
