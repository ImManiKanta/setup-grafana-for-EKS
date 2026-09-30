# Grafana + Prometheus Monitoring on Amazon EKS

Set up cluster monitoring on Amazon EKS using the `kube-prometheus-stack` Helm chart. This gives you Prometheus for metrics storage, Alertmanager for alerts, and Grafana for dashboards.

## Contents

- [Prerequisites](#prerequisites)
- [1. Create the EKS cluster](#1-create-the-eks-cluster)
- [2. Connect kubectl to the cluster](#2-connect-kubectl-to-the-cluster)
- [3. Install Helm](#3-install-helm)
- [4. Install kube-prometheus-stack](#4-install-kube-prometheus-stack)
- [5. Verify the installation](#5-verify-the-installation)
- [6. Access Grafana and Prometheus](#6-access-grafana-and-prometheus)
- [Components installed](#components-installed)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

## Prerequisites

- An AWS account and credentials configured (`aws configure` or an SSO profile)
- `aws` CLI v2, `kubectl` and `eksctl` installed
- Run `kubectl` on your **own machine** if you want to open Grafana in your browser. Port-forwarding from AWS CloudShell or a remote VM does not work with your local browser.

## 1. Create the EKS cluster

Create the cluster with `eksctl` (or with Terraform if you prefer).

`cluster.yaml`

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: demo-cluster
  region: us-east-1

managedNodeGroups:
  - name: workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
```

```bash
eksctl create cluster -f cluster.yaml
```

Cluster creation takes around 15-20 minutes.

## 2. Connect kubectl to the cluster

`eksctl` normally updates your kubeconfig automatically. To connect from another machine or to refresh it:

```bash
aws eks update-kubeconfig --region us-east-1 --name demo-cluster
kubectl get nodes
```

You should see your 2 worker nodes in `Ready` state.

## 3. Install Helm

On an Amazon Linux machine:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify the installation:

```bash
helm version
```

## 4. Install kube-prometheus-stack

Add the chart repository:

```bash
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the chart into a new `monitoring` namespace:

```bash
helm install monitoring prometheus/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

> Type `--create-namespace` with two plain hyphens. Copying from a document or chat can turn it into an em dash (`—`), which makes the command fail.

## 5. Verify the installation

Watch the pods start (press `Ctrl+C` to stop watching):

```bash
kubectl get pods -n monitoring -w
```

All pods should reach `Running` status. It usually takes a few minutes.

## 6. Access Grafana and Prometheus

Port-forward the services to your local machine. Run each command in its own terminal and leave it running.

**Grafana**

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open <http://localhost:3000>

**Prometheus**

```bash
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
```

Open <http://localhost:9090>

### Grafana login

- **Username:** `admin`
- **Password:** read it from the Kubernetes secret:

```bash
kubectl -n monitoring get secret monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

The chart's default password is `prom-operator`. Change it after your first login.

## Components installed

| Component | What it does |
|---|---|
| **Prometheus Operator** | Manages the Prometheus and Alertmanager instances |
| **Prometheus** | Time-series database that stores all application and infrastructure metrics |
| **Alertmanager** | Handles alerts fired by Prometheus rules |
| **Node Exporter** | Collects metrics from each node (CPU, memory, disk, network) |
| **kube-state-metrics** | Exposes the state of cluster objects such as Deployments, Pods and Services |
| **Grafana** | Visualisation tool for all the metrics, with built-in Kubernetes dashboards |

## Troubleshooting

### `kubectl` says "the server has asked for the client to provide credentials"

Your AWS identity is missing, expired, or not the one used to create the cluster.

```bash
aws sts get-caller-identity
```

- With SSO, log in again: `aws sso login --profile <profile>`, then `aws eks update-kubeconfig --region us-east-1 --name demo-cluster --profile <profile>`.
- Unset stale environment variables that override your profile: `env | grep AWS_`.

### `kubectl` says `Forbidden` (for example "cannot list resource nodes")

You are authenticated, but the cluster hasn't granted your identity permissions. From an identity that already has admin access, create an access entry:

```bash
aws eks create-access-entry --cluster-name demo-cluster --region us-east-1 \
  --principal-arn <your-iam-user-or-role-arn>

aws eks associate-access-policy --cluster-name demo-cluster --region us-east-1 \
  --principal-arn <your-iam-user-or-role-arn> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

If the entry already exists, the first command returns `ResourceInUseException`. That is harmless, and you only need the second command. Wait about 30 seconds for the change to apply.

For everyday use, prefer a narrower policy than cluster admin. Note that `AmazonEKSViewPolicy` does not allow `kubectl port-forward`.

### Port-forward prints "Forwarding from..." but the browser won't load

That message means the tunnel is working, and the command stays in the foreground. Check these:

1. Are you running `kubectl` on the same machine as your browser? In AWS CloudShell, `127.0.0.1` is not your laptop. Run `kubectl` locally instead.
2. Use `http://127.0.0.1:3000`, not `https://`.
3. Test the tunnel from a second terminal: `curl -I http://127.0.0.1:3000/login` should return `200 OK`.
4. If a company VPN or proxy interferes, add `127.0.0.1, localhost` to the browser's proxy bypass list, or try a private window in another browser.

### "Address already in use" when port-forwarding

An old port-forward is still running.

```bash
pkill -f "kubectl port-forward"
```

Or use a different local port, for example `kubectl port-forward svc/monitoring-grafana 38080:80 -n monitoring`, then open port 38080.

### Grafana rejects the admin password

Grafana reads the admin password from the secret only when its database is first created. If the login fails even with the value from the secret:

```bash
# Set a new password in the secret, then restart Grafana
kubectl -n monitoring patch secret monitoring-grafana \
  -p '{"stringData":{"admin-password":"<new-password>"}}'
kubectl -n monitoring rollout restart deploy/monitoring-grafana
kubectl -n monitoring rollout status deploy/monitoring-grafana
```

Then restart the port-forward, because it stays attached to the old pod. Alternatively, reset it inside the running pod:

```bash
kubectl -n monitoring exec -it deploy/monitoring-grafana -c grafana -- \
  grafana cli admin reset-admin-password '<new-password>'
```

Check the stored value for stray characters, such as curly quotes copied from a document:

```bash
kubectl -n monitoring get secret monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d | xxd
```

To make the password persist across Helm upgrades:

```bash
helm upgrade monitoring prometheus/kube-prometheus-stack -n monitoring \
  --reuse-values --set grafana.adminPassword='<new-password>'
```

### Security notes

- Never paste AWS access keys into chats, tickets or Git. If a key is exposed, deactivate and delete it in IAM, then create a new one.
- Prefer SSO profiles (short-lived credentials) over long-lived IAM user keys.
- Don't expose Grafana publicly with the default admin login. Use port-forward, a VPN-only internal load balancer, or an Ingress with SSO.

## Cleanup

Remove the monitoring stack:

```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

Delete the cluster to stop AWS charges:

```bash
eksctl delete cluster -f cluster.yaml
```

Prometheus CRDs are left behind after `helm uninstall`. To remove them, follow the [kube-prometheus-stack uninstall notes](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#uninstall-chart).
