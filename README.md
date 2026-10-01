# Ansible Role: freeradius

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-freeradius)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-freeradius)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-freeradius)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/main.yml?query=branch%3Amain)

Install and configure a FreeRADIUS server with explicit clients and local PAP
authentication.

## Purpose

Install, configure, enable and start the distribution FreeRADIUS 3 service with
explicit RADIUS clients and local PAP authentication. Repeated application with
unchanged inputs makes no changes.

## Scope

### Managed

- Distribution packages, the complete radiusd.conf and a dedicated local users
  file.
- One IPv4 UDP authentication listener, explicit client networks and local PAP
  passwords.
- Native configuration validation and service restarts after configuration
  changes.

### Not Managed

- EAP, TLS, RadSec, accounting, proxying, LDAP, SQL and dynamic VLAN assignment.
- Firewall rules, certificates and network transport encryption.

## Requirements

- RADIUS clients must send Message-Authenticator on every Access-Request.
- community.general provides the zypper backend on openSUSE.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
```

## Role Variables

### `freeradius_listen_address`

Type: `str`. Required: `false`.

IPv4 address on which to accept authentication requests.

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

Authoritative list of RADIUS clients; every client must send
Message-Authenticator.

Default:

```yaml
freeradius_clients: []
```

### `freeradius_users`

Type: `list`. Required: `false`.

Authoritative local PAP users; an empty list denies all authentication.

Default:

```yaml
freeradius_users: []
```

### `freeradius_backup`

Type: `bool`. Required: `false`.

Back up managed configuration before replacement; backups can contain
credentials.

Default:

```yaml
freeradius_backup: false
```

## Managed Files

- `/etc/freeradius/3.0/radiusd.conf` Main configuration on Debian and Ubuntu.
- `/etc/freeradius/3.0/ansible-users` Local PAP users on Debian and Ubuntu.
- `/etc/raddb/radiusd.conf` Main configuration on AlmaLinux, Fedora and
  openSUSE.
- `/etc/raddb/ansible-users` Local PAP users on AlmaLinux, Fedora and openSUSE.
- `/usr/local/libexec/freeradius-validate-config` Adapter for native validation
  of a main configuration candidate.

## Check Mode

Check mode predicts changes on an already provisioned host without restarting
the daemon.

- Initial installation in check mode cannot validate or use packages and service
  accounts that have not been installed yet.
- The full installed-configuration check runs during normal convergence; check
  mode does not apply candidate user changes.

## Service Behavior

The service is always enabled and started; changed configuration is validated
before a restart.

### Handlers

- Restart the distribution freeradius or radiusd service after successful
  configuration validation.

## Security Notes

- Only loopback is bound by default. Empty client and user lists grant no
  access; no demonstration credentials are installed.
- Every client requires Message-Authenticator. Shared secrets must have at least
  16 characters and should be randomly generated.
- PAP over UDP does not provide transport encryption. Deploy only on a trusted
  network or through an encrypted tunnel.
- Store client secrets and user passwords in Ansible Vault. Secret-bearing tasks
  suppress output and diffs.
- Managed configuration is readable only by root and the service group.
  Credential backups are disabled by default.

## Operational Notes

- Each client requires name, address and secret. Names accept letters, digits,
  underscores and hyphens; addresses accept IPv4 or IPv4 CIDR. Secrets accept 16
  to 128 letters, digits or characters from ._~!@#%^&*+=:/?-. Each user requires
  name and password. Usernames also accept dots and at signs; DEFAULT is
  reserved. Passwords must be nonempty and contain no NUL, CR or LF characters.
- This role replaces the complete main configuration. Distribution example
  clients, virtual servers and modules are not loaded. The client and user lists
  are authoritative; removing an entry revokes it on the next convergence.
- FreeRADIUS accepts a configuration directory rather than an arbitrary main
  filename. The template validator checks the actual main candidate in a
  temporary directory. A second native check validates the complete installed
  configuration, including users, before handlers run. Failed user validation
  leaves the candidate on disk and fails the play before restart.
- The native -C check does not prove that the listener can bind; service startup
  and the Molecule protocol checks cover runtime behavior.

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

### Local PAP authentication

The vault variables contain a random shared secret and the local user's password.

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
          password: "{{ vault_radius_alice_password }}"
```

## References

- [FreeRADIUS configuration](https://wiki.freeradius.org/config/Configuration-files)
- [Native configuration check](https://www.freeradius.org/radiusd/man/radiusd.html)
- [RADIUS protocol testing](https://www.freeradius.org/radiusd/man/radclient.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
