# demo-flux

Flux GitOps for the demo apps (`nginx-web` + `whoami-api`) and kube-prometheus-stack.

App charts are published to `oci://ghcr.io/jaroprec/charts` (public; no pull secrets). Chart **versions** live in `overlays/` (not in the base HelmReleases). Alert `PrometheusRule`s live in `configs/` so they apply after Prometheus Operator CRDs exist.

## Layout

```
clusters/staging/          # example Kustomization specs (optional; CLI creates the same objects)
overlays/staging/          # cluster values: chart versions, env, ingress host
infrastructure/            # namespaces, HelmRepositories, HelmReleases
configs/                   # PrometheusRules (not inside app charts)
```

Staging overlay currently pins:

- `nginx-web` / `whoami-api` chart `2026.908.757`
- `kube-prometheus-stack` `88.6.2`
- Ingress host `demo.local`

## Prerequisites (minikube)

Give the VM enough room; kube-prometheus-stack is heavy:

```bash
minikube start --cpus=2 --memory=8192
minikube addons enable ingress
minikube addons enable metrics-server   # optional; HPA likes it
```

Install the Flux **CLI**, then Flux on the cluster (`flux install` is the same on every OS).

macOS (Homebrew):

```bash
brew install fluxcd/tap/flux
```

Linux: Homebrew works if you already use it (`brew install fluxcd/tap/flux`). Otherwise:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

Binaries: [fluxcd/flux2 releases](https://github.com/fluxcd/flux2/releases). Check with `flux version --client`.

```bash
flux install
```

## Point Flux at this repo

`flux bootstrap` is not required. A `GitRepository` plus two `Kustomization`s is enough; Flux will keep polling git and applying changes.

Replace the URL/branch if needed:

```bash
flux create source git flux-system \
  --url=https://github.com/jaroprec/demo-flux \
  --branch=main \
  --interval=1m

flux create kustomization infrastructure \
  --source=GitRepository/flux-system \
  --path=./overlays/staging \
  --prune=true \
  --wait=true \
  --interval=10m \
  --retry-interval=1m \
  --timeout=20m

flux create kustomization configs \
  --source=GitRepository/flux-system \
  --path=./configs \
  --depends-on=infrastructure \
  --prune=true \
  --interval=10m \
  --retry-interval=1m \
  --timeout=5m
```

Wait until HelmReleases are ready (Prometheus can take several minutes):

```bash
flux get kustomizations
flux get helmreleases -A
kubectl -n demo get pods,svc,ingress
kubectl -n monitoring get pods
```

### Auto-sync after a git push

Bootstrap is not involved. After you push to the branch in `--url`:

1. The source polls git (`--interval=1m`).
2. A new commit updates the source revision.
3. Flux reconciles `infrastructure` (`./overlays/staging`) then `configs` (`./configs`). A new revision triggers them; you do not wait the full 10m.

Changes under `infrastructure/`, `overlays/staging/`, and `configs/` are picked up automatically.

The `GitRepository` and `Kustomization` objects created by `flux create` live **in the cluster**. Editing `clusters/staging/*.yaml` in git does not change those objects unless you recreate them with `flux create`.

Force a pull anytime:

```bash
flux reconcile source git flux-system
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization configs --with-source
```

## Test the apps

In-cluster (always works; does not use Ingress):

```bash
kubectl -n demo exec deploy/nginx-web -- wget -qO- http://127.0.0.1:8080/
kubectl -n demo exec deploy/nginx-web -- wget -qO- http://127.0.0.1:8080/api/
```

`/` is the nginx page. `/api/` is proxied to `whoami-api` (you should see whoami headers).

### Ingress (macOS and Linux)

Flux, in-cluster `wget`, and Grafana port-forward are the same on every OS. What differs is how the **laptop** reaches minikube Ingress.

The ingress addon is a **NodePort** Service. `minikube tunnel` only publishes LoadBalancer IPs, on every OS.

**macOS (Docker driver):** `minikube ip` is usually not reachable. Use a port-forward (below).

**Linux:** with kvm2 / qemu / VirtualBox you can often skip the forward:

```bash
echo "$(minikube ip) demo.local" | sudo tee -a /etc/hosts
curl -sS http://demo.local/
curl -sS http://demo.local/api/
```

Browser: [http://demo.local/](http://demo.local/). With the **Docker** driver, try `minikube ip` first; if it hangs, use the same port-forward as on Mac.

Port-forward to the controller (works on macOS and Linux):

```bash
kubectl -n ingress-nginx get pods   # wait until Running
kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80
```

In another terminal:

```bash
curl -sS -H "Host: demo.local" http://127.0.0.1:8080/
curl -sS -H "Host: demo.local" http://127.0.0.1:8080/api/
```

In a **browser**, a custom `Host` header is not available. Map the name once, keep the port-forward running, then open the URL with port 8080:

```bash
echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts
```

Open [http://demo.local:8080/](http://demo.local:8080/) and [http://demo.local:8080/api/](http://demo.local:8080/api/). `http://127.0.0.1:8080/` will not match the Ingress host and usually returns 404.

Confirm Ingress is programmed:

```bash
kubectl get ingressclass
kubectl -n demo get ingress nginx-web -o wide
```

`ADDRESS` should be set. Empty ADDRESS means class `nginx` has no controller (`minikube addons enable ingress`).

NetworkPolicy on `nginx-web` only allows traffic from namespace `ingress-nginx`. Direct `kubectl port-forward svc/nginx-web` can fail if a CNI enforces NetworkPolicy (Calico/Cilium). Default minikube CNI often ignores it.

## Monitoring

kube-prometheus-stack is in namespace `monitoring`. App alerts are `PrometheusRule`s in `configs/`, labeled `release: kube-prometheus-stack`.

```bash
kubectl -n monitoring get prometheusrule
```

Grafana (same on macOS and Linux; change the password after first login):

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Open [http://localhost:3000](http://localhost:3000). Default user `admin`, password `prom-operator`. If that fails:

```bash
kubectl -n monitoring get secret kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

## Scratch / retry

```bash
flux suspend helmrelease nginx-web -n demo
flux resume helmrelease nginx-web -n demo
# or: helm uninstall nginx-web -n demo  then reconcile again
```

Full reset: delete HelmReleases/namespaces, `flux uninstall`, optionally `minikube delete`.
