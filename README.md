# Homelab Infrastructure: Tailscale & Dockhand

This repository documents the homelab architecture leveraging **Tailscale** for secure mesh networking and **Dockhand** on Docker for containerized development, data science, and storage management.

---

## 1. Tailscale Mesh Network Architecture

The homelab relies on Tailscale for encrypted, zero-trust remote access across bare-metal hosts and virtual machines without opening firewall ports.

| Node | OS / Platform | Tailscale IP | Role & Capabilities |
| :--- | :--- | :--- | :--- |
| **`pve`** | Linux 7.0.2-6-pve | `100.78.109.113` | **Proxmox VE Hypervisor** with **Subnet Routing** enabled. Advertises local physical and internal virtual subnets, enabling seamless remote access to all VMs and hypervisor consoles. |
| **`mini24`** | macOS 27.0.1 | `100.102.11.66` | **Workstation & Network Gateway**. Configured with **Subnets**, **Exit Node**, and **Peer Relay** for failover traffic routing and relaying. |
| **`dockhand`** | Linux 6.12.111 (Debian 13) | `100.113.128.113` | **Dedicated Docker VM** running on Proxmox (4 cores, 13.7 GB RAM) hosting the Dockhand container fleet. |
| **`homeassistant`** | Linux 6.18.52-haos | `100.126.185.111` | **Home Assistant OS (HAOS)** node for home automation, IoT control, and monitoring. |

### Subnet Routing Highlights
- **Proxmox VE (`pve`)**: By advertising subnet routes (`tailscale up --advertise-routes=...`), remote devices on the tailnet can access Proxmox-managed bridge networks, internal VM IPs, and local LAN devices without needing individual Tailscale clients on every VM.
- **Mac Gateway (`mini24`)**: Acts as a backup subnet router and exit node, allowing encrypted routing of all internet traffic when working remotely.

---

## 2. Reverse Proxy & Unified Tailnet Access

Using Tailscale Serve alongside [Caddyfile](file:///Users/sebh/GitHub/selfhost/caddy/Caddyfile), all containerized web applications are exposed under a single secure MagicDNS domain:  
**`https://dockhand.tailnet.ts.net/`**

### Route Mapping via Caddy
Caddy listens on internal port `:8081` and reverse-proxies subpaths to individual container ports:

```
https://dockhand.tailnet.ts.net/
  ├── /             --> 127.0.0.1:3000 (Dockhand Container Management GUI)
  ├── /code/        --> 127.0.0.1:8080 (VS Code Server)
  ├── /jupyter/     --> 127.0.0.1:8888 (JupyterLab)
  ├── /scipy/       --> 127.0.0.1:8889 (SciPy Notebook)
  └── /fb/          --> 127.0.0.1:8900 (Filebrowser Quantum)
```

### Key Caddyfile Directives
- **Subpath Normalization**: Automatic 308 redirects from `/app` to `/app/` (e.g. `redir /code /code/ 308`) ensure browser asset references load properly.
- **Path Stripping**: `handle_path /code/*` strips the `/code` prefix before forwarding to VS Code Server, which expects root-relative requests.
- **Header Propagation**: Proxying to Filebrowser passes `Host`, `X-Forwarded-For`, and `X-Forwarded-Proto https` headers for secure cookie and session handling behind reverse proxies.
- **Default Fallback**: An unpathed `handle` block directs all root traffic directly to the Dockhand GUI.

---

## 3. Dockhand Container Fleet

Managed via Dockhand (`fnsys/dockhand`) running on Docker 29.9.1:

| Container | Image | Port | Description |
| :--- | :--- | :--- | :--- |
| **`dockhand`** | `fnsys/dockhand` | `3000` | Web UI for Docker container lifecycle management, stack orchestration, container logs, terminal shell, and resource metrics. |
| **`code-server`** | `codercom/code-server` | `8080` | Browser-based VS Code IDE allowing full remote development and terminal access directly on the host VM filesystem. |
| **`jupyter`** | `jupyter/base-notebook` | `8888` | Lightweight JupyterLab environment for rapid Python scripting and notebook workflows. |
| **`scipy-notebook`** | `quay.io/jupyter/scipy-notebook` | `8889` | Full scientific Python notebook environment bundled with NumPy, SciPy, Pandas, Matplotlib, and SymPy. |
| **`marimo`** | `ghcr.io/marimo-team/marimo` | — | Reactive, reproducible Python notebook and application runner where code changes propagate deterministically without hidden state. |
| **`filebrowser`** | `ghcr.io/gtsteffaniak/filebrowser:latest` | `8900` | **Filebrowser Quantum** multi-source file manager for web-based access to container filesystems via bind mounts. |

---

## 4. Shared Storage & File Access (Filebrowser Quantum)

To avoid container filesystem isolation and eliminate the need for manual `docker cp` or SSH commands, **Filebrowser Quantum** provides a unified web GUI for shared storage.

Refer to [filebrowser.md](file:///Users/sebh/GitHub/selfhost/filebrowser/filebrowser.md) for full configuration details.

### Bind Mount Architecture
Common directories on the host (e.g., `./files`, `./data`, workspace roots) are bind-mounted across both Filebrowser Quantum and computational containers (`code-server`, `jupyter`, `scipy-notebook`, `marimo`).

```yaml
services:
  filebrowser:
    image: ghcr.io/gtsteffaniak/filebrowser:latest
    container_name: quantum-prod
    user: "1000:1000"
    ports:
      - 8900:80
    restart: unless-stopped
    volumes:
      - ./data:/home/filebrowser/data
      - ./files:/files
      # Shared data volumes mounted across notebooks & IDEs:
      # - /srv/workspaces:/files/workspaces
    environment:
      - "FILEBROWSER_CONFIG=data/config.yaml"
```

### Multi-Source Configuration Primer
In `data/config.yaml`:
- **`baseURL: "/fb"`**: Matches the Caddy reverse proxy path.
- **`trustProxyHeaders: true`**: Respects HTTPS forwarding from Caddy/Tailscale.
- **`sources`**: Exposes `/files` as the primary user root (`Home`) with automatic per-user directory scoping (`/users/<username>`), and exposes `/home/filebrowser` as `Backend` for administrative configuration.
- Any file uploaded via Filebrowser Quantum is instantly available inside VS Code Server, JupyterLab, and Marimo notebooks.
