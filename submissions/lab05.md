# Lab 5 Submission

## Task 1. CI pipeline and GitOps preparation

### What I added

I implemented the required CI workflow in [`.github/workflows/ci.yml`](/Users/pavel/Documents/study/sre/lab1/SRE-Intro/.github/workflows/ci.yml) and updated the Kubernetes Deployments in [`k8s/gateway.yaml`](/Users/pavel/Documents/study/sre/lab1/SRE-Intro/k8s/gateway.yaml), [`k8s/events.yaml`](/Users/pavel/Documents/study/sre/lab1/SRE-Intro/k8s/events.yaml), and [`k8s/payments.yaml`](/Users/pavel/Documents/study/sre/lab1/SRE-Intro/k8s/payments.yaml).

The workflow:

- triggers on push to `main`
- logs into `ghcr.io`
- builds and pushes all 3 service images with the tag `${{ github.sha }}`

The manifests now:

- use `ghcr.io/shnupel/quickticket-<service>:REPLACE_WITH_COMMIT_SHA`
- set `imagePullPolicy: Always`
- define `imagePullSecrets: [ghcr-secret]`
- add a visible `version: "v2"` label to `gateway` for the GitOps sync proof

### CI workflow

File:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  IMAGE_OWNER: shnupel

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      fail-fast: false
      matrix:
        service:
          - gateway
          - events
          - payments

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push ${{ matrix.service }}
        uses: docker/build-push-action@v6
        with:
          context: ./app/${{ matrix.service }}
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_OWNER }}/quickticket-${{ matrix.service }}:${{ github.sha }}
```

### Required verification steps

The remaining part of the lab must be executed after pushing this branch to GitHub and after starting a working Kubernetes cluster with ArgoCD access.

1. Push workflow and manifests to `main`.
2. Wait for the GitHub Actions run to finish successfully.
3. Replace `REPLACE_WITH_COMMIT_SHA` in the three Deployment manifests with the actual commit SHA from the successful CI run.
4. Create the pull secret:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=shnupel \
  --docker-password=YOUR_CLASSIC_PAT
```

5. Install ArgoCD and create the application:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=120s
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

kubectl port-forward svc/argocd-server -n argocd 8443:443 &
argocd login localhost:8443 --insecure --username admin --password <PASSWORD>
argocd app create quickticket \
  --repo https://github.com/Shnupel/SRE-Intro.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
argocd app get quickticket
```

### Proof of work

I could not honestly paste the final runtime outputs for this section from the current environment, because:

- the local kubeconfig points to `https://0.0.0.0:53559`, but the API server currently returns `connection refused`
- `k3d` is not installed in this shell environment
- GitHub Actions / ghcr / ArgoCD verification requires external services and a running cluster

Current local checks:

```bash
git remote -v
git rev-parse HEAD
kubectl get nodes
```

Output:

```text
origin  git@github.com:Shnupel/SRE-Intro.git (fetch)
origin  git@github.com:Shnupel/SRE-Intro.git (push)

9f866f76ff167ea5acab296dfe01bc4594c75cbc

The connection to the server 0.0.0.0:53559 was refused - did you specify the right host or port?
```

After the cluster is started and the workflow completes, paste the following evidence here:

1. GitHub Actions run link with green status
2. Output of:

```bash
gh api user/packages?package_type=container --jq '.[].name'
```

Expected package names:

```text
quickticket-gateway
quickticket-events
quickticket-payments
```

3. Output of:

```bash
argocd app get quickticket
```

Expected key lines:

```text
Sync Status: Synced
Health Status: Healthy
```

4. Output proving the Git change was synced:

```bash
kubectl get deployment gateway -o jsonpath='{.metadata.labels.version}'
echo
```

Expected output:

```text
v2
```

### What happens if someone manually runs `kubectl edit` on an ArgoCD-managed resource?

ArgoCD treats Git as the source of truth. A manual `kubectl edit` changes the live cluster state, so ArgoCD will detect drift and mark the application `OutOfSync`.

If automated sync is enabled, ArgoCD will reconcile the resource back to the version stored in Git. In practice, manual changes on managed resources are temporary unless the same change is committed to the repository.
