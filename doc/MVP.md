# AsciiArtify — MVP: GitOps Deployment of go-demo-app with ArgoCD

## 1. Goal

Deploy the AsciiArtify product (the [go-demo-app](https://github.com/den-vasyliev/go-demo-app) sample application) to the local k3d cluster using ArgoCD, and demonstrate the full GitOps cycle:

1. ArgoCD tracks a Git repository (our fork).
2. A change is pushed to Git.
3. ArgoCD detects it and synchronizes the cluster automatically, with no manual `kubectl apply`.

Previous stages: [Concept.md](Concept.md) (tool comparison, k3d chosen) and [POC.md](POC.md) (ArgoCD installed on k3d, UI available to the team).

## 2. Prerequisites

The PoC environment is up and running:

```bash
kubectl config current-context     # expect: k3d-asciiartify
kubectl -n argocd get pods         # all pods Running
curl -I http://argocd.local:8080   # ArgoCD UI answers
```

If the cluster was removed, recreate it using steps 3–5 of [POC.md](POC.md).

## 3. Fork the repository

```bash
gh auth login                      # once
gh repo fork den-vasyliev/go-demo-app --clone
cd go-demo-app
```

Or use the **Fork** button on GitHub. The fork lives at `https://github.com/<username>/go-demo-app`. ArgoCD will track the fork, so changes pushed there can be demonstrated.

## 4. Inspect how the app is packaged

ArgoCD needs the path inside the repo that holds the Kubernetes definitions. Find it:

```bash
ls
find . -name Chart.yaml -o -name kustomization.yaml
find . -name '*.yaml' -not -path './.git/*' | head -30
```

| What you find | ArgoCD tool | `path` in the Application |
|---|---|---|
| `Chart.yaml` | Helm | the directory with `Chart.yaml` |
| `kustomization.yaml` | Kustomize | the directory with `kustomization.yaml` |
| plain `*.yaml` manifests | Directory | the directory with the manifests |

Write the result down. Below it is referred to as `<MANIFEST_PATH>`.

## 5. Create the ArgoCD Application

Declarative definition, stored in the AsciiArtify repository as `argocd/go-demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<username>/go-demo-app.git
    targetRevision: HEAD
    path: <MANIFEST_PATH>
  destination:
    server: https://kubernetes.default.svc
    namespace: go-demo
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # revert manual changes in the cluster
    syncOptions:
      - CreateNamespace=true
```

Apply it:

```bash
kubectl apply -f argocd/go-demo-app.yaml
kubectl -n argocd get applications
```

> The fork is public, so no repository credentials are needed. For a private repository, add credentials first (Settings → Repositories in the UI).

## 6. Verify the deployment

```bash
kubectl -n argocd get application go-demo-app    # SYNC STATUS: Synced, HEALTH: Healthy
kubectl -n go-demo get all
```

In the UI (<http://argocd.local:8080>) the `go-demo-app` tile should be **Synced** and **Healthy**. Open it to see the resource tree.

If the application stays `Progressing`, check `kubectl -n go-demo get pods` and `kubectl -n go-demo describe pod <name>`.

## 7. Access the application

Find the service and port the app exposes:

```bash
kubectl -n go-demo get svc
```

Quick access with port-forward (replace the service name and ports with the real ones):

```bash
kubectl -n go-demo port-forward svc/<service-name> 8081:<service-port>
curl http://localhost:8081
```

Or expose it via Ingress on the existing k3d load balancer (`http://<app-host>:8080`), following the same pattern as the ArgoCD Ingress in POC.md.

## 8. Demo: Sync Workflow

![ArgoCD sync Demo](/.data/argocd_sync_workflow.gif)

Goal: show that a `git push` alone changes the cluster.

Open two windows: the ArgoCD UI (application view) and a terminal.

```bash
# Terminal: watch the cluster
kubectl -n go-demo get pods -w
```

Make a visible change in the fork, for example increase the replica count (or change an image tag or an environment variable) in the file under `<MANIFEST_PATH>`:

```bash
cd go-demo-app
# edit the manifest/values: replicas: 1 -> replicas: 3
git add .
git commit -m "Scale go-demo-app to 3 replicas"
git push origin main
```

Expected behavior:

1. ArgoCD notices the new commit. By default it polls Git about every 3 minutes. To avoid waiting during the recording, click **Refresh** in the UI or run `argocd app get go-demo-app --refresh`.
2. The application briefly becomes **OutOfSync**, then ArgoCD applies the change automatically.
3. New pods appear in the terminal; the application returns to **Synced / Healthy**.

### Self-heal (optional bonus)

```bash
kubectl -n go-demo scale deployment <deployment-name> --replicas=1
```

ArgoCD detects the drift from Git and restores the replica count to the value stored in Git.

## 9. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `ComparisonError` / path not found | Wrong `path` or `targetRevision`: check the repo layout (section 4) |
| `ImagePullBackOff` | The image referenced in the manifests is not reachable from the cluster; check image name/tag and registry access |
| Stuck on `Progressing` | Pods not ready: `kubectl -n go-demo describe pod`, check logs |
| Not syncing after push | Wait for the poll interval or press **Refresh**; confirm `syncPolicy.automated` is set |
| Manual changes keep reverting | Expected with `selfHeal: true`; change Git instead |
| Application in wrong namespace | `destination.namespace` must match; `CreateNamespace=true` creates it |

## 10. Cleanup

```bash
kubectl delete -f argocd/go-demo-app.yaml   # removes the app and, with prune, its resources
k3d cluster delete asciiartify              # removes the whole cluster
```

## 11. Conclusions

- ArgoCD tracks the fork of the product repository and keeps the cluster in sync with it.
- A change committed to `main` reaches the k3d cluster without manual deployment commands.
- Self-heal and prune keep the cluster state equal to the Git state, which is the core GitOps guarantee.

**Next steps:** separate environments (dev/stage) through ApplicationSet, image tag updates from CI, TLS and SSO for the ArgoCD UI, and secret management.