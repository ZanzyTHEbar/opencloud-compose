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

Include `idm/external-idp.yml` and set these variables to your Authentik values:

```
IDP_DOMAIN=auth.zacariahheim.com
IDP_ISSUER_URL=https://auth.zacariahheim.com/application/o/opencloud/
IDP_ACCOUNT_URL=https://auth.zacariahheim.com/user/
OC_OIDC_CLIENT_ID=<from Authentik>
```

OpenCloud still uses the LDAP service for user storage in external IdP mode.

### Optional Compose Env Vars (Not Used by Coolify)

Coolify doesn't use `COMPOSE_FILE` or `COMPOSE_PATH_SEPARATOR`, so you can omit those for deployments in Coolify.
