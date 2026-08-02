# 05 — Building a custom Docker image

## Goal

Everything in the stack so far used images built by someone else
(`prom/prometheus`, `grafana/grafana`, `prom/node-exporter`, `nginx`). This was
the first attempt at writing an application myself, packaging it with a
Dockerfile, and running it as a container from scratch, using a small Python
script to keep the focus on Docker itself rather than application logic.

## The app

A minimal Python HTTP server (`app.py`) that responds to any GET request with
a random number, in the same plain-text metric format node-exporter uses:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import random

class MetricsHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        value = random.uniform(0, 100)
        body = f"my_random_metric {value}\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(body.encode())

HTTPServer(("0.0.0.0", 8000), MetricsHandler).serve_forever()
```

No real logic on purpose. The goal was to isolate the Dockerfile itself, not
debug application code at the same time.

## The Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python3", "app.py"]
```

- `FROM python:3.12-slim` — start from an official image that already has
  Python installed, instead of installing it by hand on a bare Linux image.
- `WORKDIR /app` — every following instruction runs as if `cd /app` had been
  executed inside the image being built.
- `COPY app.py .` — copies the file from the host into the image, into the
  directory set by `WORKDIR`. This only happens once, at build time.
- `CMD ["python3", "app.py"]` — the command that runs when a container is
  started from this image. Unlike the lines above, this doesn't execute
  during `docker build`; it's only recorded as an instruction for `docker run`.

## Build vs run

`docker build -t random-metrics .` executes every instruction above except
`CMD`, and produces a static image, a filesystem snapshot with Python and the
script baked in. `docker run` (or `docker compose up`) takes that already-built
image and starts a live process from it, executing whatever `CMD` recorded.
Building happens once per change to the code; running can happen any number
of times from the same build, exactly like baking a cake once and eating
multiple slices from it later.

## Wiring it into the existing stack

Rather than running the container by hand, it was added as a service in the
same `docker-compose.yml` used for Prometheus and Grafana:

```yaml
random-metrics:
  build: ../random-metrics
  container_name: random-metrics
  ports:
    - "8000:8000"
  restart: unless-stopped
```

`build: ../random-metrics` points Compose at the folder containing the
Dockerfile (not the Dockerfile itself; Compose looks for a file with that
exact name inside the given directory). `docker compose up -d --build` forces
a rebuild from that Dockerfile instead of assuming a cached image is still
current.

## Gotcha: one missing letter, target down

The service in `docker-compose.yml` was named `random-metrics` (plural). The
target added to `prometheus.yml` was:

```yaml
- job_name: "random-metric"
  static_configs:
    - targets: ["random-metric:8000"]
```

A typo, missing the trailing `s`. Docker's internal DNS only resolves exact
service names, so `random-metric` and `random-metrics` are two completely
different hosts to it. The target showed as DOWN on `/targets`, with nothing
in the container logs pointing at the actual cause. The container itself was
healthy; it just had the wrong name pointed at it.

Fix was a one-character change, matching the target to the real service name:

```yaml
- job_name: "random-metric"
  static_configs:
    - targets: ["random-metrics:8000"]
```

(the `job_name` label itself doesn't need to match anything, it's just a
display name; only the value inside `targets` has to match the real service
name exactly)

This is the same category of mistake as the `alert-rules.yml` vs
`alert_rules.yml` issue from the monitoring setup: a naming mismatch between
two files or configs that looks identical at a glance and produces no error
message pointing at the real cause, just a target quietly sitting DOWN.

## What I learned

- **A Dockerfile is a recipe, not a running thing.** `FROM`, `WORKDIR`, `COPY`
  execute at build time and get baked into the image. `CMD` is just stored,
  and only runs when a container starts.
- **An image built from your own code behaves identically to one pulled from
  Docker Hub.** Same `docker ps` output, same networking rules, same
  everything. The only difference is who wrote the Dockerfile.
- **Naming mismatches are the recurring failure mode in this whole stack.**
  First `alert-rules.yml` vs `alert_rules.yml`, now `random-metric` vs
  `random-metrics`. Docker's service discovery is exact-match; a single
  missing letter is enough to break it silently, with no error beyond
  "target down."

_Debugged with AI in the loop; documented afterwards to make sure I understood it,
not just pasted it._
