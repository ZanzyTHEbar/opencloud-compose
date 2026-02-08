## Coolify Deployment

This stack is designed to run behind Coolify's built-in Traefik proxy. Use the Coolify overlay to expose only the internal service ports and let Coolify generate Traefik labels and routes.

### Compose File Order

Put the Coolify overlay **last** so it can override network settings and expose ports cleanly.

Example `COMPOSE_FILE`:

```
COMPOSE_FILE=docker-compose.yml:weboffice/collabora.yml:search/tika.yml:antivirus/clamav.yml:radicale/radicale.yml:monitoring/monitoring.yml:monitoring/monitoring-collaboration.yml:docker-compose.coolify.yml
```

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
