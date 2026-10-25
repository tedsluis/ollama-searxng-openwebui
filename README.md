# Local AI Stack GitOps: Ollama + SearXNG + Open WebUI (Podman Quadlets & Materia)

This repository contains the complete declarative GitOps configuration for a local AI platform on **Fedora Linux**, consisting of:
1. **Ollama** (`docker.io/ollama/ollama:latest`) — LLM Inference Server with NVIDIA GPU acceleration (CDI) and hybrid CPU pinning.
2. **SearXNG** (`docker.io/searxng/searxng:latest`) — Privacy-friendly metasearch engine (with JSON output enabled) for live web search and RAG.
3. **Open WebUI** (`ghcr.io/open-webui/open-webui:main`) — Web interface connected to Ollama, SearXNG, and an optional OpenAI-compatible proxy (`genai-proxy`).

The stack is deployed as **rootful Podman Quadlets** (systemd-managed containers) and orchestrated via **[Prima Materia (`materia`)](https://github.com/stryan/materia)**.

---

## 1. System Architecture & Hardware Profile

This configuration is specifically tuned for the following host specifications:

| Component | Specification | Configuration / Optimization |
| :--- | :--- | :--- |
| **OS** | Fedora 45 Linux | SELinux active (`enforcing`), Systemd, Podman v5.4+ |
| **CPU** | Intel Core i5 (14th Gen) | 20 logical threads split across **14 physical cores**:<br>• **6 P-cores** (CPUs `0-11`, up to 5000 MHz)<br>• **8 E-cores** (CPUs `12-19`, up to 3700 MHz)<br>• Energy profile: `tuned` set to `throughput-performance` |
| **Memory & Storage** | 64 GB DDR5 RAM & 1 TB NVMe SSD | Sufficient RAM for hybrid GPU/CPU offloading of large MoE models (20B–35B). |
| **GPU** | NVIDIA RTX A2000 12GB[cite: 1, 2] | • **12,282 MB VRAM** (192-bit bus, 3,328 CUDA cores, VBIOS `94.06.32.00.22`)[cite: 1]<br>• **PCIe Gen4 x16** (16.0 GT/s, Bus ID `PCI:1:0:0`)[cite: 1]<br>• **70W Power Cap** (via PCIe slot)<br>• **ECC Disabled** (full 12 GB VRAM usable)<br>• **Driver:** `615.78.08` (NVML `13.615.78.08`, CUDA `13.4`)[cite: 2] |

### Key Performance Optimizations
* **CPU Pinning (`--cpuset-cpus=0,2,4,6,8,10,12,13,14,15,16,17,18,19`):** Binds the Ollama container to exactly one thread per physical core (6 P-cores without Hyper-Threading + 8 E-cores = 14 cores). This prevents Hyper-Threading contention and maximizes DDR5 memory bandwidth for models partially running in system RAM.
* **Maximum VRAM Allocation (`OLLAMA_GPU_OVERHEAD=134217728`):** Restricts the reserved VRAM safety margin to 128 MB so that nearly the entire 12,282 MB is available for model layers and KV-cache[cite: 1].
* **Single-User Focus (`OLLAMA_NUM_PARALLEL=1` & `OLLAMA_MAX_LOADED_MODELS=1`):** Prevents Ollama from pre-allocating context memory 4x over for concurrent requests.
* **Flash Attention (`OLLAMA_FLASH_ATTENTION=1`):** Accelerates attention computations and reduces context VRAM consumption.
* **Host Networking (`Network=host`):** Both `ollama` and `open-webui` run directly on the host network stack (`127.0.0.1`), eliminating container NAT overhead (`pasta`/`slirp4netns`) during token streaming.

---

## 2. Repository Structure

```text
ollama-searxng-openwebui/
├── MANIFEST.toml                             # Links components to hostname 'fedora'
├── README.md                                 # This documentation
├── attributes/
│   └── vault.toml                            # Centralized variables (ports, paths, CPU pinning, tags)
└── components/
    ├── ollama/
    │   ├── MANIFEST.toml                     # Component definition & service binding
    │   └── ollama.container.gotmpl           # Podman Quadlet template for Ollama (with GPU CDI)
    ├── searxng/
    │   ├── MANIFEST.toml                     # Automatically restarts searxng.service when settings.yml changes
    │   ├── searxng.container.gotmpl          # Podman Quadlet template for SearXNG (port 8112)
    │   └── settings.yml                      # SearXNG configuration with HTML + JSON output enabled
    └── open-webui/
        ├── MANIFEST.toml                     # Restarts open-webui.service when open-webui.env changes
        ├── open-webui.container.gotmpl       # Podman Quadlet template for Open WebUI (port 8080)
        └── open-webui.env.gotmpl             # Environment variables template for Open WebUI
```

---

## 3. Host Configuration (Outside this Git Repo)

To deploy this Git repository on a Fedora system, several components must be configured directly on the host OS.

### 3.1 Host Performance & NVIDIA GPU Persistence
```bash
# 1. Set the Fedora energy profile to maximum throughput (prevents 800 MHz idle drops on P-cores)
sudo tuned-adm profile throughput-performance

# 2. Keep the NVIDIA driver active (eliminates 2–4s startup delays between model loads)
sudo systemctl enable --now nvidia-persistenced
```

### 3.2 NVIDIA Container Toolkit & CDI (Container Device Interface)
Podman uses CDI (`/etc/cdi/nvidia.yaml`) to pass the RTX A2000 (`nvidia.com/gpu=all`) to the Ollama container[cite: 1]:
```bash
# 1. Install the toolkit
sudo dnf install -y nvidia-container-toolkit

# 2. Generate the CDI specification
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# 3. Verify that the GPU is recognized (shows [nvidia.com/gpu=0](https://nvidia.com/gpu=0) and [nvidia.com/gpu=all](https://nvidia.com/gpu=all))
nvidia-ctk cdi list

# Note: Re-run 'sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml' after every NVIDIA driver update!
```

### 3.3 Podman API Socket
Materia communicates with Podman via the systemd socket (`/run/podman/podman.sock`):
```bash
sudo systemctl enable --now podman.socket
```

### 3.4 Persistent Data Directories & SELinux Context
Actual AI models and Open WebUI user data reside **outside** Materia on the host disk to survive container updates:
* **Ollama Models (`/usr/share/ollama/.ollama`):** Mounted to `/root/.ollama:Z` inside the `ollama` container.
* **Open WebUI Data (`/opt/open-webui`):** Mounted to `/app/backend/data:Z` inside the `open-webui` container.

Ensure SELinux allows container access to these directories:
```bash
sudo chcon -Rt container_file_t /usr/share/ollama/.ollama
sudo chcon -Rt container_file_t /opt/open-webui
```

### 3.5 Installing Materia (`v0.8.0+`)
The Materia binary and man pages are installed under `/usr/local`:
```bash
wget [https://github.com/stryan/materia/releases/download/v0.8.0/materia-v0.8.0-amd64.tar.gz](https://github.com/stryan/materia/releases/download/v0.8.0/materia-v0.8.0-amd64.tar.gz) -O /tmp/materia.tar.gz
tar -xvzf /tmp/materia.tar.gz -C /tmp
sudo install -m 0755 /tmp/bin/materia /usr/local/bin/materia
sudo cp -r /tmp/man/* /usr/local/share/man/ && sudo mandb
sudo mkdir -p /etc/materia /var/lib/materia
```

### 3.6 Materia Configuration File (`/etc/materia/config.toml`)
This file points Materia to your GitHub repository, specifies the SSH key, and sets a higher systemd timeout for large container pulls:
```toml
attributes = "file"

[source]
kind = "git"
url = "git@github.com:tedsluis/ollama-searxng-openwebui.git"
branch = "main"
private_key = "/root/.ssh/id_rsa"

[file]
base_dir = "attributes"

[services]
timeout = 600
```

### 3.7 Host Paths Generated by Materia
When `materia update` runs, Materia translates the `.gotmpl` templates into the following host paths:
* **Local Git Cache:** `/var/lib/materia/source/`
* **Generated Podman Quadlets:**
  * `/etc/containers/systemd/ollama/ollama.container`
  * `/etc/containers/systemd/searxng/searxng.container`
  * `/etc/containers/systemd/open-webui/open-webui.container`
* **Generated Component Data & Configs:**
  * `/var/lib/materia/components/ollama/MANIFEST.toml`
  * `/var/lib/materia/components/searxng/settings.yml`
  * `/var/lib/materia/components/open-webui/open-webui.env`

### 3.8 Optional: Automated GitOps Sync Timer (`/etc/systemd/system/materia-sync.timer`)
For fully automated pull-based GitOps updates, activate this systemd timer:
* **`/etc/systemd/system/materia-sync.service`**:
  ```ini
  [Unit]
  Description=Materia GitOps Sync
  After=network-online.target podman.socket
  Wants=network-online.target

  [Service]
  Type=oneshot
  ExecStart=/usr/local/bin/materia update -c /etc/materia/config.toml
  ```
* **`/etc/systemd/system/materia-sync.timer`**:
  ```ini
  [Unit]
  Description=Run Materia GitOps Sync every 15 minutes

  [Timer]
  OnBootSec=2min
  OnUnitActiveSec=15min
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```

---

## 4. Management & Command Cheatsheet

### 4.1 GitOps & Materia Commands
Use these commands to validate, plan, and roll out changes from your Git repository:

| Action | Command |
| :--- | :--- |
| **Check active Materia configuration** | `sudo materia config -c /etc/materia/config.toml` |
| **Display discovered host facts** | `sudo materia facts -c /etc/materia/config.toml` |
| **Validate repository & templates** | `sudo materia validate -c /etc/materia/config.toml` |
| **Dry-run: Preview planned changes** | `sudo materia plan -c /etc/materia/config.toml` |
| **Run without syncing (use local cache)** | `sudo materia --nosync plan -c /etc/materia/config.toml` |
| **Apply changes (Sync Git + Render Quadlets + Restart)** | `sudo materia update -c /etc/materia/config.toml` |
| **Check for corrupted components (Dry-run)** | `sudo materia doctor -c /etc/materia/config.toml` |

#### Standard GitOps Workflow for Changes:
```bash
cd /opt/ollama-searxng-openwebui

# 1. Modify a file (e.g. attributes/vault.toml or a .gotmpl template)
vi attributes/vault.toml

# 2. Commit and push to GitHub
sudo git add .
sudo git commit -m "Update configuration"
sudo git push origin main

# 3. Plan and apply via Materia
sudo materia plan -c /etc/materia/config.toml
sudo materia update -c /etc/materia/config.toml
```

---

### 4.2 Systemd & Quadlet Management
Because Quadlets are dynamically converted into systemd services via `/usr/lib/systemd/system-generators/podman-system-generator`, manage running containers directly using `systemctl` and `journalctl`:

| Action | Command |
| :--- | :--- |
| **Check status of all 3 services** | `sudo systemctl status ollama.service searxng.service open-webui.service` |
| **Restart a specific service** | `sudo systemctl restart ollama.service` *(or `searxng`, `open-webui`)* |
| **Test Quadlet generation manually (Dry-run)** | `/usr/lib/systemd/system-generators/podman-system-generator --dryrun` |
| **Reload Quadlets after manual inspection** | `sudo systemctl daemon-reload` |
| **Stream live Ollama logs** | `journalctl -u ollama.service -f` |
| **Stream live Open WebUI logs** | `journalctl -u open-webui.service -f` |
| **Stream live SearXNG logs** | `journalctl -u searxng.service -f` |

---

### 4.3 Ollama & LLM Model Management (Inside the Container)
Since Ollama runs inside a container, execute `ollama` CLI commands via `sudo podman exec -it ollama ollama <command>`.

*(Tip: Add `alias ollama='sudo podman exec -it ollama ollama'` to your `~/.bashrc` to run commands directly).*

| Action | Command |
| :--- | :--- |
| **List all installed models** | `sudo podman exec -it ollama ollama list` |
| **Show currently loaded model & GPU/CPU split** | `sudo podman exec -it ollama ollama ps` |
| **Download a new model** | `sudo podman exec -it ollama ollama pull gemma4:12b` |
| **Remove a model** | `sudo podman exec -it ollama ollama rm <model_name>` |
| **Run a CLI benchmark** | `sudo podman exec -it ollama ollama run gpt-oss:20b "Explain how Linux containers work in 100 words." --verbose`[cite: 3] |
| **Unload a model to free VRAM immediately** | `sudo podman exec -it ollama ollama stop <model_name>` |

---

### 4.4 Container Updates & Disk Cleanup (Podman)
To update to newer container image versions:

```bash
# 1. Pull the latest container images
sudo podman pull docker.io/ollama/ollama:latest
sudo podman pull docker.io/searxng/searxng:latest
sudo podman pull ghcr.io/open-webui/open-webui:main

# 2. Restart services to apply the new images
sudo systemctl restart ollama.service searxng.service open-webui.service

# 3. Clean up dangling (<none>) image layers to free up NVMe space
sudo podman image prune -f
```

---

### 4.5 GPU & System Diagnostics
Verify whether the RTX A2000 (12 GB VRAM)[cite: 1, 2] and CPU cores perform optimally under load:

```bash
# 1. Monitor real-time GPU utilization, power draw (70W cap), clock speed, and PCIe traffic
nvidia-smi dmon -s pucvmet -d 1

# 2. Check Ollama logs for layer offloading and VRAM assignments
journalctl -u ollama.service -n 100 --no-pager | grep -E "inference compute|offload| memory|flash|print_timing"

# 3. Verify SearXNG JSON output on port 8112
curl -s "[http://127.0.0.1:8112/search?q=fedora+linux&format=json](http://127.0.0.1:8112/search?q=fedora+linux&format=json)" | head -c 500
```
