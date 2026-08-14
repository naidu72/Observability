# homelab-gitops

App-of-apps ArgoCD repo. `bootstrap/root-app.yaml` is the one thing you
apply by hand; everything under `apps/` is then reconciled automatically.

## Layout

```
bootstrap/root-app.yaml              <- apply this once, manually
apps/kube-prometheus-stack-app.yaml  <- ArgoCD Application (multi-source)
apps/monitoring-secrets.yaml         <- ExternalSecrets (Grafana admin, Discord webhook)
charts/kube-prometheus-stack/values.yaml  <- Helm values, referenced by the Application above
```

## One-time setup steps, in order

### 1. Enable etcd metrics on pi5-1

k3s does not expose etcd's metrics endpoint by default. Since pi5-1 is
your sole control-plane/etcd node, and etcd quorum loss is exactly what
you've been debugging (rock64 EBADMSG incident), this is worth doing
before anything else.

```bash
ssh pi@pi5one
sudo vi /etc/rancher/k3s/config.yaml
```

Add:
```yaml
etcd-expose-metrics: true
```

Then restart:
```bash
sudo systemctl restart k3s
```

Verify metrics are reachable (from pi5-1 itself, or any node if you
also open the port in your firewall — nftables output earlier showed
warnings about iptables-nft compat, worth a separate look sometime):
```bash
curl -s http://192.168.0.X:2381/metrics | head
```

Update `charts/kube-prometheus-stack/values.yaml` -> `kubeEtcd.endpoints`
with pi5-1's actual IP.

### 2. Confirm your StorageClass name

```bash
kubectl get storageclass
```

If it isn't `nfs-client`, update `prometheus.prometheusSpec.storageSpec`
in `values.yaml` to match.

### 3. Confirm your cert-manager ClusterIssuer name

```bash
kubectl get clusterissuer
```

Update the `cert-manager.io/cluster-issuer` annotation in the Grafana
ingress section of `values.yaml` if it isn't `letsencrypt-dns01`.

### 4. Write secrets into Vault

```bash
vault kv put kv/monitoring/grafana \
  admin-user=admin \
  admin-password=<generate one, e.g. `openssl rand -base64 24`>

vault kv put kv/monitoring/discord \
  webhook_url=<your Discord channel webhook URL>
```

These reuse your existing `vault-backend` ClusterSecretStore — no new
Vault auth or ESO config needed.

### 5. Push this repo, then bootstrap ArgoCD

Push to whatever git remote you use (update the `repoURL` fields in
`bootstrap/root-app.yaml` and `apps/kube-prometheus-stack-app.yaml`
first if this differs from the placeholder).

```bash
kubectl apply -f bootstrap/root-app.yaml
```

ArgoCD will pick up `apps/kube-prometheus-stack-app.yaml` and
`apps/monitoring-secrets.yaml`, create the `monitoring` namespace, and
deploy everything.

### 6. Watch it come up

```bash
kubectl get application -n argocd
kubectl get pods -n monitoring -w
```

Grafana should become reachable at `https://grafana.naidu72.info` once
the ingress and cert-manager challenge complete (DNS-01 via Cloudflare,
same as your other services — should take under a minute).

## Known gaps / next steps

- `additionalScrapeConfigs` is intentionally empty in values.yaml — once
  core cluster metrics are flowing, natural next targets are Vault,
  AWX, and the ArgoCD components themselves (all commonly have
  Prometheus endpoints or exporters).
- Alert thresholds (85%/95% memory) are starting guesses — revisit
  after a week of real data, especially given pi5-1 already runs hot
  at ~74% baseline.
- No Loki/log aggregation yet — this setup is metrics-only. Worth a
  separate pass if you want centralized logs alongside metrics.
