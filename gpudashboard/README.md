# gpu-dashboard

A static GPU cluster monitoring dashboard served by unprivileged nginx on Kubernetes and exposed through a `LoadBalancer` Service. The dashboard shows a simulated 8-node × 8-GPU pool. All metrics are generated in the browser, so there is no backend or external dependency.

## What the dashboard shows

- **Utilization heatmap:** one cell per GPU, shaded by utilization. An amber border marks a GPU above 82 °C, and a red cell marks a simulated Xid fault. Select a cell to see that GPU's details.
- **Trends:** mean GPU utilization, total power draw (kW) and fabric throughput (GB/s), each with a 2-minute sparkline.
- **GPU detail:** job, utilization, temperature, memory, power and state for the selected GPU.
- **Node table:** status (Healthy, Draining or GPU fault), job, mean utilization, hottest GPU and power for each node.

The data refreshes every 2 seconds. `gpu-node-08` is always shown as draining.

## Layout

```
gpu-dashboard/
├── index.html          # Dashboard page (HTML/CSS/JS, self-contained)
├── nginx.conf          # nginx server block, mounted as conf.d/default.conf
├── namespace.yaml      # Namespace with Pod Security "restricted" enforced
├── deployment.yaml     # 2 replicas of nginx-unprivileged
├── service.yaml        # LoadBalancer Service, port 80 -> 8080
├── pdb.yaml            # PodDisruptionBudget, minAvailable: 1
└── kustomization.yaml  # Ties it together; generates the ConfigMaps
```

`kustomization.yaml` builds two ConfigMaps: `dashboard-html` from `index.html` and `dashboard-nginx` from `nginx.conf`. Each name gets a content-hash suffix, so any edit to either file changes the pod template and triggers a rolling restart.

## Requirements

- Kubernetes 1.25 or later (for Pod Security admission and `policy/v1` PDBs)
- `kubectl` 1.14 or later (for built-in `-k` / kustomize support)
- A load balancer implementation for `type: LoadBalancer`: a cloud provider, MetalLB or kube-vip. Without one, use NodePort (see [Exposing without a load balancer](#exposing-without-a-load-balancer)).
- Nodes that can pull `nginxinc/nginx-unprivileged:1.27-alpine` from Docker Hub, or a mirror of it.

## Deploy

```bash
kubectl apply -k .
kubectl -n gpu-dashboard rollout status deploy/gpu-dashboard
kubectl -n gpu-dashboard get svc gpu-dashboard -w      # wait for EXTERNAL-IP
```

Preview the rendered manifests without applying them:

```bash
kubectl kustomize .
kubectl apply -k . --dry-run=server
```

## Verify

```bash
IP=$(kubectl -n gpu-dashboard get svc gpu-dashboard \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -s  http://$IP/healthz                             # -> ok
curl -sI http://$IP/ | grep -Ei 'content-security|x-frame|x-content-type'
kubectl -n gpu-dashboard get pods -o wide               # 2 Ready pods, ideally on different nodes
kubectl -n gpu-dashboard get endpointslices -l kubernetes.io/service-name=gpu-dashboard
```

Then open `http://$IP/` in a browser.

## Configuration

| Setting | Where | Default |
|---|---|---|
| Replicas | `deployment.yaml` → `spec.replicas` | `2` |
| Image | `deployment.yaml` → `containers[0].image` | `nginxinc/nginx-unprivileged:1.27-alpine` |
| Container port | `nginx.conf` → `listen`, and `deployment.yaml` → `containerPort` | `8080` |
| Service port | `service.yaml` → `ports[0].port` | `80` |
| Service type | `service.yaml` → `spec.type` | `LoadBalancer` |
| CPU / memory | `deployment.yaml` → `resources` | 10m / 16Mi requested, 200m / 64Mi limit |
| Namespace | `kustomization.yaml` → `namespace`, and `namespace.yaml` | `gpu-dashboard` |

If you change the container port, change it in both `nginx.conf` and `deployment.yaml`. The Service targets the named port `http`, so it needs no change.

### Pinning a MetalLB address

`spec.loadBalancerIP` is deprecated. Use the MetalLB annotation instead:

```yaml
metadata:
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.0.0.50
```

### Exposing without a load balancer

In `service.yaml`, set `type: NodePort` and uncomment `nodePort: 30080`. The dashboard is then at `http://<any-node-ip>:30080`.

## Updating the dashboard

Edit `index.html` or `nginx.conf`, then run:

```bash
kubectl apply -k .
kubectl -n gpu-dashboard rollout status deploy/gpu-dashboard
```

The ConfigMap hash changes, so the pods roll automatically. Do not edit the ConfigMaps in the cluster directly. `nginx.conf` is mounted with `subPath`, which never picks up in-place changes, and any live edit is overwritten on the next apply.

To test the page locally without a cluster:

```bash
docker run --rm -p 8080:8080 \
  -v "$PWD/index.html:/usr/share/nginx/html/index.html:ro" \
  -v "$PWD/nginx.conf:/etc/nginx/conf.d/default.conf:ro" \
  nginxinc/nginx-unprivileged:1.27-alpine
# open http://localhost:8080
```

## Security

**Pod hardening.** The pods pass the `restricted` Pod Security Standard, which the namespace enforces:

- They run as UID/GID 101 (non-root) with the `RuntimeDefault` seccomp profile.
- All capabilities are dropped and `allowPrivilegeEscalation` is `false`.
- The root filesystem is read-only. nginx writes only to an in-memory `emptyDir` at `/tmp`.
- `automountServiceAccountToken` is `false`, because the app never calls the Kubernetes API.

**Response headers.** nginx sets `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer` and `Cache-Control: no-cache`. These are set at the server level on purpose: an `add_header` inside a `location` block would silently drop all of them. The CSP allows inline scripts and styles because the page is a single self-contained file.

**Not included.** The Service serves plain HTTP with no authentication. Before exposing it beyond a trusted network, do one of the following:

- Put it behind an ingress or Gateway with TLS (cert-manager) and SSO (oauth2-proxy), and switch the Service to `ClusterIP`.
- Restrict source ranges with `spec.loadBalancerSourceRanges` (where your LB implementation supports it) or a NetworkPolicy.
- Keep the LoadBalancer on an internal-only address pool.

## Availability

- 2 replicas, spread across nodes with `topologySpreadConstraints`. The spread uses `ScheduleAnyway`, so on a single-node cluster both pods still schedule.
- Rolling updates use `maxUnavailable: 0` and `maxSurge: 1`, so there is no gap in serving.
- The PodDisruptionBudget keeps at least 1 pod running during node drains.
- Readiness and liveness probes hit `/healthz`, which is served by nginx directly and excluded from the access log.

## Troubleshooting

| Symptom | Check |
|---|---|
| `EXTERNAL-IP` stuck at `<pending>` | No LB implementation. Run `kubectl get pods -A \| grep -Ei 'metallb\|kube-vip'`, or switch to NodePort. |
| Pods rejected at admission | `kubectl -n gpu-dashboard get events`. A Pod Security violation means the securityContext was changed. |
| `CrashLoopBackOff` | `kubectl -n gpu-dashboard logs deploy/gpu-dashboard`. The usual causes are an nginx syntax error or a write outside `/tmp`. |
| 404 on `/` | The `dashboard-html` ConfigMap is missing `index.html`. Find it with `kubectl -n gpu-dashboard get cm`, then run `describe` on the `dashboard-html-*` entry. |
| Edits not showing | Re-run `kubectl apply -k .`, not `kubectl apply -f`. Then hard-refresh the browser. |
| `ImagePullBackOff` | The nodes can't reach Docker Hub. Mirror the image and update `image:`. |

Test the nginx config in the running pod:

```bash
kubectl -n gpu-dashboard exec deploy/gpu-dashboard -- nginx -t
```

## Removal

```bash
kubectl delete -k .
```

This deletes the namespace and everything in it.
