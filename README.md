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
  credentials, using hashes by default.
- Per-user attempt limits and structured authentication events through syslog.
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
- Every client requires Message-Authenticator. Client secrets must be distinct,
  randomly generated and at least 16 characters long.
- Client CIDR prefixes must be between 1 and 32; unrestricted /0 networks are
  rejected before configuration changes.
- PAP over UDP does not provide transport encryption. Deploy only on a trusted
  network or through an encrypted tunnel.
- Store client secrets and user credentials in Ansible Vault. Secret-bearing
  tasks suppress output and diffs.
- PAP does not require stored cleartext passwords. Prefer hashes;
  Cleartext-Password is available only through explicit selection.
- Managed configuration is readable only by root and the service group.
  Credential backups are disabled by default.
- Authentication outcomes are logged with auth=yes and
  auth_badpass/auth_goodpass=no. Structured authpriv syslog events contain
  event, result, user_hex, client, source and stage fields. Usernames are hex
  encoded to prevent log injection; passwords and stored hashes are excluded.
  The host logging service controls retention and forwarding.
- The local rlm_cache counter admits five attempts per user in a fixed
  300-second window by default. After five failures, further requests are
  rejected, including correct passwords. A successful authentication before the
  limit resets the counter. Blocked requests do not extend the window. Only
  configured users allocate entries; the cache never stores passwords or hashes.

## Operational Notes

- Each client requires name, address and secret. Names accept letters, digits,
  underscores and hyphens; addresses accept IPv4 or IPv4 CIDR. Secrets accept 16
  to 128 letters, digits or characters from ._~!@#%^&*+=:/?-. Each user requires
  name and password. Usernames also accept dots and at signs; DEFAULT is
  reserved. The password field defaults to a Crypt-Password hash. Migrate
  previous cleartext values to hashes, or explicitly select Cleartext-Password
  with password_attribute. No format is inferred from the password value.
- freeradius_password_attribute defaults to Crypt-Password; each user can
  override password_attribute. Supported encodings are SHA-512 crypt ($6$,
  optional rounds) or yescrypt ($y$), PBKDF2-Password in
  $PBKDF2$HMACSHA2+256:rounds:base64-salt$base64-digest format (also
  HMACSHA2+512), and SSHA2-512-Password as Base64(SHA512(password + salt) +
  salt), without a header. The target system's libcrypt must support the
  selected crypt algorithm. NT-Password accepts a 32-digit hexadecimal NT hash;
  Cleartext-Password accepts a nonempty password without NUL, CR or LF. Both
  require explicit selection.
- Extension boundary: only PAP is currently implemented. Future method selection
  must explicitly validate credential compatibility: EAP-TLS uses certificates
  and no user password; PAP and TTLS/PAP accept supported hashes; MSCHAPv2 uses
  NT hashes or an AD backend (cleartext can also supply the NT hash); CHAP and
  EAP-MD5 require access to cleartext passwords. There must be no automatic
  downgrade to cleartext or activation of a method based on stored credential
  fields. TLS methods require an explicit certificate trust policy. An AD
  backend delegates account lockout to the DC rather than reusing the local-user
  cache.
- The limiter reserves attempts before checking credentials, so parallel
  requests consume the same atomic budget. Already admitted requests may finish;
  success resets the shared per-user entry. This is a fixed window starting with
  the first attempt, not a sliding lockout starting with the final failure.
  Cache state is process-local, is lost on restart and is not shared between
  servers. Known usernames can be deliberately locked by callers with a valid
  client secret. AD authentication and DC lockout policies are outside this
  role's local-user backend.
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

- [FreeRADIUS configuration](https://wiki.freeradius.org/config/Configuration-files)
- [Native configuration check](https://www.freeradius.org/radiusd/man/radiusd.html)
- [RADIUS protocol testing](https://www.freeradius.org/radiusd/man/radclient.html)
- [PAP password attributes](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/mods-available/pap)
- [Cache configuration](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/mods-available/cache)
- [Structured logging](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/mods-available/linelog)
- [Method and password compatibility](https://www.freeradius.org/documentation/freeradius-server/3.2.9/concepts/protocol/authproto.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
