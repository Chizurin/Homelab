# Homelab — k3s quick setup

Leaving this reference for bootstrapping and recovering this homelab cluster in case I break something

Currently using this as a single node cluster to learn GitOps (ArgoCD and Helm)

The system has two layers of desired state:
- **OS layer** — the bootc image at https://github.com/chizurin/bootc.git (k3s should be installed and configured)
- **App layer** — this repo (everything ArgoCD manages)

To fully recover on a fresh machine: boot the image, run the bootstrap, done.

---

## Prerequisites

Boot the machine with the bootc image. k3s will already be running.

Clone this repo on the server:

```sh
git clone https://github.com/chizurin/Homelab.git
cd Homelab
```

---

## Bootstrap *(run once, or to recover)*

### 1. Kubeconfig

The bootc image sets `--write-kubeconfig-mode=644`, so this may already be done:

```sh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
```

Optional — make `kubectl` point to `k3s kubectl` in interactive shells:

```sh
echo 'alias kubectl="k3s kubectl"' >> ~/.bashrc
source ~/.bashrc
```

### 2. Install Argo CD

```sh
kubectl apply -f bootstrap/argocd/namespace.yaml
kubectl apply --server-side -f bootstrap/argocd/install.yaml
kubectl -n argocd rollout status deployment argocd-server
```

### 3. Connect the cluster to git

```sh
kubectl apply -f apps/root-app.yaml
```

Argo CD will now sync everything under `apps/` from the repo automatically.

### 4. Get the initial Argo CD admin password

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## Client Setup

Copy the kubeconfig from the server and update the API endpoint:

```sh
scp user@your-server-ip:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/your-server-ip/g' ~/.kube/config
kubectl get nodes
```

If the API is unreachable, ensure port `6443` is open on the server firewall.

### Access the Argo CD UI

Port-forward on the server, then SSH tunnel from your client:

```sh
# on the server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# on the client
ssh -L 8080:localhost:8080 user@your-server-ip
```

Browse to `https://localhost:8080`.

---

## Sealed Secrets

### Install kubeseal (client)

```sh
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.26.0-linux-amd64.tar.gz kubeseal
sudo mv kubeseal /usr/local/bin/
```

To seal a secret, kubeseal fetches the cert from the cluster automatically:

```sh
kubeseal < secret.yaml > sealed-secret.yaml
```

### Install the controller (server)

```sh
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system \
  --set fullnameOverride=sealed-secrets-controller
kubectl rollout status deployment sealed-secrets-controller -n kube-system
```

---

## Troubleshooting

- Verify `kubectl` context with `kubectl get nodes`.
- Ensure port `6443` is reachable from any remote client that needs API access.
- Use `kubectl -n argocd rollout status deployment argocd-server` to check Argo CD readiness.
