## Coolify Deployment

This stack is designed to run behind Coolify's built-in Traefik proxy. Coolify auto-generates Traefik labels from domain assignments in its UI -- the compose file does **not** contain any Traefik labels.

### Compose File

Use `docker-compose.yaml` in this repo. It includes OpenCloud, Collabora, WOPI, Tika, ClamAV, Radicale, and external IdP (Authentik LDAP outpost) in a single file.

Do **not** include any `traefik/*` or `external-proxy/*` overlays when using Coolify.

### Domains and Ports

In Coolify's standard (non-raw) deployment mode, you assign domains **per service** in the UI. Coolify generates all Traefik routing labels automatically. The port suffix tells Coolify which container port to route traffic to -- it is **not** exposed publicly.

Assign these domains in the Coolify UI for the opencloud app:

| Service         | Domain in Coolify UI                        | Container port |
| --------------- | ------------------------------------------- | -------------- |
| opencloud       | `https://opencloud.example.com:9200`        | 9200           |
| collaboration   | `https://wopi.example.com:9300`             | 9300           |
| collabora       | `https://collabora.example.com:9980`        | 9980           |

Replace `example.com` with your real domain (e.g. `zacariahheim.com`).

Services that do **not** need a public domain (tika, clamav, radicale, authentik-ldap) should be left without a domain in the UI. They stay private on the internal Docker network.

Optional metrics ports (`9205` for opencloud, `9304` for collaboration) are exposed internally but not routed by Traefik.

### Networking

All services are attached to the external `coolify` network. Service-to-service URLs use Docker DNS (`opencloud`, `collaboration`, `collabora`, `tika`, `authentik-ldap`) to avoid public DNS when possible.

### Required Environment

Set these in Coolify env vars. **OC_DOMAIN is required** so the backend serves the correct server URL (config.json / theme.json) and CSP allows your origin:

```
OC_DOMAIN=opencloud.example.com
COLLABORA_DOMAIN=collabora.example.com
WOPISERVER_DOMAIN=wopi.example.com
INSECURE=false
```

Use your real domain (e.g. `opencloud.zacariahheim.com`), not the default `cloud.opencloud.test`. If OC_DOMAIN is missing or wrong you get "Missing or invalid config" and CSP blocking theme.json.

For a **single-domain** setup (one host for OpenCloud only), set `COMPANION_DOMAIN` to the same value as `OC_DOMAIN` so CSP allows theme/config requests.

### Storage

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
IDP_DOMAIN=auth.example.com
IDP_ISSUER_URL=https://auth.example.com/application/o/opencloud/
IDP_ACCOUNT_URL=https://auth.example.com/user/
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
AUTHENTIK_HOST=https://auth.example.com
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

### Troubleshooting: "Missing or invalid config" and CSP blocking theme.json

If the Web UI shows "Missing or invalid config" and the browser console reports CSP blocking `https://cloud.opencloud.test/themes/opencloud/theme.json`:

1. **Set OC_DOMAIN in Coolify** to the domain you use to access OpenCloud (e.g. `opencloud.zacariahheim.com`), with no `https://` or port. The proxy uses it to build `OC_URL` and to fill CSP rules; if it's missing, the app keeps using `cloud.opencloud.test`.
2. **Redeploy** after changing env so the opencloud container picks up the new `OC_DOMAIN`.
3. If the problem persists, ensure no persisted config overrides the URL: in the app's `storage/config` on the server, check for `proxy.yaml` or other oCIS config that might hardcode `cloud.opencloud.test`. oCIS normally prefers environment variables over config files; removing or fixing such a file and restarting can help.

### Collabora: "WopiDiscovery: wopi app url failed" / 404 on /hosting/discovery

The **collaboration** (WOPI) service calls `https://${COLLABORA_DOMAIN}/hosting/discovery` to find Collabora CODE. If that URL returns 404:

1. **Assign the Collabora domain to the correct service in Coolify**
   In the Coolify UI, set the `collabora` service's domain to `https://collabora.example.com:9980`. The `:9980` tells Coolify to route traffic to the Collabora CODE container (port 9980), not to the collaboration/WOPI container (port 9300). If the domain is assigned to the wrong service, discovery returns 404.

2. **Verify discovery locally**
   From the server: `curl -k https://collabora.example.com/hosting/discovery` should return XML. If it 404s, the domain is hitting the wrong backend -- check the Coolify domain assignment for the collabora service.
