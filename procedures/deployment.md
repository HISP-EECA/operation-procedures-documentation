# Deployment guide

Step-by-step guide for standing up the Uzbekistan Civil Registry
integration, both locally (Podman or Docker) and in production
(LXC / LXD containers following
[`dhis2/dhis2-server-tools/deploy/roles/integration`](https://github.com/dhis2/dhis2-server-tools/tree/main/deploy/roles/integration)).

> For a general project overview see [README.en.md](https://github.com/HISP-EECA/uzbekistan-dhis2-civil-registry-integration/blob/main/README.en.md).

---

## Table of contents

1. [Architecture recap](#1-architecture-recap)
2. [Step 0 — Prerequisites](#step-0--prerequisites)
3. [Section A — Local replication (Podman or Docker)](#section-a--local-replication-podman-or-docker)
4. [Section B — Production deployment on LXC](#section-b--production-deployment-on-lxc)
5. [Section C — Redeployment / upgrades](#section-c--redeployment--upgrades)
6. [Section D — Troubleshooting](#section-d--troubleshooting)

---

## 1. Architecture recap

Two independently deployable components:

| Component | Artifact | Runs on |
|---|---|---|
| **Middleware** (Camel / Spring Boot) | Fat JAR | Its own LXC container (prod) / `middleware` image (local) |
| **Civil Registry plugin** | ZIP installed via DHIS2 App Management | Inside DHIS2 (all environments) |

The middleware queries the Uzbekistan civil registry (`ssv.uz`) using OAuth2
and returns a flat person object. DHIS2 proxies the plugin's request
through its `civil-registry` route to the middleware.

---

## Step 0 — Prerequisites

Same prerequisites for local and production build machines. On the
production LXC hosts you only need the runtime bits (Java 17); the
build itself happens on a workstation or CI runner.

### Common (build machine)

| Tool | Minimum version | Check with | Purpose |
|---|---|---|---|
| JDK | 17 (Temurin / OpenJDK) | `java --version` | Compile middleware, enforced by [pom.xml](middleware/pom.xml) |
| Maven | 3.9+ | `mvn --version` | Middleware build |
| Node.js | 20+ | `node --version` | Plugin build |
| Yarn | 1.22 (classic) | `yarn --version` | Plugin build scripts |
| Git | any | `git --version` | Clone / branch |

Yarn is not always packaged. Recommended install path via
[Corepack](https://nodejs.org/api/corepack.html) (bundled with Node ≥ 16):

```sh
corepack enable
corepack prepare yarn@1.22.22 --activate
yarn --version    # should print 1.22.22
```

### Local replication — choose Podman **or** Docker

Both are supported by [`docker-compose.yml`](docker-compose.yml). Only
install one.

#### Option A — Podman (rootless, no daemon)

| Tool | Minimum version | Check |
|---|---|---|
| Podman | 4.4+ | `podman --version` |
| podman-compose | 1.0.6+ | `podman-compose --version` |

Install (Fedora/RHEL):
```sh
sudo dnf install -y podman podman-compose
```

Debian/Ubuntu:
```sh
sudo apt install -y podman podman-compose
```

Or via `pipx` if the distro package is too old:
```sh
pipx install podman-compose
```

One-time Podman configuration for this project (required because the
compose file uses unqualified image names like `nginx`, `busybox`):

```sh
mkdir -p ~/.config/containers
cat > ~/.config/containers/registries.conf <<'EOF'
unqualified-search-registries = ["docker.io", "quay.io"]
short-name-mode = "permissive"
EOF

systemctl --user enable --now podman.socket
echo 'export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/podman/podman.sock"' >> ~/.bashrc
```

Reload the shell (`exec bash`) and verify:
```sh
podman info | grep -A3 registries
echo "$DOCKER_HOST"
```

#### Option B — Docker

| Tool | Minimum version | Check |
|---|---|---|
| Docker Engine | 24+ | `docker --version` |
| Docker Compose plugin | v2.20+ | `docker compose version` |

Install:
```sh
# Fedora/RHEL
sudo dnf install -y docker-ce docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"   # log out and back in

# Debian/Ubuntu
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Verify:
```sh
docker run --rm hello-world
```

### Substitution rule for the rest of this guide

The commands are written with `podman-compose`. If you chose Docker,
replace `podman-compose` with `docker compose` (space, no dash).
Everything else is identical.

### Network / port requirements (local)

Reserve these ports on your workstation:

| Port | Container | Purpose |
|---|---|---|
| **8080** | `gateway` (nginx) | DHIS2 web UI |
| **9070** | `middleware` | Camel REST endpoint (`/api/find`) |
| **9071** | `middleware` | Spring Boot management + Hawtio |
| **5432** | `db` | Postgres (**collision risk** if you already run Postgres locally) |

---

## Section A — Local replication (Podman or Docker)

### A.1 — Clone the repo and prepare `.env`

```sh
git clone https://github.com/HISP-EECA/uzbekistan-dhis2-civil-registry-integration.git
cd uzbekistan-dhis2-civil-registry-integration
cp .env.example .env
```

Edit `.env` and give **all** variables a non-blank value. For pure
local sanity checks the OAuth2 and civil-registry URLs don't need to
resolve — the middleware still boots; only actual lookups will fail
until you plug in real Uzbek SSV credentials.

Minimum non-blank content:
```dotenv
CIVIL_REGISTRY_CITIZEN_URL=https://test-api2.ssv.uz/v3/Citizen
CIVIL_REGISTRY_CITIZEN_ADDRESS_URL=https://test-api2.ssv.uz/v3/CitizenAddress
OAUTH2_TOKENENDPOINT=https://test-sso.ssv.uz/oauth/token
OAUTH2_CLIENTID=placeholder
OAUTH2_CLIENTSECRET=placeholder
```

### A.2 — Build the middleware JAR

Podman/Docker won't do this for you: the middleware Dockerfile `COPY`s
from `middleware/target/`, so Maven must produce the JAR first.

```sh
mvn -B clean package -DskipTests -f middleware/pom.xml
ls -lh middleware/target/civil-registry-middleware-1.0.0-SNAPSHOT.jar
```

Expected: a self-contained fat JAR (~60–80 MB).

### A.3 — Verify the middleware base image

The [middleware/Dockerfile](middleware/Dockerfile) uses a Java 17 base
image. Confirm it's still pullable:

```sh
grep '^FROM' middleware/Dockerfile
```

If it says `openjdk:17-slim` and pulling fails (Docker Hub retired
that tag in 2024), replace it with the officially recommended
successor:

```sh
sed -i 's|^FROM openjdk:17-slim|FROM docker.io/eclipse-temurin:17-jre-jammy|' middleware/Dockerfile
```

### A.4 — Build the plugin ZIP

```sh
cd plugin
yarn install --frozen-lockfile
yarn build --no-verify
cd ..
ls -lh plugin/build/bundle/
```

Expected: `civil-registry-plugin-1.0.0.zip` (~750 KB).

### A.5 — Bring the stack up

```sh
podman-compose up -d --build
podman-compose ps
```

Five containers should appear: `db`, `dhis2`, `gateway`, `middleware`,
`seed`. DHIS2 takes ~2 minutes on first boot to seed its schema. Watch
it:

```sh
podman-compose logs -f dhis2
# Ctrl-C once you see "Server startup" or the healthcheck flips green
```

Health check:
```sh
podman ps --format 'table {{.Names}}\t{{.Status}}'
```

### A.6 — Test each component

```sh
# DHIS2 (through the gateway on port 8080)
curl -sf http://localhost:8080/api/ping && echo OK

# Middleware Camel endpoint
curl -sS -X POST -H 'Content-Type: application/json' \
     -d '{"id":"12345"}' http://localhost:9070/api/find
# A 4xx/5xx is expected here — the SSV endpoints are not reachable.
# The goal is only to confirm the middleware is up and listening.

# Middleware management
curl -sf http://localhost:9071/management/health
```

### A.7 — Install the plugin into DHIS2

The `seed` container installs the route, `pluginConfig` and the
`personMap` datastore key, but **not** the plugin zip (that line is
commented out). Use the yarn script:

```sh
yarn deploy
# Wraps: cd plugin && yarn deploy http://localhost:8080
# You'll be prompted for username / password — use admin / district
```

Or, one-liner with `curl`:

```sh
curl -u admin:district \
     -F file=@plugin/build/bundle/civil-registry-plugin-1.0.0.zip \
     http://localhost:8080/api/apps
```


### A.8 — Verify in the browser

1. Open http://localhost:8080 → log in with `admin` / `district`.
2. Open the Capture app, pick a Tracker programme, start an
   enrollment.
3. The `Patient ID` lookup field renders.
4. Clicking **Search** will show *"Failed to query civil registry"*
   (the SSV API is not reachable from your laptop) — this proves the
   plugin loaded and the JSONata pass-through mapping works.

### A.9 — Tear down when you're done

```sh
podman-compose down             # stop containers, keep DB volume
podman-compose down --volumes   # erease DB volume (fresh start next time)
```

---

## Section B — Production deployment on LXC

Follows the pattern in
[`dhis2/dhis2-server-tools/deploy/roles/integration`](https://github.com/dhis2/dhis2-server-tools/tree/main/deploy/roles/integration).
This section documents what the Ansible role does so you can follow the
same layout manually or adapt the role the middleware.

### B.1 — Environments and hosts

Recommended topology (per the `dhis2-server-tools` `hosts` inventory):

| Group | Hosts | Purpose |
|---|---|---|
| `instances` | `dhis2-prod`, `dhis2-dev` | DHIS2 core LXC containers |
| `integration` | `civil-registry-prod`, `civil-registry-dev` | Middleware LXC containers |

Each `integration` host is a stock Ubuntu 22.04 LTS LXD container
created from the `ubuntu/22.04` image. The role brings it up with
`community.general.lxd_container` (see
[`tasks/lxd.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/lxd.yml)).

### B.2 — Directory / user layout expected on the middleware host

Every path, mode and owner in the table below comes from the
`dhis2-server-tools/integration` Ansible role. Each row links to the
exact source task that produces it can be verified or adapted.

| Path | Owner:Group | Mode | Purpose | Source |
|---|---|---|---|---|
| `/usr/local/bin/civil_registry/` | `civil_registry:civil_registry` | `0750` | Working directory | [`defaults/main.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/defaults/main.yml) (`working_directory`) + `file` task in [`tasks/lxd.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/lxd.yml) |
| `/usr/local/bin/civil_registry/civil_registry_proxy.jar` | `root:root` | `0640` (+ ACL `r` for `civil_registry`) | The fat JAR | `copy` + `acl` tasks in [`tasks/jar-file-deploy.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/jar-file-deploy.yml) |
| `/usr/local/etc/application.yml` | `root:root` | `0640` | Spring Boot config with secrets | `template` task in [`tasks/jar-file-deploy.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/jar-file-deploy.yml) rendering [`templates/application.yml.j2`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/templates/application.yml.j2) |
| `/dhis2Person.ds` | `root:root` | `0640` | Datasonnet mapping (referenced by `dataSonnet.dhis2Person` in `application.yml`) | `copy` task in [`tasks/jar-file-deploy.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/jar-file-deploy.yml) |
| `/etc/systemd/system/civil_registry.service` | `root:root` | `0640` | Systemd unit | `template` task in [`tasks/jar-file-deploy.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/jar-file-deploy.yml) rendering [`templates/systemd_service.j2`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/templates/systemd_service.j2) |

The service account `civil_registry` is a locked-down system user
(`shell=/bin/false`, no home), created by the `user`/`group` tasks in
[`tasks/lxd.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/lxd.yml).
Everything except the working directory is `root`-owned; the JAR is
readable by the service user through an ACL only.

> **This doc intentionally diverges from the role:**
>
> 1. The role's [`defaults/main.yml`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/defaults/main.yml)
>    sets `service_port: 8080`. Our middleware actually listens on
>    **9070** (see [`middleware/src/main/resources/application.yaml`](middleware/src/main/resources/application.yaml)).
>    Needs to be confirmed which port is really open in prod.
> 2. The role's [`templates/application.yml.j2`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/templates/application.yml.j2)
>    assumes a single `civil-registry-url`. The Uzbekistan variant needs two
>    (`civil-registry-citizen-url` + `civil-registry-citizen-address-url`).
>    §B.7 below reflects the Uzbekistan shape.

### B.3 — Build the JAR once, on your workstation or CI

```sh
mvn -B clean package -DskipTests -f middleware/pom.xml
```

### B.4 — Provision the LXC container (host side)

The role uses Ansible. Manually, an equivalent one-shot:

```sh
# on the LXD host
lxc launch images:ubuntu/22.04 civil-registry-prod
lxc exec civil-registry-prod -- apt update
lxc exec civil-registry-prod -- apt install -y openjdk-17-jre-headless acl ufw
```

### B.5 — Create the service account & directories (inside the container)

```sh
lxc exec civil-registry-prod -- bash -eux <<'EOF'
groupadd --system civil_registry
useradd  --system --gid civil_registry --shell /bin/false --no-create-home civil_registry
install -d -o civil_registry -g civil_registry -m 0750 /usr/local/bin/civil_registry
EOF
```

### B.6 — Copy the JAR, mapping and config

From your workstation:

```sh
JAR=middleware/target/civil-registry-middleware-1.0.0-SNAPSHOT.jar

lxc file push "$JAR" \
    civil-registry-prod/usr/local/bin/civil_registry/civil_registry_proxy.jar
lxc file push config/middleware/dhis2Person.ds civil-registry-prod/dhis2Person.ds
```

Inside the container, tighten permissions and grant the service user
read-only access to the JAR:

```sh
lxc exec civil-registry-prod -- bash -eux <<'EOF'
chown root:root /usr/local/bin/civil_registry/civil_registry_proxy.jar /dhis2Person.ds
chmod 0640      /usr/local/bin/civil_registry/civil_registry_proxy.jar /dhis2Person.ds
setfacl -m u:civil_registry:r /usr/local/bin/civil_registry/civil_registry_proxy.jar
EOF
```

### B.7 — Write `application.yml` (with real secrets)

The `dhis2-server-tools` role templates
[`application.yml.j2`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/templates/application.yml.j2).
The Uzbekistan variant needs the two SSV endpoints instead of a single
`civil-registry-url`:

```sh
lxc exec civil-registry-prod -- tee /usr/local/etc/application.yml >/dev/null <<'YAML'
server:
  port: 9070
  address: 0.0.0.0

civil-registry-citizen-url: https://api2.ssv.uz/v3/Citizen
civil-registry-citizen-address-url: https://api2.ssv.uz/v3/CitizenAddress

dataSonnet:
  dhis2Person: resource:file:/dhis2Person.ds

oauth2:
  tokenEndpoint: https://sso.ssv.uz/oauth/token
  clientId: <PROD_CLIENT_ID>
  clientSecret: <PROD_CLIENT_SECRET>

logging:
  file:
    name: civil-registry-middleware.log
  level:
    findRoute: INFO
YAML

lxc exec civil-registry-prod -- chown root:root /usr/local/etc/application.yml
lxc exec civil-registry-prod -- chmod 0640 /usr/local/etc/application.yml
```

> **Never commit real client secrets to git.** Keep the production
> `application.yml` in an Ansible vault or an
> encrypted files store. Only render it at deploy time.

### B.8 — Systemd unit

Same shape as
[`systemd_service.j2`](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/templates/systemd_service.j2)
in the reference role:

```sh
lxc exec civil-registry-prod -- tee /etc/systemd/system/civil_registry.service >/dev/null <<'UNIT'
[Unit]
Description=Civil Registry Middleware
After=syslog.target network-online.target
Wants=network-online.target

[Service]
User=civil_registry
Group=civil_registry
ExecStart=/usr/lib/jvm/java-17-openjdk-amd64/bin/java \
    -Xms512m -Xmx1024m \
    -jar /usr/local/bin/civil_registry/civil_registry_proxy.jar \
    --spring.config.location=/usr/local/etc/application.yml \
    --server.port=9070
SuccessExitStatus=143
TimeoutStopSec=10
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT

lxc exec civil-registry-prod -- systemctl daemon-reload
lxc exec civil-registry-prod -- systemctl enable --now civil_registry
lxc exec civil-registry-prod -- systemctl status civil_registry --no-pager
```

Watch the logs:
```sh
lxc exec civil-registry-prod -- journalctl -u civil_registry -f
```

### B.9 — Firewall (UFW)

Restrict access to port `9070` so that **only** the DHIS2 core LXC
container can reach the middleware. This mirrors
[`tasks/lxd.yml` + firewall rules](https://github.com/dhis2/dhis2-server-tools/blob/main/deploy/roles/integration/tasks/allow-acces-on-dhis2-firewall.yml):

```sh
lxc exec civil-registry-prod -- bash -eux <<'EOF'
ufw allow from <DHIS2_IP> to any port 9070 proto tcp
ufw --force enable
ufw status verbose
EOF
```

**Port 9071 (Hawtio + Spring management)** must never be exposed
publicly. If need its needed, tunnel through SSH.

### B.10 — Configure the DHIS2 route (once, per environment)

The route lives in DHIS2, not on the middleware host. See
[config/dhis2/route.json](config/dhis2/route.json). Point the `url`
at the middleware LXC container's IP (or hostname resolvable from the
DHIS2 container):

```sh
curl -u admin:<pwd> -H 'Content-Type: application/json' \
     -d @config/dhis2/route.json \
     https://<dhis2-host>/api/routes
```

For prod, harden the route by adding an API-key `auth` block and
`sharing` rules restricting it to specific user groups.

### B.11 — Install / update the plugin

Plugin deployments do **not** involve LXC, systemd, or the middleware.
Upload directly to DHIS2:

```sh
curl -u <service-user>:<pwd> \
     -F file=@plugin/build/bundle/civil-registry-plugin-1.0.0.zip \
     https://<dhis2-host>/api/apps
```

Use a dedicated service user (not `admin`) with the
`M_dhis-web-maintenance-appmanager` authority. Uploading a same-named
app replaces the previous version in place — **no DHIS2 restart is
required**. End users may need to reload the Capture app tab to pick
up the new bundle.

### B.12 — Datastore seed (once per environment)

Also install the JSONata mapping expression:

```sh
curl -u <service-user>:<pwd> \
     -H 'Content-Type: application/json' \
     -d @config/dhis2/personMap.json \
     https://<dhis2-host>/api/dataStore/civilRegistryPlugin/personMap
```

The default value `"$"` is a pass-through — the middleware still does
the mapping. To move mapping into DHIS2, replace this expression with
a real one (no plugin rebuild needed).

---

## Section C — Redeployment / upgrades

### C.1 — Plugin only (no DHIS2 restart)

```sh
cd plugin && yarn build --no-verify && cd ..
curl -u <service-user>:<pwd> \
     -F file=@plugin/build/bundle/civil-registry-plugin-1.0.0.zip \
     https://<dhis2-host>/api/apps
```

### C.2 — Middleware JAR only (~30 s downtime)

```sh
# On workstation
mvn -B clean package -DskipTests -f middleware/pom.xml

# Push new JAR next to the current one
JAR=middleware/target/civil-registry-middleware-1.0.0-SNAPSHOT.jar
lxc file push "$JAR" \
   civil-registry-prod/usr/local/bin/civil_registry/civil_registry_proxy.jar.new

# On the LXC container
lxc exec civil-registry-prod -- bash -eux <<'EOF'
cd /usr/local/bin/civil_registry
mv civil_registry_proxy.jar     civil_registry_proxy.jar.bak-$(date +%Y%m%d)
mv civil_registry_proxy.jar.new civil_registry_proxy.jar
chown root:root civil_registry_proxy.jar
chmod 0640       civil_registry_proxy.jar
setfacl -m u:civil_registry:r   civil_registry_proxy.jar
systemctl restart civil_registry
sleep 5
systemctl status civil_registry --no-pager
EOF
```

Rollback:
```sh
lxc exec civil-registry-prod -- bash -eux <<'EOF'
cd /usr/local/bin/civil_registry
mv civil_registry_proxy.jar civil_registry_proxy.jar.failed
mv civil_registry_proxy.jar.bak-YYYYMMDD civil_registry_proxy.jar
systemctl restart civil_registry
EOF
```

### C.3 — DHIS2 core

Out of scope for this guide — follow
[`dhis2/dhis2-server-tools/deploy/roles/instance`](https://github.com/dhis2/dhis2-server-tools/tree/main/deploy/roles/instance).
DHIS2 core upgrades are independent of the plugin and the middleware.

---

## Section D — Troubleshooting

### D.1 — Podman

| Symptom | Likely cause | Fix |
|---|---|---|
| `Error: short-name "nginx" did not resolve to an alias` | Missing `registries.conf` | Re-do Step 0 → Podman config block |
| `depends_on condition: service_healthy` ignored | `podman-compose` too old | Bring services up in sequence: `db` → wait 60 s → `dhis2` → wait 120 s → `gateway middleware seed` |
| Image pull hangs | Docker Hub rate limit | `podman login docker.io` |

### D.2 — Middleware

| Symptom | Likely cause | Fix |
|---|---|---|
| `Unable to access jarfile` | Wrong path or permissions | `ls -l /usr/local/bin/civil_registry/`, check ACL: `getfacl civil_registry_proxy.jar` |
| Boots then `503` on `/api/find` | OAuth2 secrets wrong | `curl -v <OAUTH2_TOKENENDPOINT>`, verify `clientId` / `clientSecret` |
| Systemd loops `Restart=on-failure` | Config error in `application.yml` | `journalctl -u civil_registry -n 200 --no-pager` |
| Port 9070 refused from DHIS2 | UFW rule missing | `ufw status`, add DHIS2 IP |

### D.3 — Plugin

| Symptom | Likely cause | Fix |
|---|---|---|
| `Patient ID` field doesn't render | `pluginConfig` not seeded | Repost [config/dhis2/pluginConfig.json](config/dhis2/pluginConfig.json) to `/api/40/dataStore/capture/dataEntryForms` |
| Warning: *"Civil registry mapping has not been set up"* | `personMap` datastore key missing | Repost [config/dhis2/personMap.json](config/dhis2/personMap.json) |
| Warning: *"Failed to query civil registry"* | Route not reachable, middleware down, or SSV credentials wrong | Check `/api/routes/civil-registry/run` from DHIS2, then middleware logs |
| Search returns fields but names are garbled | JSONata expression outdated vs middleware payload | Edit the `civilRegistryPlugin/personMap` key in the Datastore app |

### D.4 — Getting a fresh start locally

```sh
podman-compose down --volumes
podman system prune -f
rm -rf middleware/target plugin/build plugin/node_modules
```

---

## Reference

- Reference implementation (upstream): https://github.com/dhis2/reference-civil-registry-lookup
- DHIS2 server tools (LXC role we mirror): https://github.com/dhis2/dhis2-server-tools/tree/main/deploy/roles/integration
- DHIS2 Capture plugin developer docs: https://developers.dhis2.org/docs/capture-plugins/developer/getting-started
- Spring Boot external config: https://docs.spring.io/spring-boot/reference/features/external-config.html
