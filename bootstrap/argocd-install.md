# ArgoCD Installation & K3s Homelab Bootstrapping Guide

This guide details how to perform a clean installation/rebuild of your K3s cluster and bootstrap your GitOps applications using ArgoCD.

---

## Step 1: Destroy Existing K3s Cluster
Run this command to completely remove the old K3s installation and any corrupted resources/states:
```bash
sudo /usr/local/bin/k3s-uninstall.sh
```
Verify that the service is no longer present:
```bash
sudo systemctl status k3s
```

---

## Step 2: Reinstall K3s
Install a clean, single-node K3s cluster:
```bash
curl -sfL https://get.k3s.io | sh -
```
Wait until the node is in `Ready` status:
```bash
sudo kubectl get nodes
```

---

## Step 3: Install ArgoCD (Official Manifests)
Deploy ArgoCD components cleanly without Helm management to prevent self-management and immutable selector conflicts:
```bash
# 1. Create namespace
sudo kubectl create namespace argocd

# 2. Deploy stable ArgoCD manifests
sudo kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for all ArgoCD components to be ready
sudo kubectl wait --for=condition=ready pod --all -n argocd --timeout=300s
```

---

## Step 4: Configure ArgoCD Server Insecure HTTP Mode
To allow Traefik to terminate TLS (HTTPS) on the edge and route HTTP to `argocd-server` internally, configure insecure mode via the standard configuration parameters:
```bash
# 1. Enable insecure mode in ConfigMap
sudo kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type merge -p '{"data":{"server.insecure":"true"}}'

# 2. Restart the server deployment to load the new config
sudo kubectl rollout restart deployment argocd-server -n argocd

# 3. Wait for deployment status to show success
sudo kubectl rollout status deployment/argocd-server -n argocd

# 4. Verify insecure mode is active (should output: true)
sudo kubectl get configmap argocd-cmd-params-cm -n argocd -o jsonpath='{.data.server\.insecure}'
```

---

## Step 5: Apply Ingress & Root App Bootstrapping
Expose the ArgoCD server web UI and bootstrap your nested App-of-Apps controller:
```bash
# 1. Deploy Ingress routing to the HTTP port of argocd-server (port 80)
sudo kubectl apply -f bootstrap/argocd-ingress.yaml

# 2. Apply the single root Application (homelab-root)
sudo kubectl apply -f bootstrap/root-app.yaml
```

---

## Step 6: Verify and Fetch Sealed Secrets Certificate
Wait for the apps to synchronize and fetch the sealed secrets public certificate:
```bash
# 1. Verify homelab-root is applied and has synced sub-applications (cluster-tools and apps)
sudo kubectl get applications -n argocd

# 2. Verify sealed-secrets pod is running in kube-system
sudo kubectl get pods -n kube-system | grep sealed

# 3. Create the directory for generated certs
mkdir -p generated

# 4. Fetch the public certificate and save it to the generated folder
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > generated/pub-cert.pem
```
