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

## Adding alerting rules

Dashboards are only useful if someone is staring at them. The actual point of
Prometheus is that it can watch its own data and flag a problem before a human
notices.

An alert is the same kind of query used in the Query tab, evaluated on a
schedule, with a threshold and a duration attached:

```yaml
groups:
  - name: node_alerts
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"

      - alert: DiskUsageCritical
        expr: node_filesystem_avail_bytes < node_filesystem_size_bytes * 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk usage above 90%"

      - alert: RAMUsageCritical
        expr: node_memory_MemAvailable_bytes < node_memory_MemTotal_bytes * 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "RAM usage above 90%"
```

`up` is a metric Prometheus generates automatically for every scrape target:
1 if it answered, 0 if it didn't. `for` matters as much as the threshold itself:
without it, a one-second CPU spike or a single missed scrape would fire an
alert. `for: 1m` means the condition has to hold continuously before it's
treated as real. Different alerts warrant different `for` values: something
that fails instantly and matters immediately (a target going down) gets a
short one. Something that degrades slowly (disk or memory filling up) can
tolerate a longer one without losing any real warning time.

The RAM alert deliberately uses `MemAvailable`, not a naive "total minus
used" calculation. Linux uses spare memory for disk cache and buffers, which
it hands back instantly the moment an application needs it. A calculation
based on raw "used" memory would trigger false alarms on a perfectly healthy
system just because the cache is doing its job. `MemAvailable` already
accounts for that: it estimates what a new process could actually get,
including reclaimable cache. Same comparison shape as the disk alert: the
smaller value (`MemAvailable`) checked against a fraction of the larger one
(`MemTotal`), not the other way around, since inverting them would compare
the whole to a fraction of a subset and the condition would almost never be
true.

## Gotcha 3: the rule file existed but Prometheus couldn't see it

Added `alert_rules.yml`, pointed to it from `prometheus.yml`:

```yaml
rule_files:
  - "alert_rules.yml"
```

Reloaded Prometheus. Alerts page: "No rules found."

The file existed on the host, correctly named, in the same folder as
`prometheus.yml`. But `prometheus.yml` itself only gets into the container
because of an explicit volume mount:

```yaml
volumes:
  - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
```

`alert_rules.yml` had no equivalent line. Referencing a file inside the
container's config isn't the same as making that file exist inside the
container. It needed its own mount:

```yaml
volumes:
  - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
  - ./alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
```

Second, unrelated trip on the same change: the file was named
`alert-rules.yml` on disk (hyphen) but referenced as `alert_rules.yml`
(underscore) in both `prometheus.yml` and the new volume line. Two different
filenames to the filesystem, close enough to look identical at a glance.
Once the name matched everywhere, `docker compose up -d` (not `restart`,
since the volume list itself had changed) picked it up and both rules showed
up on `/alerts` as `Inactive`, the healthy resting state.

## Deploy loop

The server never gets edited by hand:

```bash
# on the workstation
git commit && git push

# on the server
git pull
docker compose up -d          # or: docker compose restart prometheus
```

Note the distinction: `restart` is enough when only the _contents_ of an
already-mounted file changed (like adding the RAM alert to `alert_rules.yml`).
`up -d` is needed when the compose file's _structure_ changed (new volumes,
new ports, new services), since `restart` just relaunches the same container
without re-reading what should be mounted into it.

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
- **Alerting is a query plus a threshold plus a duration.** Nothing more
  mystical than that. The duration (`for`) is what separates a real signal
  from noise.
- **Percentages need the right base metric, not just any two numbers that
  seem related.** `MemAvailable` vs `MemTotal` accounts for reclaimable cache;
  the naive calculation doesn't.

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._
