# Homelab — k3s quick setup

A compact reference for setting up k3s (for me), installing Argo CD, and using Sealed Secrets.

## Server (run on your k3s server)

Prepare local kubeconfig (the bootc image sets `--write-kubeconfig-mode=644`, so this may already be done):

```sh
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER ~/.kube/config
```

Optional: make `kubectl` point to `k3s kubectl` in interactive shells:

```sh
echo 'alias kubectl="k3s kubectl"' >> ~/.bashrc
source ~/.bashrc
```

Install Argo CD:

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# wait for server
kubectl -n argocd rollout status deployment argocd-server
```

Get the initial Argo CD admin password:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Access Argo CD UI (option 1 — SSH tunnel to server):

```sh
ssh -L 8080:localhost:8080 user@your-server-ip
# then open http://localhost:8080
```

Option 2 — port-forward from the server:

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080 (accept warning if needed)
```

## Client (workstation that manages the cluster)

Copy kubeconfig from the server and update the API endpoint IP:

```sh
scp user@your-server-ip:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/your-server-ip/g' ~/.kube/config
kubectl get nodes
```

If the API is unreachable from the client, ensure port `6443` is open on the server firewall.

## Sealed Secrets (manage encrypted secrets)

Client — install `kubeseal` (example version; pick latest as needed):

```sh
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.26.0-linux-amd64.tar.gz kubeseal
sudo mv kubeseal /usr/local/bin/
# fetch cluster public key to encrypt secrets locally
kubeseal --fetch-cert > pub-cert.pem
```

Server — install the controller via Helm:

```sh
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system \
  --set fullnameOverride=sealed-secrets-controller
kubectl rollout status deployment sealed-secrets-controller -n kube-system
```

## Troubleshooting

- Verify `kubectl` context with `kubectl get nodes`.
- Ensure port `6443` is reachable from any remote client that needs API access.
- Use `kubectl -n argocd rollout status deployment argocd-server` to check Argo CD readiness.
