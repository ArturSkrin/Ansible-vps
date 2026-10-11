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

## How access works

Keys are kept per server group in `files/<keys_dir>/`, and each inventory group points to its set via `group_vars`:

| Key is in | Can log in | sudo |
|---|---|---|
| `authorized_keys` only | yes | password |
| `authorized_keys` + `sudo_authorized_keys` | yes | no password (SSH agent) |

So a CI deploy key can log in but cannot become root, while admins get passwordless sudo through their forwarded SSH agent (`pam_ssh_agent_auth`).
The sudo key file is owned by root, so a user cannot add keys to escalate. The password stays as a fallback for the provider's web console.

## Problems found and fixed on a real fleet

**Password login left enabled by cloud-init.** On Ubuntu 22.04+ `sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`, and sshd uses the first value it finds.
Cloud images often ship `50-cloud-init.conf` with `PasswordAuthentication yes`, which silently overrides edits to the main `sshd_config`.
The audit found two servers where password login was still on. Hardening now goes into `00-hardening.conf` (read first), is validated with `sshd -t` before restart, and is checked with `sshd -T` after.

**Key-based sudo failing only for Ansible on Ubuntu 24.04.** Ansible runs `sudo -n` (non-interactive). Since sudo 1.9.10, `-n` skips authentication unless `noninteractive_auth` is set, so key auth worked by hand but not from Ansible.
`harden.yml` reads the sudo version and adds `Defaults noninteractive_auth` only where it is supported; older sudo would reject the option.

## Usage

```bash
cp inventory/hosts.ini.example inventory/hosts.ini       # your servers and groups
cp inventory/group_vars/example.yml inventory/group_vars/<group>.yml
mkdir files/<group> && cp files/example/*.example files/<group>/   # then drop the .example suffix and add keys

# New server (as root or the provider's initial user; add -k if it only has a password)
ansible-playbook playbooks/setup.yml  --limit <host> -e ansible_user=root
ansible-playbook playbooks/keys.yml   --limit <host> -K
ansible-playbook playbooks/harden.yml --limit <host> -K

# Day to day
ansible-playbook playbooks/keys.yml --check --diff     # preview key changes
ansible-playbook playbooks/keys.yml                    # apply to the whole fleet
ansible-playbook playbooks/ssh_audit.yml               # audit, no sudo password needed
```

Requirements: Ansible 2.15+, `passlib` on the control node, Ubuntu 22.04/24.04 targets, an SSH agent with your key loaded (`ForwardAgent` is set in `ansible.cfg`).

## Security notes

- No secrets in Git: inventory, host names and keys are gitignored; passwords are prompted at runtime
- Every config change is validated before it is applied (`sshd -t`, `visudo -cf`)
- Agent forwarding is enabled only for hosts in this inventory: root on a server can use a forwarded agent while you are connected
