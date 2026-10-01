# Molecule scenarios

Run a scenario from the role repository with `molecule test -s <name>`.

- `default`: PAP hashes, VPN/MAB profiles, quarantine, EAP-TLS/CRL and buffered
  SQL together on one platform.
- `dev`: The same fixed configuration on all six supported platforms.
- `lockout`: PAP counter reset, concurrent failures, account isolation and
  expiry.
- `sql_outage`: Accounting buffering/replay and continued PAP admission without
  a session limit.
- `session_limits`: SQL outage rejection, MAB bypass, active/stale sessions,
  interim updates and Stop.
- `authorization`: Profile changes and enrollment of a previously quarantined
  device.
- `quarantine`: MAB quarantine with no configured users or devices.
- `eap_ocsp`: OCSP acceptance, revocation and responder outage without PAP
  users.
- `eap_no_revocation`: Explicitly disabled certificate revocation checks.
- `pap_detail`: Switch to plain PAP/detail, changed credentials/ports, disabled
  EAP and failed detail writes.
- `empty_access`: Removal of all clients, users and devices.
- `validation`: Invalid networks, repeated secrets, invalid profiles and
  conflicting grants.

`default` and `dev` converge one configuration, check idempotence and check mode,
then verify it once. They do not run configuration transitions or a restore cycle.
The three configuration-update scenarios use one `side_effect` play each; outage
and lockout scenarios change only the state needed for their specific checks.

The remaining scenarios use one platform, selected through `uns`, `img` and `tag`
as in `default`. PostgreSQL runs in a separate container only where required.
Each radclient command sends one request. Playbooks use ordinary module loops,
with no imported task files or generated packet batches.

The tests assert RADIUS decisions and response attributes. VLAN, PPP profile,
pool and filter enforcement still needs verification on the actual NAS hardware.
