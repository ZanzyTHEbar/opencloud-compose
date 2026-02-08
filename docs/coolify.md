## Coolify Deployment

This stack is designed to run behind Coolify's built-in Traefik proxy. The Compose file includes explicit Traefik labels for OpenCloud, Collabora, and WOPI.

### Compose File

Use `docker-compose.yaml` in this repo. It already includes OpenCloud, Collabora, WOPI, Tika, ClamAV, Radicale, monitoring, external IdP, and Traefik labels in a single file.

Do **not** include any `traefik/*` or `external-proxy/*` overlays when using Coolify.

### Domains and Ports

Traefik labels are defined in `docker-compose.yaml` for:

- `opencloud` -> `https://opencloud.zacariahheim.com` (port `9200`)
- `collaboration` -> `https://wopi.zacariahheim.com` (port `9300`)
- `collabora` -> `https://collabora.zacariahheim.com` (port `9980`)

If you use different domains, update the label host rules in `docker-compose.yaml`.

Optional metrics are still available on ports `9205` and `9304` if you want to expose them separately.

### Networking

All services are attached to the external `coolify` network. Service-to-service URLs use Docker DNS
(`opencloud`, `collaboration`, `collabora`, `tika`, `ldap-server`) to avoid public DNS when possible.

### Required Environment

Set these in Coolify env vars:

```
OC_DOMAIN=opencloud.zacariahheim.com
COLLABORA_DOMAIN=collabora.zacariahheim.com
WOPISERVER_DOMAIN=wopi.zacariahheim.com
INSECURE=false
```

Storage paths (bind mounts inside the Coolify LXC):

This Compose file mounts from `./storage/...`, so create a symlink inside the Coolify app directory:

```
ln -s /mnt/opencloud /data/coolify/applications/<app_uuid>/storage
```

That maps:

```
./storage/config    -> /mnt/opencloud/config
./storage/data      -> /mnt/opencloud/data
./storage/apps      -> /mnt/opencloud/apps
./storage/services  -> /mnt/opencloud/services
```

If antivirus is enabled, add it to `START_ADDITIONAL_SERVICES`:

```
START_ADDITIONAL_SERVICES=antivirus
```

### Authentik OIDC (External IdP)

Set these variables to your Authentik values:

```
IDP_DOMAIN=auth.zacariahheim.com
IDP_ISSUER_URL=https://auth.zacariahheim.com/application/o/opencloud/
IDP_ACCOUNT_URL=https://auth.zacariahheim.com/user/
OC_OIDC_CLIENT_ID=<from Authentik>
```

OpenCloud still uses LDAP for user storage in external IdP mode. We use the Authentik LDAP outpost for this.

### Authentik LDAP Outpost (Required)

Create an LDAP provider and outpost in Authentik:

1. Create an **LDAP provider + application**.
2. Use the default **Base DN** `DC=ldap,DC=goauthentik,DC=io` (or set your own).
3. Create a service account user (e.g. `ldapservice`), set a password, and grant **Search full LDAP directory** permission.
4. Create an **LDAP outpost** for the provider and copy the outpost token.

Authentik LDAP provider defaults:
- Users are under `ou=users,<base DN>` and groups under `ou=groups,<base DN>`.
- The provider exposes `cn` (username), `uid` (unique identifier), `displayName`, `mail`, and `memberOf`.
- The LDAP provider requires an LDAP outpost.
- Manual outpost deployment uses `ghcr.io/goauthentik/ldap` and the `AUTHENTIK_HOST` + `AUTHENTIK_TOKEN` env vars.
- The default service account DN is `cn=ldapservice,ou=users,DC=ldap,DC=goauthentik,DC=io`.

OpenCloud uses LDAP even with OIDC. When `PROXY_AUTOPROVISION_ACCOUNTS=false`, users must already exist in LDAP; when `true`, a writable LDAP is required.

Set these Coolify env vars:

```
# Outpost container -> Authentik
AUTHENTIK_HOST=https://auth.zacariahheim.com
AUTHENTIK_INSECURE=false
AUTHENTIK_LDAP_TOKEN=<outpost-token>

# OpenCloud -> LDAP outpost
AUTHENTIK_LDAP_URI=ldap://authentik-ldap:3389
AUTHENTIK_LDAP_INSECURE=true
AUTHENTIK_LDAP_BIND_DN=cn=ldapservice,ou=users,dc=ldap,dc=goauthentik,dc=io
AUTHENTIK_LDAP_BIND_PASSWORD=<service-account-password>
AUTHENTIK_LDAP_USER_BASE_DN=ou=users,dc=ldap,dc=goauthentik,dc=io
AUTHENTIK_LDAP_GROUP_BASE_DN=ou=groups,dc=ldap,dc=goauthentik,dc=io
AUTHENTIK_LDAP_USER_SCHEMA_ID=uid
AUTHENTIK_LDAP_GROUP_SCHEMA_ID=uid
AUTHENTIK_LDAP_USER_FILTER=(objectClass=user)
```

If you enable LDAPS, switch `AUTHENTIK_LDAP_URI` to `ldaps://authentik-ldap:6636` and set
`AUTHENTIK_LDAP_INSECURE=false` once the outpost has a trusted certificate.

### Optional Compose Env Vars (Not Used by Coolify)

Coolify doesn't use `COMPOSE_FILE` or `COMPOSE_PATH_SEPARATOR`, so you can omit those for deployments in Coolify.
