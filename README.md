# Qwen Code stack (lite)

Deploys vLLM, Open WebUI, Prometheus/Grafana and an Nginx HTTPS reverse proxy (no LiteLLM, no Keycloak). Also updates /etc/hosts on the controller.

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

