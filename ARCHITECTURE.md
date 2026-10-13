# Architecture

How the k3s server is built and how applications get to production.

## Overview

```
                    Internet
                       │
                  Cloudflare (TLS, proxy)
                       │  outbound tunnel, started from the cluster
┌──────────────────────┼──────────────────────────────────────────┐
│ k3s server           ▼                                          │
│                cloudflared (2 replicas)                         │
│                      │                                          │
│                   Traefik (ClusterIP)  ── routes by Host ──►  app pods
│                                                                 │
│   Argo CD ◄── watches app repos in Git                          │
│   Sealed Secrets controller ── decrypts SealedSecrets           │
│                                                                 │
│   Host firewall (nftables): only SSH is open                    │
└─────────────────────────────────────────────────────────────────┘
```

No HTTP/HTTPS ports are open on the server. All public traffic comes through
Cloudflare Tunnel, which is an outbound connection, so the origin IP is not exposed.

## Two repos, two jobs

| Where | What it answers | Changes how often |
|---|---|---|
| `ansible-vps` (this repo) | How to build the server | Rarely: upgrades, new platform parts |
| App repo, `k8s/` folder | What runs on the server | On every release |

### This repo: `playbooks/k3s.yml`

| Role | Responsibility |
|---|---|
| `k3s` | nftables host firewall, pinned k3s binary, Traefik as ClusterIP |
| `sealed_secrets` | Sealed Secrets controller with our own sealing key |
| `argocd` | Argo CD, private repo credentials (sealed), Application objects |
| `cloudflared` | Cloudflare Tunnel connector and routing rules |

All roles deploy to the cluster through the k3s auto-deploy folder
(`/var/lib/rancher/k3s/server/manifests/`), so no Helm or Kubernetes
collections are needed on the control node.

### Inventory (gitignored)

Environment-specific values live in `inventory/group_vars/k3s_servers.yml`:

- `cloudflared_tunnel_id`, `cloudflared_ingress`: tunnel and public hostnames
- `argocd_apps`: which repos, branches and folders Argo CD deploys

### App repo: `k8s/`

Kustomize folder: Deployment, Service, Ingress, SealedSecrets,
and `kustomization.yaml` with the image tag. Argo CD syncs it automatically
(prune and self-heal on).

## Request path

1. Browser → Cloudflare (TLS ends here)
2. Cloudflare → `cloudflared` pod through the tunnel
3. `cloudflared` → Traefik; unknown hostnames get 404 at this step
4. Traefik → app Service by the Ingress host rule
5. Service → a ready app pod

## Release path

1. Push to `main` in the app repo
2. GitHub Actions builds the image and pushes it to GHCR as `sha-<commit>`
3. A commit sets that tag in `k8s/kustomization.yaml`
4. Argo CD sees the change and rolls out with zero downtime (`maxUnavailable: 0`)

Rollback: set the previous tag in `kustomization.yaml`.

## Secrets

- Stored in Git only as SealedSecrets; the controller decrypts them in the cluster.
- The sealing key pair is ours, generated once and kept outside Git
  (password manager + `files/sealed-secrets/`, gitignored). Automatic key renewal is off.
- The same key is installed on every rebuild, so SealedSecrets in all repos keep working
  after moving to a new server.
- The public certificate is committed in app repos, so anyone can seal a secret offline:
  `kubeseal --cert k8s/sealing-cert.pem`.

## Access

Nothing is exposed; everything goes through SSH.

```bash
# kubectl from the control node
ssh -fN -L 6443:127.0.0.1:6443 <host>
KUBECONFIG=~/.kube/<host>.yaml kubectl get pods -A

# Argo CD UI: on the server, then open https://localhost:8080 via an SSH tunnel
sudo kubectl -n argocd port-forward svc/argocd-server 8080:443
ssh -N -L 8080:127.0.0.1:8080 <host>
```

## Common tasks

| Task | How |
|---|---|
| Release an app | Push to `main` in the app repo, then update the image tag in `k8s/` |
| Add an app | `k8s/` folder in its repo + entry in `argocd_apps` + hostname in `cloudflared_ingress` + CNAME to the tunnel |
| Upgrade a platform part | Change the version variable in the role defaults, run the playbook |
| Run one role | `ansible-playbook playbooks/k3s.yml --limit <host> --tags <role>` |
| Move to a new server | See below |

## Moving to a new server

1. Base playbooks: `setup.yml`, `keys.yml`, `harden.yml`
2. Add the host to `k3s_servers`, run `playbooks/k3s.yml`
3. Argo CD pulls every app from Git; `cloudflared` joins the same tunnel
4. Check pods and the tunnel status, then switch off the old server

No DNS changes are needed: DNS points to the tunnel, not to a server IP.
State: apps are stateless, databases are external.
