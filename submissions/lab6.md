# Lab 6 — Containers: Dockerize QuickNotes

**Author:** HNS ([@HNS2112](https://github.com/HNS2112))
**Date:** 5 August 2026
**Environment:** macOS on Apple Silicon (arm64), Docker 29.6.2, Compose v5.3.1

All images in this report are `linux/arm64`, built and measured on the same
machine in one session. The `hello-world` smoke test reported `(arm64v8)`,
confirming native execution rather than emulation.

---

## Task 1 — Multi-Stage Dockerfile

### 1.1 The Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.24.6-bookworm AS builder
WORKDIR /src
COPY go.mod go.su[m] ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build \
      -trimpath \
      -ldflags='-s -w' \
      -o /out/quicknotes .
RUN mkdir -p /data-empty

FROM busybox:1.37-uclibc AS busybox

FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=builder /out/quicknotes /app/quicknotes
COPY --from=builder /src/seed.json /app/seed.json
COPY --from=busybox /bin/wget /bin/wget
COPY --from=builder --chown=65532:65532 /data-empty /data
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/quicknotes"]
```

Two details are not obvious from the requirements and are explained in §1.6 and
§2.5: the `go.su[m]` glob, and the `/data-empty` directory copied with `--chown`.

### 1.2 Image size

```console
$ docker images quicknotes:lab6
IMAGE             ID             DISK USAGE   CONTENT SIZE
quicknotes:lab6   8e0d9ff6b7cc       16.2MB         3.88MB
```

Size comparison across the three images involved:

| Image | Disk usage |
|---|---|
| `golang:1.24.6-bookworm` (builder) | 1.26 GB |
| `gcr.io/distroless/static:nonroot` (runtime base) | 6.37 MB |
| `quicknotes:lab6` (final) | **16.2 MB** |

The multi-stage split discards 99% of the weight: the toolchain never reaches
the runtime image. The final image is comfortably under the 25 MB limit even
after adding `wget` (see §2.5 e), which cost roughly 2 MB.

### 1.3 Runtime configuration

```console
$ docker inspect quicknotes:lab6 --format '{{json .Config}}' | python3 -m json.tool
{
    "User": "nonroot:nonroot",
    "ExposedPorts": { "8080/tcp": {} },
    "Env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt"
    ],
    "Entrypoint": ["/app/quicknotes"],
    "WorkingDir": "/app"
}
```

### 1.4 It runs

```console
$ docker run --rm -d -p 8080:8080 --name qn quicknotes:lab6
$ curl -s http://localhost:8080/health
{"notes":4,"status":"ok"}

$ docker logs qn
2026/08/05 17:38:23 quicknotes listening on :8080 (notes loaded: 4)
```

Four seed notes loaded, so `SEED_PATH` resolved correctly against `WORKDIR /app`.

### 1.5 Design questions

#### a) Why does layer order matter?

`COPY go.mod go.su[m] ./` and `RUN go mod download` come before `COPY . .`
deliberately. Docker caches layers by the hash of their inputs, so as long as
`go.mod`/`go.sum` have not changed, the dependency-download layer is reused in
full even when the entire rest of the codebase changed.

Measured on this machine:

| Build | Time |
|---|---|
| Cold (`--no-cache`) | 11.1 s |
| Incremental (source edited only) | 4.4 s |

The log confirms which layers were reused:

```
=> CACHED [builder 2/6] WORKDIR /src
=> CACHED [builder 3/6] COPY go.mod go.su[m] ./
=> CACHED [builder 4/6] RUN go mod download
=> [builder 5/6] COPY . .
=> [builder 6/6] RUN CGO_ENABLED=0 go build ...
```

A secondary effect showed up in the build context: it shrank from 17.40 kB to
2.12 kB between the two builds. That is not `.dockerignore` — the log reports
`transferring context: 2B`, meaning no such file exists. It is BuildKit keeping
a snapshot of the previous context and transferring only what actually changed.

Adding a `.dockerignore` would still be worth doing: a local `data/` directory
would otherwise land in the build context for no reason.

**Honest caveat on the before/after comparison the question asks for.**
QuickNotes has zero external dependencies — `go.mod` has no `require` block and
no `go.sum` exists. So the badly-ordered variant (`COPY . .` then
`go mod download`) would download nothing either way, and the measured
difference between the two strategies on *this* project would be close to noise.
The ordering shown here is correct practice, but this codebase cannot
demonstrate its full payoff.

#### b) Why `CGO_ENABLED=0`?

Without it, Go may link the binary dynamically against the system libc — cgo
gets pulled in by packages such as `net`, which prefers the cgo-based resolver
when it is available. `distroless/static` contains no libc and no dynamic
loader, so such a binary fails at startup with an error along the lines of
`no such file or directory` — misleading, because the file *is* there; what is
missing is the interpreter named in its ELF header.

`CGO_ENABLED=0` produces a fully static binary with no external dependencies,
which is the only kind that can run in `distroless/static`.

#### c) What is `gcr.io/distroless/static:nonroot`?

Not emptiness — a stripped-down Debian. Trivy identifies the base directly as
`debian 13.6` and reports zero vulnerabilities in it. The package manager, the
shell, and everything not needed to launch a static binary have been removed.

Something useful does remain: `docker inspect` shows
`SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt`, so the CA bundle is present
and an application can validate TLS chains for outbound HTTPS without shipping
certificates itself. That is the practical difference from `scratch`.

So "distroless" means an OS with everything that gives an attacker a foothold
removed — shell, package manager, spare binaries — while keeping the minimum
runtime dependencies a static Go binary actually needs.

For CVEs this matters twice over. Fewer packages means fewer things that *can*
have a CVE; §B.3 shows the OS layer at zero. It also means fewer things a
scanner has to track, so the noise floor of a security report drops.

#### d) `-ldflags='-s -w'` and `-trimpath`

`-s` removes the ELF symbol table, `-w` removes DWARF debug information.
Measured on this codebase:

```console
$ go build -o /tmp/qn-full .
$ go build -ldflags='-s -w' -o /tmp/qn-strip .
$ ls -la /tmp/qn-full /tmp/qn-strip
-rwxr-xr-x  1 elvira  staff  8450114  /tmp/qn-full
-rwxr-xr-x  1 elvira  staff  5781970  /tmp/qn-strip
```

8.06 MB → 5.51 MB, a 31.6% reduction.

`-trimpath` strips absolute build-machine paths from the binary. Without it the
binary embeds paths like `/src/...` or a CI runner's home directory — this is
both a size cost and a small information leak about the build environment. It
also makes builds reproducible: the same source produces the same bytes
regardless of where it was compiled.

**What the cost actually is.** Panic stack traces survive: Go stores the
information needed for them in `pclntab`, embedded in the binary, not in the
ELF symbol table or DWARF. A production panic still prints file, line and
function names. What is lost is interactive debugging — `gdb` and `delve` need
DWARF to map variables, stack frames and types, so attaching a debugger to a
stripped process gives very little. The workaround is to rebuild the same commit
locally without the flags.

### 1.6 A note on `go.su[m]`

Requirement 9 asks to copy `go.mod` and `go.sum` before the sources. QuickNotes
has no `go.sum` — with no external dependencies there is nothing to checksum —
and `COPY go.mod go.sum ./` fails the build on a missing file. Writing the path
as `go.su[m]` turns it into a glob, which matches the file when it exists and
silently matches nothing when it does not. The layer ordering is preserved and
the build works both before and after the project acquires dependencies.

---

## Task 2 — Compose, Healthcheck and Persistent Volume

### 2.1 compose.yaml

```yaml
services:
  quicknotes:
    build: ./app
    image: quicknotes:lab6
    ports:
      - "8080:8080"
    environment:
      ADDR: ":8080"
      DATA_PATH: /data/notes.json
      SEED_PATH: /app/seed.json
    healthcheck:
      test: ["CMD", "/bin/wget", "-q", "-O", "-", "http://127.0.0.1:8080/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    volumes:
      - quicknotes-data:/data
    restart: unless-stopped
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true

volumes:
  quicknotes-data:
```

`DATA_PATH` is absolute on purpose. The application's default is
`data/notes.json` — a relative path, which would resolve to `/app/data/notes.json`
and miss the mounted volume entirely, leaving the persistence test to fail
silently with data written to the container's ephemeral layer.

### 2.2 The volume permission failure

The first `docker compose up` produced a crash loop:

```console
$ docker compose ps
NAME                        STATUS
devops-intro-quicknotes-1   Restarting (1) 1 second ago

$ docker compose logs quicknotes | tail -3
quicknotes-1  | 2026/08/05 17:55:35 seed: open /data/notes.json: permission denied
quicknotes-1  | 2026/08/05 17:56:01 seed: open /data/notes.json: permission denied
```

The health status was `unhealthy` with an empty log — the check never got to run
at all, because the container kept dying before `start_period` elapsed.

Root cause: Docker creates a named volume owned by root, while the container runs
as UID 65532. This is listed in the lab's own pitfalls, and the usual fix —
`RUN chown` — is unavailable here, because distroless has no shell for `RUN` to
execute.

The fix uses the fact that Docker copies ownership and permissions from the
mount point in the image when it initialises a new empty volume. An empty
directory is created in the builder stage and copied across with an explicit
owner:

```dockerfile
RUN mkdir -p /data-empty                                   # builder stage
COPY --from=builder --chown=65532:65532 /data-empty /data  # runtime stage
```

`--chown` on `COPY` is a build-time instruction, not a command run inside the
image, so it works without a shell. After this the container came up
`Up 22 seconds (healthy)`.

### 2.3 Persistence test

```console
$ curl -s -X POST -H 'Content-Type: application/json' \
    -d '{"title":"durable","body":"survive a restart"}' \
    http://localhost:8080/notes
{"id":5,"title":"durable","body":"survive a restart","created_at":"2026-08-05T18:01:18.478288008Z"}

$ curl -s http://localhost:8080/notes | grep durable
... {"id":5,"title":"durable", ...}          # present

$ docker compose down && docker compose up -d && sleep 8
[+] down 2/2
 ✔ Container devops-intro-quicknotes-1 Removed
 ✔ Network devops-intro_default        Removed
 (no volume line)

$ curl -s http://localhost:8080/notes | grep durable
... {"id":5,"title":"durable", ...}          # STILL present

$ docker compose down -v && docker compose up -d && sleep 8
[+] down 3/3
 ✔ Container devops-intro-quicknotes-1 Removed
 ✔ Network devops-intro_default        Removed
 ✔ Volume devops-intro_quicknotes-data Removed

$ curl -s http://localhost:8080/notes | grep durable
                                              # gone
```

The absence of a volume line in the first `down` and its presence in the second
is the whole mechanism, visible in the output.

### 2.4 Health status

```console
$ docker compose ps
NAME                        IMAGE             STATUS                    PORTS
devops-intro-quicknotes-1   quicknotes:lab6   Up 22 seconds (healthy)   0.0.0.0:8080->8080/tcp
```

### 2.5 Design questions

#### e) Distroless has no shell — how do you healthcheck it?

The strategy chosen is none of the obvious three: instead of a sidecar, a debug
image, or relying on process liveness, exactly one statically linked binary —
`wget` from `busybox:1.37-uclibc` — is copied into the runtime stage with
`COPY --from`, and nothing else.

Process liveness says nothing about whether the service works: the process can
be alive while the HTTP handler is wedged. A debug image or a sidecar drags in a
package manager and a shell — precisely what distroless was chosen to avoid.
A sidecar has a further problem: the health status would attach to the neighbour
container, so `quicknotes` itself would never be marked `healthy`, and anything
depending on it via `condition: service_healthy` would be answering a question
about the wrong container. One static binary is the compromise: it does exactly
one thing, needs no command interpreter, and is one `COPY` line.

The verification of the constraints below was done with that same binary, which
is a small side benefit — the tool that proves the service is up also proves the
filesystem is read-only.

**The cost, in measured terms.** The image grew from 14.2 MB to 16.2 MB, still
well inside the 25 MB limit. More importantly, a network tool now lives inside
the image: after a compromise, an attacker has a ready-made way to fetch a
payload rather than only the running process. The read-only root filesystem
partially offsets exactly this risk — the same `wget` confirmed in practice that
it cannot write to `/etc/test` — so even with a network tool present, nothing
can be written outside `/tmp` and `/data`.

Worth naming the architectural limit: Docker's healthcheck runs *inside* the
container, which is what forces this compromise at all. Kubernetes runs
`livenessProbe: httpGet` from the kubelet, outside the container, and needs
nothing added to the image. The constraint belongs to the tool, not to the idea.

#### f) Why does the volume survive `docker compose down`?

A named volume is not part of the container's lifecycle but a separate Docker
object with its own management. `docker compose down` removes only what a
particular run describes — the container and the network — and deliberately does
not touch volume data, otherwise every restart would wipe state. That is why
there was no volume line in the `down` output and why note `id:5` survived
container recreation: the same volume was mounted back at `/data` with the same
files on disk. The explicit `-v` flag is an opt-in to "delete the data too",
hence `✔ Volume ... Removed` and the note disappearing.

Beyond `-v`, a volume is destroyed by `docker volume rm <name>` directly, and by
`docker volume prune`. The `prune` case has a subtlety: without flags it removes
only *anonymous* volumes, and named ones require `--all`. That asymmetry is
deliberate — nobody created an anonymous volume on purpose, whereas a named
volume is an artifact of someone's intent, and deleting it silently is more
dangerous.

A bind mount such as `./data:/data` would behave differently in kind: the data
would live in the host filesystem, outside Docker's management, and
`docker compose down -v` would not touch it — `-v` applies to named and anonymous
volumes only. Removing it would require `rm -rf ./data` by hand. In short, a
named volume is exposed to Docker commands; a bind mount is exposed to whatever
you do to that directory yourself.

#### g) `depends_on` without `condition: service_healthy`

The default is `service_started`: Compose brings up the dependent service as soon
as the process in the `quicknotes` container has started, without waiting for the
healthcheck to report `healthy`.

This is the same trap as the sidecar in (e), from the other direction: the check
is not broken, the dependent service is simply getting the answer to a different
question — "does the container exist" rather than "is the service ready to serve
traffic". The bug it causes is a startup race: the dependent service begins
calling the API before the HTTP server is actually accepting connections,
producing connection-refused errors that disappear on a retry and are therefore
easy to dismiss as flakiness.

`condition: service_healthy` is the only variant that waits for the first
successful run of the `test` command in the healthcheck block.

---

## Bonus Task — The 6 Security Defaults

### B.1 Applied

| # | Default | Where |
|---|---|---|
| 1 | `USER nonroot` | Dockerfile |
| 2 | Distroless base | Dockerfile |
| 3 | Drop all capabilities | compose `cap_drop: [ALL]` |
| 4 | Read-only root + tmpfs | compose `read_only: true`, `tmpfs: [/tmp]` |
| 5 | `no-new-privileges` | compose `security_opt` |
| 6 | Trivy scan | §B.3 |

Nothing was added to `cap_add`: QuickNotes listens on 8080, and
`NET_BIND_SERVICE` is only needed below port 1024.

### B.2 Verification

```console
$ docker inspect quicknotes:lab6 --format '{{ .Config.User }}'
nonroot:nonroot

$ docker compose exec quicknotes sh
OCI runtime exec failed: exec failed: unable to start container process:
exec: "sh": executable file not found in $PATH

$ docker inspect $(docker compose ps -q quicknotes) --format '{{ .HostConfig.CapDrop }}'
[ALL]

$ docker inspect $(docker compose ps -q quicknotes) --format '{{ .HostConfig.ReadonlyRootfs }}'
true

$ docker inspect $(docker compose ps -q quicknotes) --format '{{ .HostConfig.SecurityOpt }}'
[no-new-privileges:true]
```

The read-only constraint was also tested behaviourally rather than by inspection,
using the `wget` already present for the healthcheck:

```console
$ docker compose exec quicknotes /bin/wget -O /etc/test http://127.0.0.1:8080/health
Connecting to 127.0.0.1:8080 (127.0.0.1:8080)
wget: can't open '/etc/test': Read-only file system
```

The request succeeded and the write failed — which is the point: the constraint
is enforced by the kernel, not merely recorded in a config field.

### B.3 Trivy — and a result the lab did not predict

```console
$ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy:0.59.1 image --severity HIGH,CRITICAL --no-progress quicknotes:lab6

quicknotes:lab6 (debian 13.6)
=============================
Total: 0 (HIGH: 0, CRITICAL: 0)

app/quicknotes (gobinary)
=========================
Total: 15 (HIGH: 14, CRITICAL: 1)
```

The lab text says a distroless-static image "often" scans at zero HIGH/CRITICAL.
The OS layer did exactly that. The Go binary did not.

All 15 findings are in the Go standard library compiled into the binary, because
it was built with Go 1.24.6. The CRITICAL one is `CVE-2025-68121`, incorrect
certificate validation during TLS session resumption in `crypto/tls`, fixed in
1.24.13. The rest are denial-of-service issues in `net/url`, `crypto/x509`,
`net`, `net/mail` and `mime`.

The conclusion is the one the numbers force: **a minimal base eliminates OS-layer
CVEs, but says nothing about what you put into it.** Trivy's `Fixed Version`
column is the remediation plan — bump the toolchain. An attempt to rebuild on
`golang:1.24.13-bookworm` and re-scan was blocked by repeated registry download
failures (`httpReadSeeker: failed open ... EOF`), so the before/after pair is not
included.

Two limits are worth naming even without that rebuild. Several findings are only
fixed in 1.25.x or 1.26.x — `CVE-2026-39822` needs 1.25.12 — so the `go 1.24`
directive in `go.mod` sets a ceiling on how far the binary can be patched at all.
And most of the DoS findings are in packages this application never exercises;
`net/mail` and `mime` are linked in but unreachable from QuickNotes' handlers.
The count is a starting point for triage, not a verdict.

This also connects back to Lab 3, where actions and the runner were pinned by SHA
for reproducibility. The scan shows the other face of pinning: a pin that is
never revisited accumulates vulnerabilities. Pinning buys control over *when* an
update happens — it does not remove the obligation to make it happen.

### B.4 Most security per line of YAML

Three candidates, judged by effect per line of configuration.

**`FROM gcr.io/distroless/static:nonroot`** — one line closing three vectors at
once: zero CVEs in the OS layer (confirmed above), no package manager, and no
shell, so even after RCE in the application an attacker has no command
interpreter to establish persistence or explore the system. It is a decision made
once at image-build time and cannot be accidentally disabled by a runtime flag.

**`cap_drop: [ALL]`** — two lines removing every Linux capability, including the
ones nobody thinks about (`CAP_SYS_PTRACE`, `CAP_NET_RAW`), not just the obvious
`NET_BIND_SERVICE` this application never needed.

**`read_only: true`** — one line, and demonstrably enforced: `wget` confirmed
`Read-only file system` on `/etc/test`. Even if malicious code reaches the
container, it cannot modify the binary, drop a persistence file, or tamper with
`seed.json`. The only writable paths are `/tmp` (tmpfs, ephemeral) and `/data`
(the volume, which is the application's expected working area).

On the strict "lines to effect" ratio I would put **`read_only: true`** first: one
line covering a whole class of post-exploitation persistence, regardless of how
the attacker got in. Distroless is larger in absolute effect but is an
architectural choice rather than a line of defence layered onto an existing
image. `cap_drop` comes third because it partly overlaps with read-only —
capabilities like `CAP_DAC_OVERRIDE` exist to bypass file permissions, and with
a read-only filesystem there is less left for them to bypass. These layers
intersect rather than adding up linearly.

---

## Summary

| Task | Status |
|------|--------|
| Task 1 — Multi-stage Dockerfile, 16.2 MB | Complete |
| Task 2 — Compose, healthcheck, persistent volume | Complete |
| Bonus — 6 security defaults + Trivy | Complete |
