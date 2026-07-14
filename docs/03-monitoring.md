# 03 - Monitoring stack: Prometheus + Grafana

## What's running

Three containers, defined in one `docker-compose.yml`:

| Container         | Role                                                                |
| ----------------- | ------------------------------------------------------------------- |
| **node-exporter** | Reads `/proc` and `/sys`, exposes system metrics over HTTP on :9100 |
| **prometheus**    | Scrapes those metrics every 15s and stores them as time series      |
| **grafana**       | Queries Prometheus and draws the dashboards                         |

## The chain

```
Linux writes its state to /proc, /sys
        ↓
node-exporter reads it, exposes it as HTTP text on :9100
        ↓
Prometheus scrapes :9100 every 15s, stores each value with a timestamp
        ↓
Grafana asks Prometheus for history and renders graphs
```

Four components, one job each. Remove Grafana and monitoring still works.
Remove Prometheus and there's nothing left to draw. It's the memory of the whole
system. node-exporter only knows _now_.

## Why pull, not push

Prometheus goes out and fetches metrics. Targets don't send anything.

The payoff: if a host dies, Prometheus knows immediately. The target stops
responding and shows up as DOWN. In a push model, silence is ambiguous. Did the
agent crash, or did it just have nothing to send?

Visible at `:9090/targets`.

## Why containers talk by name, not by IP

In `prometheus.yml` the target is `node-exporter:9100`, not an IP.
In Grafana, the datasource URL is `http://prometheus:9090`, not `192.168.x.x`.

Compose creates a dedicated network and runs an internal DNS server, so services
resolve each other by **service name**. This matters because container IPs are
assigned dynamically. They change on every restart. Names don't.

Kubernetes calls the same idea _service discovery_.

Two different worlds here:

- **container to container** (same Docker network): service name
- **browser to container** (from outside): host IP + published port

## Why named volumes

```yaml
volumes:
  - prometheus-data:/prometheus
  - grafana-data:/var/lib/grafana
```

Containers are ephemeral. Their filesystem dies with them, and containers get
recreated constantly (image updates, config changes). Without volumes, every
`docker compose down && up` would wipe all metric history and every dashboard.

Volumes live on the host, outside the container lifecycle.

## Why node-exporter needs `/` and Grafana doesn't

```yaml
volumes:
  - /:/host:ro,rslave
```

node-exporter has to read the **host's** `/proc` and `/sys`. A container only
sees its own empty filesystem by default, so without this it would report metrics
about itself instead of the machine.

- `ro` (read-only). A monitoring agent has no business writing to the host.
- `rslave` (mount propagation, host to container only). New disks mounted on the
  host become visible inside. Nothing the container mounts leaks back out.

Grafana gets none of this, because Grafana measures nothing. It only asks
Prometheus for numbers over HTTP. Least privilege, applied at the container level.

## Gotcha 1: the diagnostic tool was the liar

From inside the Prometheus container:

```
wget -qO- http://node-exporter:9100/metrics
→ wget: bad address 'node-exporter:9100'
```

Looked like broken DNS. But by IP it worked fine:

```
wget -qO- http://172.18.0.2:9100/metrics
→ metrics streaming
```

So DNS was the suspect. Then, same container, different tool:

```
nslookup node-exporter
→ Name: node-exporter
  Address: 172.18.0.2
```

DNS resolved it perfectly. And Prometheus itself scrapes that exact hostname
without complaint, visible as UP at `/targets`.

Root cause: BusyBox's `wget` (the minimal one shipped in Alpine images) is
unreliable at resolving container DNS. Nothing was broken except the tool I was
testing with.

**Lesson: when one tool says something is broken, confirm with a second tool
before you start fixing.** I nearly spent an hour repairing DNS that worked.

## Gotcha 2: YAML doesn't forgive

```yaml
- targets: ["localhost: 9090"] # broken, space after the colon
- targets: ["localhost:9090"] # correct
```

The space makes it an invalid host:port string. Prometheus couldn't scrape
itself, and the target showed DOWN. One character.

## Secrets

Grafana credentials come from environment variables, not from the compose file:

```yaml
environment:
  - GF_SECURITY_ADMIN_USER=${GRAFANA_USER}
  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
```

The actual values live in a local `.env`, which is in `.gitignore`.
`.env.example` documents the required variables without exposing them.

`.gitignore` only protects going forward. Anything committed once stays in git
history, and deleting it in a later commit does not remove it. That's the whole
reason pipelines run secret scanners like `gitleaks`.

## Deploy loop

The server never gets edited by hand:

```bash
# on the workstation
git commit && git push

# on the server
git pull
docker compose up -d          # or: docker compose restart prometheus
```

The repo is the source of truth. The server is just its reflection.

## What I learned

- **Prometheus is the memory.** Grafana is just a face. node-exporter only knows
  the present moment.
- **Pull beats push for failure detection.** Silence is unambiguous.
- **Service names over IPs.** IPs are ephemeral, names are declared.
- **Containers are disposable, data is not.** That's what volumes are for.
- **Give each container exactly the access it needs.** node-exporter gets the
  host filesystem, read-only. Grafana gets nothing.
- **A failing tool can lie about a working system.** Verify with a second tool.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._
