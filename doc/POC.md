# AsciiArtify — Proof of Concept: ArgoCD on k3d

## 1. Goal

Show that a GitOps system can be run on the local Kubernetes cluster approved in [Concept.md](Concept.md) (**k3d**), and that the team can reach the ArgoCD web UI. The result is a working ArgoCD installation ready for MVP work.

**Scope (minimal set of features):**

- k3d cluster created from a config file (reproducible);
- ArgoCD installed from the official manifests;
- web UI available to the team through Ingress;
- team accounts with basic access control;
- one sample application synced from Git to prove the setup works.

## 2. Prerequisites

| Tool | Purpose |
|---|---|
| Docker-compatible runtime (Colima, Docker Desktop, OrbStack) | runs k3d nodes |
| `k3d` | local Kubernetes cluster |
| `kubectl` | cluster access |
| `argocd` CLI (optional, needed for team accounts) | `brew install argocd` |

## 3. Create the cluster

```bash
k3d cluster create --config .data/k3d-config.yaml
kubectl config current-context    # expect: k3d-asciiartify
kubectl get nodes                 # expect: 3 nodes in Ready state
```

## 4. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get all -n argocd

kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
kubectl -n argocd get pods
```

> `--server-side` is used because some ArgoCD CRDs are too large for client-side apply.

## 5. Access the UI

### Option A: quick access with port-forward (single user)

```bash
kubectl -n argocd port-forward svc/argocd-server 9090:443
```

Open <https://localhost:9090> (accept the self-signed certificate warning).
Port `9090` is used so it does not clash with the k3d port mapping on `8080`.

### Option B: team access through Ingress (recommended for the PoC)

Traefik is installed in k3s by default. ArgoCD is switched to plain HTTP inside the cluster, so TLS can be handled at the edge later.

```bash
# 1. Run argocd-server in insecure (HTTP) mode
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy/argocd-server
kubectl -n argocd rollout status deploy/argocd-server

# 2. Ingress
kubectl apply -f .data/ingress.yaml

# 3. Local name resolution (each team member, once)
echo "127.0.0.1 argocd.local" | sudo tee -a /etc/hosts
```

Open <http://argocd.local:8080>.

### Initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in as `admin` with this password. Change it after first login and delete the initial secret once no longer needed.

## 6. Team access

Create login accounts for team members and restrict permissions: read-only by default, one admin.

```bash
# Log in with the CLI
argocd login localhost:9090 --insecure --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"

# Accounts
kubectl -n argocd patch configmap argocd-cm --type merge \
  -p '{"data":{"accounts.dev1":"login","accounts.dev2":"login"}}'

# RBAC: everyone read-only, dev1 is admin
kubectl apply -f .data/rbac.yaml

# Passwords
argocd account update-password --account dev1 \
  --current-password '<admin-password>' --new-password '<dev1-password>'
argocd account update-password --account dev2 \
  --current-password '<admin-password>' --new-password '<dev2-password>'

argocd account list
```

> Replace `dev1`/`dev2` with real names. Do not commit passwords to Git.

## 8. Demo

![Installation Demo](/.data/poc1.gif)
![Team Access Demo](/.data/poc2.gif)

## 9. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `curl` returns 404 | Ingress missing or in the wrong namespace: `kubectl get ingress -A` |
| Redirect loop in the browser | `server.insecure` not applied: restart `argocd-server` |
| `argocd.local` does not resolve | Missing `/etc/hosts` entry |
| Pods stuck in `Pending` / `ContainerCreating` | Not enough resources: increase CPU/memory of the container runtime |
| CRD "too long" error on install | Use `--server-side` as shown in section 4 |
| Wrong cluster | `kubectl config current-context`, expected `k3d-asciiartify` |

## 10. Cleanup

```bash
k3d cluster delete asciiartify
```

## 11. Conclusions and next steps

- ArgoCD runs on k3d with minimal resources and starts quickly.
- The UI is available to the team through Ingress; accounts and roles are configured.
- A sample Git-driven application is synced automatically.

**Next steps for the MVP:**

- Move the AsciiArtify manifests (or a Helm chart) to a dedicated Git repository and point ArgoCD at it.
- Use an app-of-apps or ApplicationSet pattern for several environments.
- Add SSO or stronger authentication, TLS, and secret management before any shared environment.