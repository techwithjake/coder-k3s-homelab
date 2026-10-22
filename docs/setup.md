# Setup

Order I used. Fedora with SELinux enforcing and firewalld running.

## 1. k3s

```
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

Disable the bundled Traefik and ServiceLB (`k3s/config.yaml`), then `sudo systemctl restart k3s`.

kubectl as a normal user:

```
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

## 2. Firewalld

Trust the pod and service ranges so pods can reach the node. Runtime first, then
persist, so rules for existing Podman containers are not disturbed by a reload:

```
sudo firewall-cmd --add-port=6443/tcp
sudo firewall-cmd --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --zone=trusted --add-source=10.43.0.0/16
sudo firewall-cmd --runtime-to-permanent
```

Open the Coder NodePort later the same way (30080/tcp).

## 3. Postgres

```
kubectl create namespace coder
PGPASS=$(openssl rand -hex 16)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql bitnami/postgresql \
  --namespace coder \
  --set image.repository=bitnamilegacy/postgresql \
  --set auth.username=coder \
  --set auth.password=$PGPASS \
  --set auth.database=coder \
  --set primary.persistence.size=10Gi
kubectl create secret generic coder-db-url -n coder \
  --from-literal=url="postgres://coder:${PGPASS}@postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable"
```

Storage comes from k3s's default `local-path` provisioner. The legacy image
ran Postgres 17.5 while the chart targets 18.x; fine here because the database was new.

## 4. Coder

```
helm repo add coder-v2 https://helm.coder.com/v2
helm repo update
helm install coder coder-v2/coder \
  --namespace coder \
  --values helm/coder-values.yaml \
  --version 2.38.1
```

The chart's default service type is LoadBalancer, which would stay pending here
with ServiceLB disabled. The values file uses NodePort instead.

## 5. Template

Create a template from the Kubernetes (Deployment) base, add the `code-server`
module, then see `template/CHANGES.md`. The Docker base template does not fit
this setup because k3s uses containerd and has no Docker socket.
