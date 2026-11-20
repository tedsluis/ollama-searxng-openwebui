# Local AI Stack GitOps: Ollama + SearXNG + Open WebUI (Podman Quadlets & Materia)

This repository contains the complete declarative GitOps configuration for a local AI platform on **Fedora Linux**. It consists of five Materia components:

1. **ollama** — LLM inference server with NVIDIA GPU acceleration (CDI) and CPU pinning.
2. **searxng** — Privacy-friendly metasearch engine with JSON output, used by Open WebUI for web search.
3. **open-webui** — Web interface connected to Ollama, SearXNG and an OpenAI-compatible proxy (`genai-proxy`, see 4.9), plus database backups.
4. **renovate** — Self-hosted Renovate that proposes container image updates as pull requests.
5. **monitoring** — Health checks, a daily throughput check and failure notifications via ntfy.

All image tags are pinned in `attributes/vault.toml` and updated by Renovate; nothing runs on `latest`.

The stack is deployed as **rootful Podman Quadlets** (systemd-managed containers) and orchestrated via **[Prima Materia (`materia`)](https://github.com/stryan/materia)**.

---

## 1. System Architecture & Hardware Profile

This configuration is tuned for the following host:

| Component | Specification | Configuration / Optimization |
| :--- | :--- | :--- |
| **OS** | Fedora 45 Linux | SELinux `enforcing`, systemd, Podman 6.1 (Materia requires 5.4 or higher) |
| **CPU** | Intel Core i5 (14th Gen) | 20 logical CPUs on **14 physical cores**:<br>• **6 P-cores** (CPUs `0-11`, two threads each, up to 5000 MHz)<br>• **8 E-cores** (CPUs `12-19`, up to 3700 MHz)<br>• Energy profile: `tuned` set to `throughput-performance` |
| **Memory & Storage** | 64 GB DDR5 RAM & 1 TB NVMe SSD | Enough RAM to run the part of a model that does not fit in VRAM. |
| **GPU** | NVIDIA RTX A2000 12GB | • **12,282 MB VRAM**<br>• **PCIe Gen4 x16**, Bus ID `PCI:1:0:0`<br>• **70 W power cap** (via PCIe slot)<br>• **Driver:** `615.78.08` (CUDA `13.4`) |

### 1.1 Ollama Settings

All values are set in `components/ollama/ollama.container.gotmpl`; the variable ones come from `attributes/vault.toml`.

| Setting | Value | Purpose |
| :--- | :--- | :--- |
| `PodmanArgs=--cpuset-cpus` | `0,2,4,6,8,10,12,13,14,15,16,17,18,19` | One logical CPU per physical core (6 P-cores without their second thread + 8 E-cores). |
| `OLLAMA_CONTEXT_LENGTH` | `16384` | Default context window for every model. |
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` | Halves the memory of the K/V cache; requires Flash Attention. |
| `OLLAMA_FLASH_ATTENTION` | `1` | Required for the quantized K/V cache. |
| `OLLAMA_GPU_OVERHEAD` | `134217728` | Reserves only 128 MB of VRAM as a safety margin. |
| `OLLAMA_NUM_PARALLEL` / `OLLAMA_MAX_LOADED_MODELS` | `1` / `1` | Single-user focus: one request and one loaded model at a time. |
| `OLLAMA_KEEP_ALIVE` | `15m` | A model stays loaded for 15 minutes after its last request. |
| `OLLAMA_HOST` | `127.0.0.1:11434` | Ollama listens on localhost only. |
| `Network=host` | — | Ollama and Open WebUI share the host network stack and talk over `127.0.0.1`. |

Ollama uses 6 inference threads by default on this CPU. Benchmarks on 2026-11-20 showed that more threads (8, 10, 14) are equal or slower, and that an 8k context does not measurably speed up the models that exceed the VRAM.

### 1.2 Reference Benchmark (2026-11-20)

Measured with `ollama-diag bench` (second run, 200 tokens, 16k context). Use these numbers to judge whether the GPU path is healthy.

| Model | On GPU | VRAM | Tokens/s |
| :--- | :--- | :--- | :--- |
| `Gemma3:4b` | 100% | 3.9 GB | 62 |
| `codellama:7B` | 100% | 8.4 GB | 57 |
| `qwen3:30b-a3b` | 98% | 10.9 GB | 50 |
| `mistral:7B` | 100% | 5.6 GB | 47 |
| `gpt-oss:20b` | 90% | 10.9 GB | 43 |
| `gemma4:12b` | 100% | 8.9 GB | 36 |
| `deepseek-coder-v2:16b` | 100% | 10.9 GB | 32 |
| `Gemma3:12B` | 100% | 9.0 GB | 26 |
| `qwen3:14b` / `deepseek-r1:14b` | 100% | 10.1 GB | 24 / 23 |
| `devstral:24b` | 69% | 10.8 GB | 8 |
| `qwen3.8:27b` | 54% | 11.1 GB | 6 |

Without the GPU the same models reach 3 to 23 tokens/s (`Gemma3:4b`: 19, `gpt-oss:20b`: 14).

---

## 2. Repository Structure

```text
ollama-searxng-openwebui/
├── MANIFEST.toml                               # Assigns the five components to hostname 'fedora'
├── README.md                                   # This documentation
├── renovate.json                               # Renovate update policy (see 4.4)
├── attributes/
│   └── vault.toml                              # Central variables: ports, paths, image tags, CPU pinning, thresholds
└── components/
    ├── ollama/
    │   ├── MANIFEST.toml                       # Defaults and service definition (timeout 900 s)
    │   └── ollama.container.gotmpl             # Quadlet template for Ollama (GPU via CDI)
    ├── searxng/
    │   ├── MANIFEST.toml                       # Restarts searxng.service when settings.yml or limiter.toml changes
    │   ├── searxng.container.gotmpl            # Quadlet template for SearXNG (127.0.0.1:8112)
    │   ├── settings.yml                        # SearXNG settings: JSON output, Tor engines removed, timeouts
    │   └── limiter.toml                        # Intentionally empty; silences a startup warning
    ├── open-webui/
    │   ├── MANIFEST.toml                       # Restarts open-webui.service when open-webui.env changes
    │   ├── open-webui.container.gotmpl         # Quadlet template for Open WebUI (127.0.0.1:8080)
    │   ├── open-webui.env.gotmpl               # Environment file template
    │   ├── open-webui-backup.service.gotmpl    # Daily online backup of the SQLite database
    │   └── open-webui-backup.timer             # Runs the backup at 03:30
    ├── renovate/
    │   ├── MANIFEST.toml                       # Timer only; the container is never restarted by Materia
    │   ├── renovate.container.gotmpl           # One-shot Renovate run
    │   └── renovate.timer                      # Every 6 hours
    └── monitoring/
        ├── MANIFEST.toml                       # Registers both timers
        ├── notify-failure@.service             # Sends an ntfy message when a unit fails
        ├── ollama-diag                         # Tool: diagnostics, trigger test and benchmark (see 4.7)
        ├── ollama-bench                        # Tool: quick tokens/s measurement of one model
        ├── stack-healthcheck.service.gotmpl    # End-to-end health check (see 4.5)
        ├── stack-healthcheck.timer             # Every 15 minutes
        ├── ollama-gpu-check                    # Script: is the loaded model on the GPU?
        ├── ollama-perf-check                   # Script: tokens/s of a fixed model against a threshold
        ├── ollama-perf-check.service.gotmpl    # Unit for the throughput check
        └── ollama-perf-check.timer             # Daily at 04:15
```

---

## 3. Host Configuration (Outside this Git Repo)

Several things must be configured directly on the host before Materia can deploy this repository.

### 3.1 Host Performance & NVIDIA GPU Persistence
```bash
# 1. Set the Fedora energy profile to maximum throughput (prevents 800 MHz idle drops on P-cores)
sudo tuned-adm profile throughput-performance

# 2. Keep the NVIDIA driver active (eliminates 2–4 s startup delays between model loads)
sudo systemctl enable --now nvidia-persistenced
```

### 3.2 NVIDIA Container Toolkit & CDI (Container Device Interface)
Podman passes the GPU (`nvidia.com/gpu=all`) to the Ollama container through a CDI specification. The specification is **generated automatically** at every start of `ollama.service` (`ExecStartPre=nvidia-ctk cdi generate --output=/run/cdi/nvidia.yaml`), so it always matches the installed driver. There is no `/etc/cdi/nvidia.yaml`, and nothing has to be re-run after a driver update.

The host only needs the `nvidia-ctk` binary (version 1.17.4 at `/usr/sbin/nvidia-ctk` on this host):
```bash
# Install the toolkit if nvidia-ctk is missing
sudo dnf install -y nvidia-container-toolkit

# Verify after ollama.service has started (shows nvidia.com/gpu=0 and nvidia.com/gpu=all)
nvidia-ctk cdi list
```

### 3.3 Podman API Socket
Materia communicates with Podman via the systemd socket (`/run/podman/podman.sock`):
```bash
sudo systemctl enable --now podman.socket
```

### 3.4 Persistent Data Directories & SELinux Context
Models and Open WebUI user data reside **outside** Materia on the host disk, so they survive container updates:
* **Ollama models (`/usr/share/ollama/.ollama`):** mounted to `/root/.ollama` in the `ollama` container.
* **Open WebUI data (`/opt/open-webui`):** mounted to `/app/backend/data` in the `open-webui` container. Backups are written to `/opt/open-webui/backup`.

Ensure SELinux allows container access to these directories:
```bash
sudo chcon -Rt container_file_t /usr/share/ollama/.ollama
sudo chcon -Rt container_file_t /opt/open-webui
```

### 3.5 Secrets and Host-Only Environment Files
No secret is stored in Git. Create these once on the host:

```bash
# Podman secrets used by the Quadlets
printf '%s' "$(openssl rand -hex 32)" | sudo podman secret create searxng_secret -
printf '%s' '<api key for the OpenAI-compatible endpoint>' | sudo podman secret create openwebui_openai_api_key -

# ntfy topic for failure notifications (used by notify-failure@.service)
echo 'NTFY_URL=https://ntfy.sh/<your-unguessable-topic>' | sudo tee /etc/materia/notify.env
sudo chmod 600 /etc/materia/notify.env

# GitHub token for Renovate
echo 'RENOVATE_TOKEN=<fine-grained personal access token>' | sudo tee /etc/materia/renovate.env
sudo chmod 600 /etc/materia/renovate.env
```

The Open WebUI session key is generated by the container and stored on the data volume (`/opt/open-webui/.webui_secret_key`).

### 3.6 Installing or Upgrading Materia (`v0.8.1`)
The Materia binary and man pages live under `/usr/local`. A first install and an upgrade use the same steps; on a first install, skip the steps marked *(upgrade only)*.

```bash
VERSION=v0.8.1
ARCH=amd64
BASE=https://github.com/stryan/materia/releases/download/${VERSION}
cd "$(mktemp -d)"

# 1. (upgrade only) Show the installed version
materia version

# 2. Download the tarball and the checksums, then verify
curl -fsSLO ${BASE}/materia-${VERSION}-${ARCH}.tar.gz
curl -fsSLO ${BASE}/SHA256SUMS
sha256sum --check --ignore-missing SHA256SUMS      # must print: materia-v0.8.1-amd64.tar.gz: OK

# 3. (upgrade only) Pause the GitOps timer so that no deploy runs while the binary is replaced
sudo systemctl stop materia-sync.timer
systemctl is-active materia-sync.service           # must print "inactive" before you continue

# 4. (upgrade only) Keep the old binary for a rollback
sudo cp -a /usr/local/bin/materia /usr/local/bin/materia.previous

# 5. Install the binary and the man pages
tar -xzf materia-${VERSION}-${ARCH}.tar.gz
sudo install -m 0755 bin/materia /usr/local/bin/materia
sudo cp -r man/* /usr/local/share/man/ && sudo mandb -q
sudo mkdir -p /etc/materia /var/lib/materia

# 6. Verify: the new version is reported and the plan contains no unexpected changes
materia version
sudo materia plan -c /etc/materia/config.toml

# 7. (upgrade only) Resume the GitOps timer
sudo systemctl start materia-sync.timer
```

* **Signature (optional):** the release signs its checksum file. Download `SHA256SUMS.cosign.bundle` from the same release and the public key from `https://primamateria.systems/cosign.pub`, then run `cosign verify-blob --key cosign.pub --bundle SHA256SUMS.cosign.bundle SHA256SUMS`.
* **Rollback:** `sudo install -m 0755 /usr/local/bin/materia.previous /usr/local/bin/materia`.
* **Why v0.8.1:** the per-service `Timeout` values in the component manifests are honoured while Materia waits for a service to stabilise.

### 3.7 Materia Configuration File (`/etc/materia/config.toml`)
This file is host-specific and not in Git. On this host it defines:
* the Git source over **HTTPS** (the repository is public, so no SSH key is stored on the server), branch `main`;
* file-based attributes with `base_dir = "attributes"`;
* `hostname = "fedora"`, which must match the host entry in `MANIFEST.toml`;
* a file lock, so that two Materia runs cannot overlap;
* a service timeout of 900 seconds.

Show the effective configuration with `sudo materia config -c /etc/materia/config.toml`.

### 3.8 Host Paths Generated by Materia
When `materia update` runs, Materia renders the templates to these host paths:
* **Local Git cache:** `/var/lib/materia/source/`
* **Quadlets:** `/etc/containers/systemd/<component>/` — `ollama.container`, `searxng.container`, `open-webui.container`, `renovate.container`
* **Plain systemd units:** `/etc/systemd/system/` — `stack-healthcheck.{service,timer}`, `ollama-perf-check.{service,timer}`, `notify-failure@.service`, `open-webui-backup.{service,timer}`, `renovate.timer`
* **Component data:** `/var/lib/materia/components/<component>/` — for example `searxng/settings.yml`, `searxng/limiter.toml`, `open-webui/open-webui.env`, `monitoring/ollama-gpu-check`, `monitoring/ollama-perf-check`, `monitoring/ollama-diag`, `monitoring/ollama-bench``

### 3.9 Automated GitOps Sync (`materia-sync.timer`)
A host-managed timer pulls the repository and applies changes every 15 minutes. The units live in `/etc/systemd/system/materia-sync.{service,timer}` and are not part of this repository. The service runs `materia update -c /etc/materia/config.toml`, has a 30-minute timeout and triggers `notify-failure@` when it fails.

```bash
systemctl cat materia-sync.service materia-sync.timer    # show the units
systemctl list-timers materia-sync.timer                 # next and last run
```

A push to `main` is therefore rolled out within 15 minutes without any manual step.

### 3.10 Host Tools
* **`jq`** is required by the monitoring scripts and tools: `sudo dnf install -y jq`.
* **`ollama-diag`** — diagnostics, trigger test and benchmark (see 4.7).
* **`ollama-bench`** — quick tokens/s measurement of one model: `ollama-bench <model> [num_thread] [num_ctx]`.

Both tools are versioned in `components/monitoring/` and deployed by Materia to `/var/lib/materia/components/monitoring/`. To call them by name, create these wrappers once on the host; they never need to change:

```bash
for t in ollama-diag ollama-bench; do
  printf '#!/usr/bin/bash\nexec /usr/bin/bash /var/lib/materia/components/monitoring/%s "$@"\n' "$t" | sudo tee /usr/local/bin/$t >/dev/null
  sudo chmod 0755 /usr/local/bin/$t
done
```

To change a tool, edit it in the repository and push; `materia-sync` rolls it out within 15 minutes.
---

## 4. Management & Command Cheatsheet

### 4.1 GitOps & Materia Commands

| Action | Command |
| :--- | :--- |
| **Show installed version** | `materia version` |
| **Check active Materia configuration** | `sudo materia config -c /etc/materia/config.toml` |
| **Display discovered host facts** | `sudo materia facts -c /etc/materia/config.toml` |
| **Validate the local working copy** | `cd /opt/ollama-searxng-openwebui && sudo materia validate --hostname fedora` |
| **Dry-run: preview planned changes** | `sudo materia plan -c /etc/materia/config.toml` |
| **Run without syncing (use local cache)** | `sudo materia --nosync plan -c /etc/materia/config.toml` |
| **Apply changes (sync Git, render, restart)** | `sudo materia update -c /etc/materia/config.toml` |
| **Check for corrupted components (dry-run)** | `sudo materia doctor -c /etc/materia/config.toml` |

`materia validate` takes `--hostname` and reads the repository from the current directory (`--source`). In this subcommand `-c` means `--component`, not the configuration file.

#### Standard GitOps Workflow for Changes
```bash
cd /opt/ollama-searxng-openwebui

# 1. Modify a file (for example attributes/vault.toml or a .gotmpl template)
vi attributes/vault.toml

# 2. Commit and push to GitHub
sudo git add -A
sudo git commit -m "Describe the change"
sudo git push origin main

# 3. Either wait for materia-sync (at most 15 minutes), or apply immediately
sudo materia plan -c /etc/materia/config.toml
sudo materia update -c /etc/materia/config.toml
```

---

### 4.2 Systemd & Quadlet Management
Quadlets are converted into systemd services by `/usr/lib/systemd/system-generators/podman-system-generator`. Manage the containers with `systemctl` and `journalctl`:

| Action | Command |
| :--- | :--- |
| **Status of the three main services** | `sudo systemctl status ollama.service searxng.service open-webui.service` |
| **All timers of this stack** | `systemctl list-timers 'stack-healthcheck*' 'ollama-perf-check*' 'open-webui-backup*' 'renovate*' 'materia-sync*'` |
| **Restart a specific service** | `sudo systemctl restart ollama.service` *(or `searxng`, `open-webui`)* |
| **Test Quadlet generation manually (dry-run)** | `/usr/lib/systemd/system-generators/podman-system-generator --dryrun` |
| **Reload units after a manual change** | `sudo systemctl daemon-reload` |
| **Stream live logs** | `journalctl -u ollama.service -f` *(or `open-webui.service`, `searxng.service`)* |

All three main services restart on failure with a backoff from 5 seconds up to 5 minutes, and trigger a notification when they fail.

---

### 4.3 Ollama & LLM Model Management (Inside the Container)
Ollama runs inside a container, so execute `ollama` CLI commands via `sudo podman exec -it ollama ollama <command>`.

*(Tip: add `alias ollama='sudo podman exec -it ollama ollama'` to your `~/.bashrc`.)*

| Action | Command |
| :--- | :--- |
| **List all installed models** | `sudo podman exec -it ollama ollama list` |
| **Show the loaded model and its GPU/CPU split** | `sudo podman exec -it ollama ollama ps` |
| **Download a new model** | `sudo podman exec -it ollama ollama pull gemma4:12b` |
| **Remove a model** | `sudo podman exec -it ollama ollama rm <model_name>` |
| **Benchmark one or more models** | `sudo ollama-diag bench gpt-oss:20b` |
| **Unload a model to free VRAM immediately** | `sudo podman exec -it ollama ollama stop <model_name>` |

Models up to roughly 10 GB run entirely on the GPU with the 16k context. Larger models are split between GPU and CPU and are much slower (see 1.2).

---

### 4.4 Container Image Updates (Renovate) & Disk Cleanup
Image tags are pinned in `attributes/vault.toml`. Do not pull images by hand; **Renovate** runs every 6 hours (`renovate.timer`), detects new tags and opens pull requests against this repository. A merged pull request is rolled out by `materia-sync` within 15 minutes.

| Image | Policy (`renovate.json`) |
| :--- | :--- |
| `ghcr.io/renovatebot/renovate` | Merged automatically. |
| `docker.io/searxng/searxng` | Merged automatically; pull requests are only created on Mondays between 00:00 and 06:59. |
| `docker.io/ollama/ollama` | Patch releases merged automatically (Mondays 00:00–06:59); minor and major releases need a manual review. |
| `ghcr.io/open-webui/open-webui` | Never merged automatically; review every pull request, because patch releases can include database migrations. |

```bash
sudo systemctl start renovate.service      # run Renovate now
journalctl -u renovate.service -n 50       # result of the last run
sudo podman image prune -f                 # remove dangling image layers
```

Before the first start of a new Open WebUI version, the service copies the database to `/opt/open-webui/backup/webui.db.pre-<tag>` (see 4.6).

---

### 4.5 Monitoring & Alerting
Every failing unit of this stack sends a message to the ntfy topic in `/etc/materia/notify.env` (`notify-failure@.service`). **Treat every `stack-healthcheck` notification as a real incident.**

| Check | Schedule | Fails when |
| :--- | :--- | :--- |
| `stack-healthcheck.service` | Every 15 minutes | The Ollama API, the Open WebUI `/health` endpoint or the SearXNG `/healthz` endpoint does not answer; `nvidia-smi -L` fails inside the Ollama container; a loaded model is less than `minGpuPercent` on the GPU; or Ollama logged `failed to initialize CUDA` in the last 16 minutes. |
| `ollama-perf-check.service` | Daily at 04:15 | `perfModel` generates fewer than `perfMinTps` tokens per second (measured on the second, warmed-up run). |
| `open-webui-backup.service` | Daily at 03:30 | The database backup fails. |
| `renovate.service`, `materia-sync.service` | Every 6 hours / every 15 minutes | The run fails. |

The thresholds are in `attributes/vault.toml` under `[components.monitoring]`: `minGpuPercent = "40"`, `perfModel = "Gemma3:4b"`, `perfMinTps = "30"`. `Gemma3:4b` reaches about 62 tokens/s on the GPU and about 19 on the CPU, so the threshold separates the two.

```bash
sudo systemctl start stack-healthcheck.service; journalctl -u stack-healthcheck -n 15 --no-pager
sudo systemctl start ollama-perf-check.service; journalctl -u ollama-perf-check -n 3 --no-pager
```

After a GPU incident the CUDA check keeps failing for up to 16 minutes, so one more notification after the fix is expected.

---

### 4.6 Backups
* **Before each new image version:** `open-webui.service` copies `webui.db` (and its `-wal` and `-shm` files) to `/opt/open-webui/backup/*.pre-<tag>` before the container starts. Existing copies are never overwritten, and a failed copy blocks the start.
* **Daily:** `open-webui-backup.timer` writes a consistent online copy to `/opt/open-webui/backup/webui-<date>.db` at 03:30 and deletes copies older than 14 days.
* **Not covered:** `vector_db/` and `uploads/` in `/opt/open-webui`, and the Ollama models. Use a host backup for `/opt/open-webui`; models can be pulled again.

---

### 4.7 GPU & System Diagnostics

```bash
# 1. Full status: versions, devices, CDI spec, journal and a real GPU load test
sudo ollama-diag status

# 2. Tokens/s and GPU share for all local models, or for the models you name
sudo ollama-diag bench
sudo ollama-diag bench gpt-oss:20b qwen3:30b-a3b

# 3. Check that every model has readable metadata
sudo ollama-diag models

# 4. Find out what removes GPU access (restarts Ollama several times)
sudo ollama-diag triggers

# 5. Monitor real-time GPU utilization, power draw (70 W cap), clock speed and PCIe traffic
nvidia-smi dmon -s pucvmet -d 1

# 6. Check Ollama logs for GPU detection and CUDA errors
journalctl -u ollama.service -b --no-pager | grep -E "inference compute|gpu memory|failed to initialize CUDA"

# 7. Verify SearXNG JSON output on port 8112
curl -s "http://127.0.0.1:8112/search?q=fedora+linux&format=json" | jq '.results | length'
```

A healthy system shows `100% GPU` in `ollama ps` for `Gemma3:4b` and no `failed to initialize CUDA` lines.

---

### 4.8 Known Pitfall: `podman update` Removes GPU Access
Never run `podman update` (for example with `--cpuset-cpus`) against the running Ollama container. The container then loses access to the NVIDIA devices:

* `nvidia-smi` inside the container reports `Failed to initialize NVML: Unknown Error`.
* Ollama logs `ggml_cuda_init: failed to initialize CUDA: no CUDA-capable device is detected`.
* `ollama ps` shows `100% CPU` for every model.

Recovery: `sudo systemctl restart ollama.service`.

Change the CPU pinning only through `cpuPinList` in `attributes/vault.toml`, followed by a deploy. `systemctl daemon-reload`, the health checks and Materia runs do not affect GPU access.

---

### 4.9 Outside This Repository: `genai-proxy`
Open WebUI uses an OpenAI-compatible endpoint at `http://127.0.0.1:8111/v1` (`OPENAI_API_BASE_URL` in `open-webui.env.gotmpl`). That endpoint is the locally built container `localhost/genai-proxy:latest`, started by the host unit `/etc/systemd/system/genai-proxy.service`. It is **not** managed by Materia, not updated by Renovate and not covered by the health checks.
