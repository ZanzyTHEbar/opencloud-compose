## Coolify Deployment

This stack is designed to run behind Coolify's built-in Traefik proxy. Coolify will run `docker compose` with a single file and generate Traefik labels for any domains you assign.

### Compose File

Use `docker-compose.yaml` in this repo. It already includes OpenCloud, Collabora, WOPI, Tika, ClamAV, Radicale, monitoring, external IdP, and the Coolify port exposures in a single file.

Do **not** include any `traefik/*` or `external-proxy/*` overlays when using Coolify.

### Domains and Ports (Coolify)

Assign domains in Coolify so it can generate Traefik labels automatically.

- `opencloud` service on port `9200` -> `opencloud.zacariahheim.com`
- `collaboration` service on port `9300` -> `wopi.zacariahheim.com`
- `collabora` service on port `9980` -> `collabora.zacariahheim.com`
- Optional metrics: `opencloud` service on port `9205` -> metrics domain (if desired)
- Optional collaboration metrics: `collaboration` service on port `9304` -> metrics domain (if desired)

### Required Environment

Set these in Coolify env vars:

```
OC_DOMAIN=opencloud.zacariahheim.com
COLLABORA_DOMAIN=collabora.zacariahheim.com
WOPISERVER_DOMAIN=wopi.zacariahheim.com
INSECURE=false
```

Storage paths (bind mounts inside the Coolify LXC):

```
OC_CONFIG_DIR=/mnt/opencloud/config
OC_DATA_DIR=/mnt/opencloud/data
OC_APPS_DIR=/mnt/opencloud/apps
RADICALE_DATA_DIR=/mnt/opencloud/services/radicale
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
