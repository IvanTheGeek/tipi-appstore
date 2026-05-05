# RunTIPI Notes

Hard-won knowledge from actual setup and troubleshooting. Any AI working on tipi-appstore or the VPS should read this first.

---

## Installation

RunTIPI installs via `runtipi-cli` which must be run as root (or via sudo). It installs itself relative to the working directory. On this VPS it is at `/srv/runtipi/` (moved from the initial install location of `/home/debian/runtipi/`).

```bash
cd /srv/runtipi && sudo ./runtipi-cli start
sudo ./runtipi-cli stop
sudo ./runtipi-cli restart
```

`/home/ivan/runtipi` is a symlink → `/srv/runtipi`.

---

## Key Paths

| Path | Purpose |
|---|---|
| `/srv/runtipi/` | RunTIPI root |
| `/srv/runtipi/.env` | Generated env — do not hand-edit; edit `state/settings.json` and restart |
| `/srv/runtipi/state/settings.json` | Persistent config — domain, appDataPath, etc. |
| `/srv/runtipi/app-data/<appstore-id>/<app-id>/` | App data (bind-mounts into containers) |
| `/srv/runtipi/apps/<appstore-id>/<app-id>/docker-compose.generated.yml` | Generated compose — RunTIPI writes this; see note below |
| `/srv/runtipi/traefik/shared/acme.json` | Let's Encrypt cert storage |
| `/srv/runtipi/traefik/dynamic/dynamic.yml` | Traefik dynamic config (default cert, TLS settings) |
| `/srv/runtipi/traefik/traefik.yml` | Traefik static config |
| `/srv/runtipi/logs/app.log` | RunTIPI app log |

---

## App Configuration Files

Each app in `apps/<appstore-id>/` has:
- `config.json` — metadata, port, categories
- `docker-compose.json` — RunTIPI dynamic compose schema (schemaVersion 2)
- `metadata/description.md` — shown in UI

### config.json critical fields

```json
{
  "tipi_version": 2,       // MUST be 2 for dynamic_config + extraLabels support
  "dynamic_config": true,  // enables docker-compose.json-based config
  "exposable": false,       // prevents RunTIPI from exposing the app via its own routing
  "port": 8380              // RunTIPI will bind host:8380 -> container:<internalPort>
}
```

### docker-compose.json critical fields

```json
{
  "schemaVersion": 2,
  "services": [{
    "internalPort": 80,   // container's listen port
    "extraLabels": {
      "traefik.enable": "true",    // REQUIRED — Traefik uses exposedByDefault=false
      // ... other Traefik labels
    }
  }]
}
```

**`traefik.enable: "true"` is mandatory.** RunTIPI's Traefik config sets `exposedByDefault: false`. Without this label, Traefik ignores the container entirely — no routing, no cert.

**`traefik.docker.network: runtipi_tipi_main_network` is required for apps with their own Docker network.** RunTIPI creates a per-app private network in addition to attaching the container to `runtipi_tipi_main_network`. When a container has multiple networks, Traefik may pick the wrong one (the private network that Traefik can't reach), causing 504 errors. This label forces Traefik to use the shared network. Any app whose `docker-compose.json` defines a second service (e.g. a database) will have its own network — always add this label.

---

## App Data Path

RunTIPI scopes `${APP_DATA_DIR}` to `<runtipiRoot>/app-data/<appstore-id>/<app-id>/`. For apps from the custom `tipi-appstore`, the path is:

```
/srv/runtipi/app-data/tipi-appstore/<app-id>/
```

**If an app bind-mounts a file (not a directory), the directory must exist on the host before the container starts.** If it doesn't, Docker creates the mount target as a directory, which then causes a "is a directory" error when trying to bind-mount a file into it. Fix: `sudo rm -rf <path>` then write the file.

---

## Generated Compose File

`docker-compose.generated.yml` is written by RunTIPI at install time and NOT regenerated on every start — only on install or app update. If you push a label change to the appstore, you must either:
1. Uninstall + reinstall the app (cleanest)
2. Or manually patch `docker-compose.generated.yml` and recreate the container:

```bash
docker compose \
  --env-file /srv/runtipi/app-data/<appstore-id>/<app-id>/app.env \
  --project-name <app-id>_<appstore-id> \
  -f /srv/runtipi/apps/<appstore-id>/<app-id>/docker-compose.generated.yml \
  up --detach --force-recreate
```

---

## Traefik

RunTIPI uses Traefik v3 as its reverse proxy (`runtipi-reverse-proxy` container).

- Port 80 and 443 are bound to the Traefik container, not to app containers directly
- App containers communicate with Traefik via the `runtipi_tipi_main_network` Docker network
- Traefik API is on port 8080 inside the container: `docker exec runtipi-reverse-proxy wget -qO- http://localhost:8080/api/rawdata`
- `exposedByDefault: false` — every app must have `traefik.enable: "true"` label
- ACME cert resolver is `myresolver`, uses HTTP challenge on port 80, stores in `/shared/acme.json`
- Traefik uses a RunTIPI-generated self-signed cert as fallback (`traefik/tls/cert.pem`); its SAN includes the configured domain (e.g. `runtipi.ivanthegeek.com`)
- Let's Encrypt certs are issued when Traefik first routes a TLS connection for a new domain — BUT only if no "provided certificate" already covers that domain
- **ACME blocked by defaultCertificate:** RunTIPI's self-signed cert includes the dashboard domain in its SAN. Traefik checks its ACME store first — if a LE cert exists there for the domain, it wins. But if the ACME store is empty (e.g. fresh setup or cleared acme.json), Traefik finds the static cert SAN match and skips ACME entirely. Fix: temporarily remove the `tls.stores.default.defaultCertificate` AND `tls.certificates` sections from `dynamic/dynamic.yml`, then trigger an HTTPS request to the domain. Traefik will issue the ACME cert and store it; after that the defaultCertificate can be restored (ACME store is checked first on subsequent restarts)

### Debugging Traefik routing

```bash
# List all routers (check status: enabled/disabled, rule)
docker exec runtipi-reverse-proxy wget -qO- http://localhost:8080/api/rawdata | python3 -c "
import json,sys
d=json.load(sys.stdin)
for name,r in d.get('routers',{}).items():
    print(name, '->', r.get('rule'), '->', r.get('status'))
"

# Check issued certs
cat /srv/runtipi/traefik/shared/acme.json | python3 -c "
import json,sys
d=json.load(sys.stdin)
for resolver,data in d.items():
    for c in (data.get('Certificates') or []):
        print(c.get('domain',{}).get('main'))
"
```

---

## Domain / Admin Panel

RunTIPI's admin panel domain is controlled by `domain` in `state/settings.json`. After editing, restart RunTIPI to regenerate `.env` and docker-compose.

- Admin panel: `https://runtipi.ivanthegeek.com`
- The dashboard Traefik router uses `Host(\`<domain>\`) && PathPrefix(\`/\`)`
- If `domain` is wrong (e.g. `example.com`), Traefik logs repeated ACME errors — fix by updating settings.json and restarting

---

## Custom Appstore

URL: `https://github.com/IvanTheGeek/tipi-appstore`  
Name in RunTIPI UI: `tipi-appstore`

Apps from this store are accessible in App Store under their appstore namespace. In the Traefik router names they appear as `<app-id>-insecure@docker` and `<app-id>@docker`.

After pushing changes to the appstore, go to Settings → Actions → Update App Stores to pull the latest. This updates `repos/<appstore-id>/` but does NOT regenerate `docker-compose.generated.yml` for existing apps.

---

## Notes on Moving RunTIPI

RunTIPI was initially installed under `/home/debian/runtipi/`. Moving it to `/srv/runtipi/`:
1. Stop all RunTIPI containers (including any managed apps)
2. `sudo mv /home/debian/runtipi /srv/runtipi`
3. `sudo sed -i 's|/home/debian/runtipi|/srv/runtipi|g' /srv/runtipi/.env`
4. Update `appDataPath` in `state/settings.json` to `/srv/runtipi`
5. Restart: `cd /srv/runtipi && sudo ./runtipi-cli start`
