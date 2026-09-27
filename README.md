# Qwen Code stack (lite)

Deploys vLLM, Open WebUI, Prometheus/Grafana and an Nginx HTTPS reverse proxy (no LiteLLM, no Keycloak). Also updates /etc/hosts on the controller.

## Infrastructure

```
 Clients (browser, Qwen Code)
        │  HTTPS (self-signed certificate) — only ports 80/443 are open
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│ Host: Debian 13 + NVIDIA GPU             (compose project: qwen-stack)│
│                                                                       │
│   nginx   :80 → redirect to HTTPS   |   :443 TLS termination          │
│     │                                                                 │
│     ├── app.qovg.local     ──►  open-webui :8080                      │
│     │                                │  OpenAI-compatible API         │
│     │                                ▼                                │
│     ├── api.qovg.local     ──►  vllm :8000  ◄──  GPU                  │
│     │   (used by Qwen Code)          ▲           (NVIDIA Container    │
│     │                                │            Toolkit)            │
│     │                                │  scrapes /metrics              │
│     │                           prometheus :9090                      │
│     │                                ▲  queries                       │
│     └── grafana.qovg.local ──►  grafana :3000                         │
│                                                                       │
│   Internal Docker network "qwen-stack_default": services reach each   │
│   other by name. Only nginx publishes ports on the host.              │
└───────────────────────────────────────────────────────────────────────┘
```

| Service    | Internal address  | Exposed as                      | Role                                   |
|------------|-------------------|---------------------------------|----------------------------------------|
| nginx      | —                 | host ports 80/443               | TLS reverse proxy, HTTP → HTTPS        |
| open-webui | `open-webui:8080` | `https://app.qovg.local`        | Chat interface                         |
| vllm       | `vllm:8000`       | `https://api.qovg.local/v1`     | Model serving (GPU), used by Qwen Code |
| prometheus | `prometheus:9090` | not exposed (no authentication) | Collects vLLM metrics                  |
| grafana    | `grafana:3000`    | `https://grafana.qovg.local`    | Dashboards                             |

Host paths used by the stack:

| Path                         | Content                                |
|------------------------------|----------------------------------------|
| `/opt/qwen-stack/`           | `docker-compose.yml`                   |
| `/opt/nginx/conf.d/`         | Nginx configuration                    |
| `/opt/nginx/ssl-selfsigned/` | Self-signed certificate and key        |
| `/opt/monitoring/`           | `prometheus.yml`                       |
| `/opt/vllm-tool-parser/`     | vLLM chat template (tool calling)      |
| `/srv/models/huggingface/`   | Downloaded models (Hugging Face cache) |

## Inspecting the stack with Docker

Run the `docker compose` commands from the compose directory:

```bash
cd /opt/qwen-stack
```

Overview:

```bash
docker compose ps                                                # services and their state
docker stats --no-stream                                         # CPU / RAM per container
docker compose config                                            # final rendered configuration
```

Logs:

```bash
docker compose logs -f vllm            # model loading, requests, errors
docker compose logs -f open-webui
docker compose logs -f nginx
docker compose logs --tail 100 grafana
```

GPU and model:

```bash
nvidia-smi                                                       # GPU seen by the host
docker compose exec vllm nvidia-smi                              # GPU seen by the vLLM container
docker compose exec vllm curl -s localhost:8000/v1/models        # model served by vLLM
docker compose exec vllm curl -s localhost:8000/metrics | head   # metrics scraped by Prometheus
```

Network, volumes and disk usage:

Operations:

```bash
docker compose exec -T nginx nginx -t              # test the Nginx configuration
docker compose restart nginx                       # apply an Nginx configuration change
docker compose up -d --force-recreate open-webui   # recreate a single service
docker compose pull && docker compose up -d        # update the images
```

From another machine on the network, check that only ports 80/443 answer:

```bash
nmap -p- <server-ip>
```

## Grafana: Prometheus data source and vLLM dashboard

Grafana starts empty: add Prometheus as a data source once, then import a
dashboard. Both steps are done in the Grafana web interface.

### 1. Log in

Open `https://grafana.qovg.local` (accept the self-signed certificate warning)
and log in with:

- user: `admin`
- password: the value of `grafana_admin_password` from your vault

### 2. Add the Prometheus data source

1. Menu **Connections** → **Data sources** → **Add new data source** → **Prometheus**.
2. **Name**: `prometheus` (any name works; you select it again at import time).
3. **Prometheus server URL**: `http://prometheus:9090`
4. Leave the other settings unchanged, then click **Save & test**. A green
   message confirms that Grafana can query Prometheus.

Use the Docker service name `prometheus`, **not** `localhost` nor the server IP:
Grafana and Prometheus run in separate containers on the same internal Docker
network, and Prometheus does not publish any port on the host.

### 3. Import a vLLM dashboard

Download the file here : https://grafana.com/grafana/dashboards/24756-vllm-monitoring-v2/

Then in Grafana:

1. Menu **Dashboards** → **New** → **Import**.
2. **Upload dashboard**
3. In the data source field, select the Prometheus data source created above.
4. Click **Import**.

## Disk partitioning (1 TB SSD)

| Volume    | Mount point   | Suggested size            | Content                                             |
|-----------|---------------|---------------------------|-----------------------------------------------------|
| ESP (EFI) | `/boot/efi`   | 1 GB, vfat                | UEFI boot                                           |
| `/boot`   | `/boot`       | 2 GB, ext4                | Kernels (outside LVM)                               |
| swap      | —             | 32 GB                     | Memory safety margin (adjust to the installed RAM)  |
| `home`    | `/`           | 100 GB, ext4              | System, APT packages                                |
| `root`    | `/`           | 100 GB, ext4              | System, APT packages                                |
| `var`     | `/var`        | 100 GB, ext4              | Logs, Docker images and volumes (`/var/lib/docker`) |
| `data`    | `/srv/models` | remaining (~590 GB), ext4 | Hugging Face cache, Qwen model weights              |

Docker images are large (vLLM alone is about 30 GB, Open WebUI about 7 GB).
Keep an eye on `/var` with `df -h /var` and `docker system df`, and grow the
volume when needed (`-r` also resizes the filesystem):

## Layout

```
.
├── ansible.cfg
├── 01_system_prep.yml      # system_prep + nvidia_nouveau_blacklist   (may reboot)
├── 02_nvidia_driver.yml    # nvidia_driver                            (may reboot)
├── 03_stack.yml            # Docker, toolkit and the application stack
├── requirements.yml
├── inventory/
│   ├── hosts.ini
│   └── group_vars/qwen_servers/
│       ├── main.yml              # shared variables
│       ├── vault.yml.example     # example of the secrets file
│       └── vault.yml             # YOUR encrypted secrets (not versioned)
└── roles/
    ├── system_prep/
    ├── nvidia_nouveau_blacklist/
    ├── nvidia_driver/
    ├── docker_engine/
    ├── nvidia_container_toolkit/
    ├── app_directories/
    ├── vllm_tool_parser/
    ├── tls_selfsigned/
    ├── stack_config/
    ├── stack_deploy/
    ├── local_hosts/
```

Every role follows the same structure: `defaults/`, `files/`, `handlers/`,
`meta/`, `tasks/`, `templates/`, `vars/`.

## Why three playbooks?

Installing the NVIDIA driver needs up to two reboots (unload `nouveau`, then
load the new driver). Each reboot ends the current playbook, and you simply run
the next one once the machine is back:

```bash
ansible-playbook 01_system_prep.yml   --ask-vault-pass
# (reboot if needed)
ansible-playbook 02_nvidia_driver.yml --ask-vault-pass
# (reboot if needed)
ansible-playbook 03_stack.yml         --ask-vault-pass
```

If no reboot is needed (driver already working), just run the three playbooks
one after the other. Every playbook is idempotent and can be replayed safely.

### Reboot behaviour

- **Target = the machine running Ansible** (`localhost`, or the controller
  reached through SSH): `ansible.builtin.reboot` cannot be used, because it
  would kill Ansible itself. The role schedules a reboot in
  `qwen_reboot_delay_seconds` seconds (`systemd-run ... systemctl reboot`),
  prints the next playbook to run, and ends the play cleanly.
- **Remote target**: `ansible.builtin.reboot` reboots and waits for the host.

## Prerequisites

On the controller:

```bash
sudo apt install -y python3 python3-pip python3-venv
python3 -m venv ~/.venv-ansible
source ~/.venv-ansible/bin/activate
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

On the target: Debian 13, Python 3, internet access. For a remote target, a
user with sudo rights and an SSH key.

## NVIDIA driver installer (.run)

The roles never download the driver. Drop the NVIDIA `.run` installer in
`/tmp` (`qwen_nvidia_run_source_dir`) before running `01_system_prep.yml`.

- The roles look for `*.run` files whose name contains `nvidia`
  (case-insensitive) and pick **the most recent one**.
- Debian 13 mounts `/tmp` as tmpfs, **wiped on every reboot**:
  `nvidia_nouveau_blacklist` copies the installer to `/var/tmp`
  (`qwen_nvidia_run_staging_dir`) before rebooting, and `nvidia_driver`
  looks in both directories.
- The installer is made executable (`mode: 0755`), run with
  `--dkms --silent --no-questions`, then `ldconfig` is run.
- Set `qwen_manage_nvidia_driver: false` to never touch the driver.

## Secrets (ansible-vault)

The secrets file must live **next to the inventory**, so that both
`ansible-playbook` and ad-hoc `ansible` commands load it:

```bash
ansible-vault create inventory/group_vars/qwen_servers/vault.yml
```

See `vault.yml.example` for the expected keys. `stack_config` refuses to run
while a required secret still has its `CHANGEME` default.

Check that the vault is picked up:

```bash
ansible qwen_servers -m debug -a "var=grafana_admin_password" --ask-vault-pass
```

To avoid typing the password every time, store it outside the repository
(`~/.vault_pass.txt`, `chmod 600`) and uncomment `vault_password_file` in
`ansible.cfg`.

## After deployment

- `https://app.qovg.local` (Open WebUI)
- `https://api.qovg.local/v1` (API used by Qwen Code)
- `https://grafana.qovg.local` (Grafana)

The certificate is self-signed: browsers show a warning (expected). To use an
internal company PKI instead, see the comments in
`roles/tls_selfsigned/tasks/main.yml`.

`local_hosts` adds the subdomains to `/etc/hosts` on the controller, so no IP
needs to be typed in the browser.

## Example
<img width="2558" height="1362" alt="image" src="https://github.com/user-attachments/assets/29c2959c-2314-4ac9-8da9-44fe1fa6aac4" />

