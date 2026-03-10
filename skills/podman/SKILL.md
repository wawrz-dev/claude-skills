---
name: podman
description: "Container engine reference for Podman and Podman Compose. INVOKE THIS SKILL whenever writing Dockerfiles, Containerfiles, docker-compose.yml, compose.yaml, container commands, or Makefile targets that build/run containers. Also invoke when the user mentions podman, containers, rootless containers, or is migrating from Docker. This skill ensures commands target Podman (not Docker) and handles rootless gotchas, networking, and compose differences."
---

## Why This Skill Exists

Podman is a drop-in replacement for Docker with the same CLI, but its **daemonless, rootless-by-default** architecture creates subtle differences that cause real bugs — permission errors on volume mounts, compose networking surprises, port binding failures. This skill prevents those issues by giving you the right mental model and the correct flags upfront.

The golden rule: `podman` accepts the same flags as `docker` in almost all cases. When it doesn't, this skill tells you why and what to use instead.

---

## Architecture: How Podman Differs from Docker

| Aspect | Docker | Podman |
|--------|--------|--------|
| Daemon | `dockerd` (root, always running) | None — fork/exec per container |
| Default privileges | Root (rootless optional) | Rootless (rootful optional) |
| Socket | `/var/run/docker.sock` | `$XDG_RUNTIME_DIR/podman/podman.sock` |
| Image storage (rootless) | `/var/lib/docker` | `~/.local/share/containers/storage` |
| Networking backend | Docker bridge | **Netavark** (default since 4.0) |
| Rootless networking | RootlessKit / VPNKit | **Pasta** (default) or slirp4netns |
| Build backend | BuildKit | **Buildah** (integrated) |
| Compose | `docker compose` (built-in plugin) | `podman compose` (thin wrapper) |
| Orchestration | Swarm | K8s YAML via `podman play kube` |

The practical consequence: root and rootless Podman are **separate universes**. `podman images` and `sudo podman images` show different image stores. Never mix them accidentally.

---

## CLI Quick Reference

### Direct Docker equivalents (identical syntax)
These commands work exactly the same — just replace `docker` with `podman`:

```
run, ps, pull, push, build, exec, logs, stop, start, rm, rmi,
images, inspect, cp, export, import, tag, commit, login, logout,
volume create|ls|rm|inspect|prune,
network create|ls|rm|inspect|connect|disconnect|prune
```

### Podman-only commands
```bash
podman pod create|ls|stop|rm|inspect   # Kubernetes-style pod grouping
podman generate kube <container>       # Export to K8s YAML
podman play kube deployment.yaml       # Run K8s YAML locally
podman generate systemd --new <ctr>    # Create systemd unit
podman machine init|start|stop|ssh     # Manage Linux VM (macOS/Windows)
podman system service                  # Start Docker-compatible API socket
podman unshare                         # Run command in user namespace
```

---

## Containerfile / Dockerfile

Podman looks for **Containerfile** first, then **Dockerfile**. The syntax is identical — same FROM, RUN, COPY, etc. Both produce OCI-compliant images.

```bash
podman build -t myimage .              # Finds Containerfile or Dockerfile
podman build -f Dockerfile -t myimage . # Explicit file
```

Build differences from Docker:
- Backend is **Buildah**, not BuildKit. `DOCKER_BUILDKIT=1` has no effect.
- `RUN --mount=type=cache` works in Buildah >= 1.24
- `--format docker` (default) or `--format oci` controls image format
- Multi-stage builds and layer caching work the same

---

## Podman Compose

`podman compose` is a **thin wrapper** that delegates to an external provider:

1. **`docker-compose`** — takes precedence if installed
2. **`podman-compose`** (`pip install podman-compose`) — community Python tool

The wrapper sets env vars so the provider talks to Podman's socket.

```bash
# Override which provider is used
export PODMAN_COMPOSE_PROVIDER=/path/to/provider

# Suppress "external compose" warnings
export PODMAN_COMPOSE_WARNING_LOGS=false
```

### Using docker-compose with Podman's socket directly
```bash
# Start Podman's Docker-compatible API
podman system service --time=0 unix:///tmp/podman.sock &
export DOCKER_HOST=unix:///tmp/podman.sock
docker-compose up -d
```

### Compose gotchas vs Docker Compose
- `podman-compose` may not support every Compose directive — test complex configs
- `depends_on` with `condition: service_healthy` works but healthcheck timing can differ
- `network_mode: host` still has user-namespace isolation in rootless mode
- Bind mounts may need SELinux labels (`:z` or `:Z`) on RHEL/Fedora — see Volumes section
- Build context uses Buildah instead of BuildKit (transparent, but BuildKit-specific Dockerfile syntax may not work in older Podman versions)

---

## Rootless Containers

Rootless is the default and the main source of "it works in Docker but not Podman" bugs.

**How it works:** your container's root (UID 0) maps to your host UID via user namespaces. Subordinate UIDs are configured in `/etc/subuid` and `/etc/subgid`.

### Common rootless issues and fixes

**Port binding below 1024:**
```bash
# Fails in rootless:
podman run -p 80:80 nginx

# Fix: lower the unprivileged port floor
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Or just use a high port (preferred):
podman run -p 8080:80 nginx
```

**Volume permission errors:**
```bash
# Problem: container expects UID 0 but host files are owned by your UID
# Fix: --userns=keep-id maps your host UID into the container
podman run --userns=keep-id -v ./data:/data:Z myimage

# Map to a specific container user
podman run --userns=keep-id:uid=1000,gid=1000 -v ./data:/data myimage
```

**"Image not found" after using sudo:**
```bash
# These are SEPARATE image stores:
podman pull myimage          # your user's storage
sudo podman pull myimage     # root's storage
# Don't mix them. Pick one.
```

---

## Networking

### Netavark (default backend since Podman 4.0)
- DNS resolution between containers on the same network works automatically
- Default bridge subnet: `10.88.0.0/16`

### Rootless networking uses Pasta (default)
- Copies host IPs into container namespace (no NAT)
- Alternative: `slirp4netns` (older, `--network=slirp4netns`)

```bash
# Create and use a custom network
podman network create mynet
podman run --network mynet --name web nginx
podman run --network mynet alpine ping web   # DNS resolves by name

# Connect a running container
podman network connect mynet existing-container
```

---

## Volumes

### SELinux labels (RHEL, Fedora, CentOS)
```bash
# :z — shared label, multiple containers can access
podman run -v /host/path:/ctr/path:z myimage

# :Z — private label, only this container
podman run -v /host/path:/ctr/path:Z myimage

# On macOS or non-SELinux systems, :z/:Z are safely ignored
```

### Auto-fix ownership with `:U`
```bash
# :U automatically adjusts mounted directory ownership to match the container user
# This is the simplest fix for rootless UID mapping permission issues
podman run -v ./data:/app/data:U myimage

# Combine with SELinux label on Fedora/RHEL:
podman run -v ./data:/app/data:U,Z myimage
```

### Podman-specific volume features
```bash
podman volume export myvol -o backup.tar   # Backup
podman volume import myvol backup.tar      # Restore
podman volume mount myvol                  # Mount to host filesystem
```

---

## Podman Machine (macOS)

Containers are Linux processes, so macOS needs a Linux VM. Podman Machine manages this.

```bash
# First-time setup
podman machine init --cpus 4 --memory 8192 --disk-size 100
podman machine start

# Verify
podman machine list
podman info

# Day-to-day
podman machine stop
podman machine start
podman machine ssh            # Shell into the VM

# Adjust resources (machine must be stopped first)
podman machine stop
podman machine set --cpus 6 --memory 16384
podman machine start

# Nuclear option — destroys ALL machines, images, containers
podman machine reset
```

### macOS-specific gotchas
- Volume mounts go through **virtio-fs** to share host dirs with the VM — file watching for hot-reload may be slower than Docker Desktop
- Default machine name: `podman-machine-default`
- Machine config lives in `~/.config/containers/podman/machine/`
- If `podman` commands fail with "cannot connect", run `podman machine start`

---

## Makefile Patterns

Always use a `CONTAINER_ENGINE` variable — never hardcode `podman` or `docker` in targets. This is the single most important pattern for portable Makefiles, because it lets Docker users run the same targets without editing anything.

```makefile
# These two lines go at the top of the Makefile, before any targets.
# Every container command below MUST use these variables, not bare podman/docker.
CONTAINER_ENGINE ?= podman
COMPOSE ?= $(CONTAINER_ENGINE) compose

.PHONY: up down build shell

up:
	$(COMPOSE) up -d

down:
	$(COMPOSE) down

build:
	$(COMPOSE) build

shell:
	$(COMPOSE) exec agent-service /bin/bash

# Docker users override with: make up CONTAINER_ENGINE=docker
```

---

## Migration Cheat Sheet: Docker → Podman

**Step 1 — Quick alias** (for muscle memory):
```bash
alias docker=podman
```

**Step 2 — Compose:**
```bash
# Option A: podman-compose
pip install podman-compose && podman-compose up -d

# Option B: podman compose wrapper
podman compose up -d

# Option C: docker-compose with Podman socket
systemctl --user start podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker-compose up -d
```

**Step 3 — Fix common issues:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| Permission denied on bind mount | Rootless UID mapping | `--userns=keep-id` and/or `:Z` suffix |
| Cannot bind port 80 | Rootless port restriction | `sysctl net.ipv4.ip_unprivileged_port_start=80` or use port ≥1024 |
| "Short-name resolution" prompt | Podman doesn't default to docker.io | Set `unqualified-search-registries = ["docker.io"]` in `/etc/containers/registries.conf` |
| Image missing after sudo | Root/rootless are separate stores | Stay consistent — don't mix `sudo podman` and `podman` |
| Docker socket expected by a tool | No daemon = no socket by default | `systemctl --user enable --now podman.socket` + set `DOCKER_HOST` |

---

## Out of Scope

This skill does not cover: Kubernetes deployment (use kubectl/Helm), Docker Swarm (not supported), Podman Desktop GUI, standalone Buildah/Skopeo usage, or CRI-O.
