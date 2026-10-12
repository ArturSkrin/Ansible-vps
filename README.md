# Ansible VPS

Playbooks I use to bring a fresh Ubuntu VPS to a production-ready state and keep a fleet of servers consistent and secure.
A new server goes from first login to ready-to-deploy in about 5 minutes, down from about an hour of manual setup.

## Playbooks

| Playbook | What it does |
|---|---|
| `setup.yml` | Bootstraps a fresh server: `deploy` user with sudo, login keys, Docker Engine + Compose plugin, SSH hardening |
| `keys.yml` | Syncs login and sudo keys from Git to every server; removing a key from the file revokes access everywhere in one run |
| `harden.yml` | SSH hardening with verification, sudo via SSH agent keys, password rotation |
| `ssh_audit.yml` | Read-only audit of the *effective* sshd config across the fleet |
| `k3s.yml` | Single-node Kubernetes (k3s) with Argo CD: pinned versions, nftables host firewall, no public ports except SSH |

## How access works

Keys are kept per server group in `files/<keys_dir>/`, and each inventory group points to its set via `group_vars`:

| Key is in | Can log in | sudo |
|---|---|---|
| `authorized_keys` only | yes | password |
| `authorized_keys` + `sudo_authorized_keys` | yes | no password (SSH agent) |

So a CI deploy key can log in but cannot become root, while admins get passwordless sudo through their forwarded SSH agent (`pam_ssh_agent_auth`).
The sudo key file is owned by root, so a user cannot add keys to escalate. The password stays as a fallback for the provider's web console.

## k3s server

`playbooks/k3s.yml` turns a server into a single-node k3s host with Argo CD in one run.

| Role | What it does |
|---|---|
| `k3s` | nftables host firewall, k3s binary pinned and verified by sha256, own systemd unit and config, Traefik as ClusterIP |
| `argocd` | Argo CD from the official Helm chart, installed by the k3s built-in helm-controller (no helm binary needed) |

Design choices:

- **No public ports except SSH.** ServiceLB is disabled and Traefik is ClusterIP.
  The Kubernetes API and the Argo CD UI are reached through SSH tunnels only.
- **Everything pinned.** k3s and Argo CD chart versions are role variables; an upgrade is a one-line change and a re-run.
- **Rebuildable.** Add a new host to the `k3s_servers` group and the same playbook rebuilds it.
- **Works next to Docker.** Rolled out on a server with a live Docker Compose stack, which kept serving.

```bash
ansible-playbook playbooks/k3s.yml --limit <host>

# Cluster access from the control node (nothing exposed to the internet)
ssh <host> 'sudo cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/<host>.yaml
ssh -fN -L 6443:127.0.0.1:6443 <host>
KUBECONFIG=~/.kube/<host>.yaml kubectl get nodes

# Argo CD UI: run on the server, then open https://localhost:8080 through an SSH tunnel
sudo kubectl -n argocd port-forward svc/argocd-server 8080:443
ssh -N -L 8080:127.0.0.1:8080 <host>
```

## Problems found and fixed on a real fleet

**Password login left enabled by cloud-init.** On Ubuntu 22.04+ `sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`, and sshd uses the first value it finds.
Cloud images often ship `50-cloud-init.conf` with `PasswordAuthentication yes`, which silently overrides edits to the main `sshd_config`.
The audit found two servers where password login was still on. Hardening now goes into `00-hardening.conf` (read first), is validated with `sshd -t` before restart, and is checked with `sshd -T` after.

**Key-based sudo failing only for Ansible on Ubuntu 24.04.** Ansible runs `sudo -n` (non-interactive). Since sudo 1.9.10, `-n` skips authentication unless `noninteractive_auth` is set, so key auth worked by hand but not from Ansible.
`harden.yml` reads the sudo version and adds `Defaults noninteractive_auth` only where it is supported; older sudo would reject the option.

**Restarting the host firewall would wipe Docker and Kubernetes networking.** Ubuntu's `nftables.service` runs `nft flush ruleset` on stop, and the default `/etc/nftables.conf` starts with the same command.
Docker and k3s keep their rules in the same kernel ruleset (through iptables-nft), so a plain firewall restart would cut every container off the network.
The `k3s` role keeps host rules in a dedicated table (`inet host_firewall`), reloads only that table atomically, and overrides `ExecStop` to delete only it.
Verified on a live server: after `systemctl restart nftables` the Docker NAT rules were identical and the site kept serving.

## Usage

```bash
cp inventory/hosts.ini.example inventory/hosts.ini       # your servers and groups
cp inventory/group_vars/example.yml inventory/group_vars/<group>.yml
mkdir files/<group> && cp files/example/*.example files/<group>/   # then drop the .example suffix and add keys

# New server (as root or the provider's initial user; add -k if it only has a password)
ansible-playbook playbooks/setup.yml  --limit <host> -e ansible_user=root
ansible-playbook playbooks/keys.yml   --limit <host> -K
ansible-playbook playbooks/harden.yml --limit <host> -K

# Optional: make it a k3s server (host must be in the k3s_servers group)
ansible-playbook playbooks/k3s.yml --limit <host>

# Day to day
ansible-playbook playbooks/keys.yml --check --diff     # preview key changes
ansible-playbook playbooks/keys.yml                    # apply to the whole fleet
ansible-playbook playbooks/ssh_audit.yml               # audit, no sudo password needed
```

Requirements: Ansible 2.15+, `passlib` on the control node, Ubuntu 22.04/24.04 targets, an SSH agent with your key loaded (`ForwardAgent` is set in `ansible.cfg`).
Lint: `ansible-lint` (runs in CI on every push and pull request).

## Security notes

- No secrets in Git: inventory, host names and keys are gitignored; passwords are prompted at runtime
- Every config change is validated before it is applied (`sshd -t`, `visudo -cf`, `nft -c`)
- k3s hosts: default-drop host firewall; the Kubernetes API and kubelet are not reachable from the internet (checked from outside)
- Agent forwarding is enabled only for hosts in this inventory: root on a server can use a forwarded agent while you are connected
