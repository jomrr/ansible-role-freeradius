# Ansible Role: freeradius

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-freeradius)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-freeradius)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-freeradius)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-freeradius/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-freeradius/actions/workflows/main.yml?query=branch%3Amain)

Configure FreeRADIUS with PAP VPN profiles, MAC admission, EAP-TLS VLAN policies
and buffered accounting.

## Purpose

Run FreeRADIUS for local PAP with optional VPN profiles, MAC admission with
VLANs and quarantine, EAP-TLS certificate policies, and accounting over IPv4
UDP.

## Scope

### Managed

- Distribution packages, the complete server configuration, local users and the
  enabled, running service.
- Local PAP authentication with Crypt-Password, PBKDF2-Password,
  SSHA2-512-Password, NT-Password or Cleartext-Password credentials.
- Per-user PAP attempt limits and Accept/Reject logging.
- Optional client-specific PAP VPN profiles with MikroTik PPP groups, IP pools,
  filters and session limits.
- Optional MAC admission for UniFi and MikroTik, with device-specific VLANs and
  an explicit quarantine profile.
- Optional EAP-TLS with CA, issuer, clientAuth EKU and certificate policy
  checks; CRL or OCSP revocation and dynamic VLAN replies.
- Optional accounting to local detail files or asynchronously to PostgreSQL,
  including PostgreSQL connection and TLS settings.
- Optional PostgreSQL schema initialization and login limits based on open SQL
  accounting sessions.

### Not Managed

- CHAP, MSCHAPv2 and other EAP methods; LDAP, AD and SQL authentication
  backends.
- RADIUS over TLS, RADIUS proxying, switch/AP configuration and WPA3-Enterprise
  192-bit profile enforcement.
- PostgreSQL server, database and login creation; existing-schema migrations and
  accounting retention or backups.
- Firewall rules, network tunnels, NAS IP pools and PPP profiles, and
  certificate provisioning.
- RouterOS administrative logins using MS-CHAPv2, UniFi VPN integration, and
  CoA/Disconnect.

## Requirements

- RADIUS clients must send Message-Authenticator on every Access-Request.
- VPN authorization expects PAP with Service-Type Framed-User and
  Framed-Protocol PPP, such as MikroTik OpenVPN with user-auth-method=pap.
  Referenced PPP profiles, IP pools and filters must already exist on the
  gateway.
- MAB requires a configured client service discriminator, Ethernet or
  Wireless-802.11 NAS-Port-Type, and the MAC address in User-Name, User-Password
  and Calling-Station-Id. Set RouterOS
  mac-auth-mode=mac-as-username-and-password. Enable RADIUS-assigned VLANs in
  UniFi; provision VLANs and their isolation rules on both platforms.
- EAP-TLS requires FreeRADIUS 3.2+, a PEM server certificate chain and key, and
  a client CA bundle readable by the service account. The switch or AP must
  support standard RADIUS tunnel attributes for dynamic VLAN assignment.
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

Authoritative local PAP users; an empty list disables user authentication
without affecting EAP-TLS or MAB.

Default:

```yaml
freeradius_users: []
```

### `freeradius_authorization_enabled`

Type: `bool`. Required: `false`.

Require a VPN profile grant after PAP authentication; false preserves
credential-only PAP.

Default:

```yaml
freeradius_authorization_enabled: false
```

### `freeradius_access_profiles`

Type: `list`. Required: `false`.

Named VPN and MAB reply profiles; values refer to resources already provisioned
on the NAS.

Default:

```yaml
freeradius_access_profiles: []
```

### `freeradius_mab_enabled`

Type: `bool`. Required: `false`.

Enable MAC admission on clients with the mab service; requires profile
authorization enabled.

Default:

```yaml
freeradius_mab_enabled: false
```

### `freeradius_mab_devices`

Type: `list`. Required: `false`.

MAC devices and client-specific grants; absent grants select quarantine after
valid MAC credentials.

Default:

```yaml
freeradius_mab_devices: []
```

### `freeradius_mab_quarantine_profile`

Type: `str`. Required: `false`.

Required when MAB is enabled; name of a mab profile with the quarantine VLAN.

### `freeradius_failure_limit`

Type: `int`. Required: `false`.

Maximum admitted PAP attempts per local user in a fixed window, from 1 to 1000;
success resets the counter.

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

Open SQL sessions allowed per local PAP user; zero disables checking, positive
values require postgresql.

Default:

```yaml
freeradius_simultaneous_use: 0
```

### `freeradius_eap_tls_enabled`

Type: `bool`. Required: `false`.

Enable certificate-only EAP-TLS with TLS 1.2 or 1.3 alongside local PAP;
requires FreeRADIUS 3.2 or newer.

Default:

```yaml
freeradius_eap_tls_enabled: false
```

### `freeradius_eap_tls_certificate_file`

Type: `path`. Required: `false`.

Required for EAP-TLS; absolute PEM server certificate chain path on the RADIUS
host.

### `freeradius_eap_tls_private_key_file`

Type: `path`. Required: `false`.

Required for EAP-TLS; absolute PEM server private key path on the RADIUS host,
readable by the service account.

### `freeradius_eap_tls_private_key_password`

Type: `str`. Required: `false`.

Private key password; empty for an unencrypted key. Letters, digits and
._~!@#^&*+=:/?- are accepted.

Default:

```yaml
freeradius_eap_tls_private_key_password: ''
```

### `freeradius_eap_tls_ca_file`

Type: `path`. Required: `false`.

Required for EAP-TLS; absolute PEM client CA bundle path on the RADIUS host.

### `freeradius_eap_tls_issuer`

Type: `str`. Required: `false`.

Required for EAP-TLS; exact client issuer DN in OpenSSL compat format, such as
/O=Example/CN=Device CA.

### `freeradius_eap_tls_revocation`

Type: `str`. Required: `false`.

Certificate revocation policy; crl and ocsp reject unavailable or invalid
revocation information.

Default:

```yaml
freeradius_eap_tls_revocation: crl
```

### `freeradius_eap_tls_ca_path`

Type: `path`. Required: `false`.

Required for crl; absolute OpenSSL-rehashed CA and CRL directory, refreshed by
FreeRADIUS every 300 seconds.

Default:

```yaml
freeradius_eap_tls_ca_path: ''
```

### `freeradius_eap_tls_ocsp_url`

Type: `str`. Required: `false`.

Required for ocsp; explicit HTTP responder URL. Signed responses and nonces are
required; timeout is five seconds.

Default:

```yaml
freeradius_eap_tls_ocsp_url: ''
```

### `freeradius_eap_tls_vlan_policies`

Type: `list`. Required: `false`.

Required nonempty for EAP-TLS; exactly one policy must match. Certificate
policies with qualifiers are rejected.

Default:

```yaml
freeradius_eap_tls_vlan_policies: []
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

- MAB identifies devices by spoofable MAC addresses. VLAN isolation and the
  quarantine firewall policy must be enforced by the NAS/network.
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
- EAP-TLS clients must validate the server CA and expected server name. Restrict
  private key permissions to the service account. Selecting revocation mode none
  permits revoked certificates until expiry.

## Operational Notes

- Molecule checks RADIUS responses on the supported server platforms. Actual
  VLAN, PPP profile, pool and filter enforcement requires a separate hardware
  check on the deployed UniFi/MikroTik devices; it is not covered by these
  tests.
- freeradius_authorization_enabled requires a matching user access grant and a
  client with services: [vpn]. Profiles are applied only after successful
  authentication and session checks. Disabled authorization preserves
  credential-only PAP.
- MAB additionally requires freeradius_mab_enabled and the client service mab.
  Select mab_service_type to match the NAS: UniFi typically sends Call-Check,
  RouterOS dot1x can send Framed-User. Confirm the attributes on the deployed
  firmware.
- MAC case and plain, colon, hyphen or dotted notation are normalized. Valid
  matching MAC credentials without a device grant receive the quarantine
  profile; malformed or inconsistent credentials are rejected. No user login
  falls back to quarantine. Empty device lists quarantine all valid MAB
  requests. MAC admission bypasses PAP attempt and SQL session limits.
- The role replaces radiusd.conf and the local users file. Client and user lists
  are authoritative. An empty freeradius_users list disables local user
  authentication; enabled accounting accepts records from configured clients
  independently of the local user list.
- User passwords must match freeradius_password_attribute, with an optional
  per-user password_attribute override. The default Crypt-Password accepts
  SHA-512 crypt or yescrypt hashes; yescrypt requires support in the target
  system's libcrypt.
- Reaching freeradius_failure_limit blocks further PAP requests, including valid
  passwords, until freeradius_failure_window expires. Success before the limit
  resets the counter. Counters reset on restart and are not shared between
  servers.
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
- freeradius_simultaneous_use applies to local PAP users and requires
  PostgreSQL; zero disables the limit. A positive limit rejects logins when the
  recorded open-session count reaches the limit or SQL lookup fails. Accounting
  delays and parallel logins prevent a strict concurrent-session guarantee.
- EAP-TLS authenticates independently of freeradius_users. Exactly one
  configured policy OID must match; missing or ambiguous matches and policy
  qualifiers are rejected. TLS 1.2 and 1.3 are enabled; session resumption is
  disabled.
- CRL mode requires an OpenSSL-rehashed directory with current CA certificates
  and CRLs for the full chain; it reloads every 300 seconds. OCSP mode requires
  the configured responder to support nonces. Both modes reject failed
  revocation checks. Certificate provisioning and renewal are external; restart
  FreeRADIUS after replacing its certificate or key.

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

### MikroTik PAP VPN authorization

Use OpenVPN user-auth-method=pap and enable PPP RADIUS authentication.
Create vpn-staff, vpn-pool and vpn-filter on the gateway.

```yaml
- name: Configure VPN admission
  hosts: radius_servers
  gather_facts: true
  roles:
    - role: jomrr.freeradius
      freeradius_listen_address: 192.0.2.10
      freeradius_authorization_enabled: true
      freeradius_clients:
        - name: mikrotik_vpn
          address: 192.0.2.20
          secret: "{{ vault_vpn_radius_secret }}"
          services: [vpn]
      freeradius_access_profiles:
        - name: vpn_staff
          kind: vpn
          mikrotik_group: vpn-staff
          framed_pool: vpn-pool
          filter_id: vpn-filter
          session_timeout: 3600
          idle_timeout: 600
      freeradius_users:
        - name: alice
          password: "{{ vault_radius_alice_password_hash }}"
          access:
            - {client: mikrotik_vpn, profile: vpn_staff}
```

### UniFi and MikroTik MAC admission with quarantine

Example VLANs 300 and 999 need printer and quarantine firewall policies.
Verify NAS request attributes before rollout.

```yaml
- name: Configure device admission
  hosts: radius_servers
  gather_facts: true
  roles:
    - role: jomrr.freeradius
      freeradius_listen_address: 192.0.2.10
      freeradius_authorization_enabled: true
      freeradius_mab_enabled: true
      freeradius_mab_quarantine_profile: quarantine
      freeradius_clients:
        - name: unifi_switch
          address: 192.0.2.30
          secret: "{{ vault_unifi_radius_secret }}"
          services: [mab]
          mab_service_type: Call-Check
        - name: mikrotik_switch
          address: 192.0.2.40
          secret: "{{ vault_mikrotik_radius_secret }}"
          services: [mab]
          mab_service_type: Framed-User
      freeradius_access_profiles:
        - {name: printers, kind: mab, vlan_id: 300}
        - {name: quarantine, kind: mab, vlan_id: 999, session_timeout: 300}
      freeradius_mab_devices:
        - mac: '00:11:22:33:44:55'
          access:
            - {client: unifi_switch, profile: printers}
            - {client: mikrotik_switch, profile: printers}
```

### EAP-TLS with certificate policy VLANs

Provision PKI files first; use policy OIDs under the assigned PEN.

```yaml
- name: Configure certificate authentication
  hosts: radius_servers
  gather_facts: true
  roles:
    - role: jomrr.freeradius
      freeradius_listen_address: 192.0.2.10
      freeradius_clients:
        - name: access_switch
          address: 192.0.2.20
          secret: "{{ vault_radius_client_secret }}"
      freeradius_eap_tls_enabled: true
      freeradius_eap_tls_certificate_file: /etc/radius-pki/server.pem
      freeradius_eap_tls_private_key_file: /etc/radius-pki/server.key
      freeradius_eap_tls_ca_file: /etc/radius-pki/ca.pem
      freeradius_eap_tls_issuer: /O=Example/CN=Device CA
      freeradius_eap_tls_ca_path: /etc/radius-pki/trust
      freeradius_eap_tls_vlan_policies:
        - {oid: '1.3.6.1.4.1.32473.1', vlan_id: 100}
        - {oid: '1.3.6.1.4.1.32473.2', vlan_id: 200}
```

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

- [MikroTik RADIUS attributes](https://manual.mikrotik.com/docs/authentication-authorization-accounting/radius/)
- [MikroTik OpenVPN](https://help.mikrotik.com/docs/spaces/ROS/pages/2031655/OpenVPN)
- [UniFi MAC VLAN assignment](https://help.ui.com/hc/en-us/articles/115004589707-MAC-Based-VLAN-Assignment-Using-802-1x-in-UniFi-Network)
- [EAP-TLS configuration](https://github.com/FreeRADIUS/freeradius-server/blob/v3.2.x/raddb/mods-available/eap)
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
