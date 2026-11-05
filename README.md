# WSO2 Identity Server 7.3 + DPDP accelerator on Kubernetes

Deploys WSO2 Identity Server 7.3 (U2 updates) with the
[DPDP accelerator](https://github.com/wso2/dpdp-accelerator) 1.0.0 on Kubernetes, using WSO2's
official Helm chart. Built for the Nutanix Kubernetes Platform (NKP), and tested locally on
Rancher Desktop first.

```
Browser ──► Traefik (IngressRoute, sticky cookie) ──► IS pod 1 ─┐
                                                      IS pod 2 ─┴──► MySQL (StatefulSet) ──► PVC
```

| Path | What it is |
| --- | --- |
| `kubernetes-is/` | WSO2's [kubernetes-is](https://github.com/wso2/kubernetes-is) chart, branch `7.3.x` at `a561463`, with one change (below) |
| `values-dpdp.yaml` | Image, 2 replicas, anti-affinity, MySQL datasources, and the accelerator's settings (`extraConfigs`) |
| `values-secrets.example.yaml` | Admin and database credentials template. Copy it to `values-secrets.yaml`, which git ignores |
| `mysql.yaml` | MySQL StatefulSet with a 20 Gi PVC, and the `mysql` Service IS connects to |
| `ingressroute.yaml` | Traefik IngressRoute: sticky sessions, HTTPS to the pods |
| `local/` | Rancher Desktop overrides for the same deployment |

## How it fits together

**One change to the WSO2 chart.** Pod anti-affinity was hard-coded to
`topology.kubernetes.io/zone`. On a cluster where every node reports the same zone, that does not
keep the two IS pods on different nodes. `templates/deployment.yaml` now reads the key from
`deployment.antiAffinityTopologyKey` (the default is unchanged), and `values-dpdp.yaml` sets it to
`kubernetes.io/hostname`.

**The accelerator's settings are in `values-dpdp.yaml`.** The chart generates `deployment.toml`
from its values and copies it over the one in the image, so the accelerator's own block (everything
below the banner in its `wso2is-7.3.0-deployment.toml`) is carried in
`deploymentToml.extraConfigs`. `WSO2DPDP_DB` reuses the identity database's credentials through
`$ref{}`, so passwords appear only in `values-secrets.yaml`.

**The chart's Ingress is disabled.** It carries nginx-only annotations. Traefik takes sticky
sessions and backend HTTPS on an IngressRoute instead.

## Images

Two images, built for `linux/amd64` from the IS + DPDP Docker setup in `solutions-dpdp/deployment`.
They are kept in one **private** Docker Hub repository, `<DOCKERHUB_USER>/wso2-dpdpiam`, because the
base image carries licensed WSO2 U2 updates:

| Tag | Contents |
| --- | --- |
| `1.0.0-is7.3.0.<U2 level>` | `registry.wso2.com/wso2-is/is:7.3.0.0` + DPDP accelerator 1.0.0 + MySQL JDBC driver |
| `1.0.0-is7.3.0.<U2 level>-mysql` | MySQL 8.4 that seeds the IS, consent v2 and DPDP schemas on first start |

The MySQL image creates the database account IS uses with the credentials it was **built** with.
`values-secrets.yaml` must use the same ones.

## Deploy on NKP

Fill in the placeholders first: `<IS_HOSTNAME>` in `values-dpdp.yaml` and `ingressroute.yaml`, the
image tags in `values-dpdp.yaml` and `mysql.yaml`, and `values-secrets.yaml`.

```powershell
$env:KUBECONFIG="<path to the workload cluster kubeconfig>"

kubectl create namespace wso2-dpdp

# Read-only Docker Hub access token
kubectl create secret docker-registry dockerhub-pull -n wso2-dpdp `
  --docker-server=https://index.docker.io/v1/ --docker-username=<DOCKERHUB_USER> --docker-password=<READ_ONLY_TOKEN>

# MySQL root password (runtime only)
kubectl create secret generic mysql-root -n wso2-dpdp --from-literal=password=<MYSQL_ROOT_PASSWORD>

# 1. Database. Wait for 1/1 Ready: the first start seeds the schema.
kubectl apply -n wso2-dpdp -f mysql.yaml
kubectl get pods -n wso2-dpdp -w

# 2. Identity Server. Expect two pods, on different nodes.
helm upgrade --install wso2is .\kubernetes-is -n wso2-dpdp -f values-dpdp.yaml -f values-secrets.yaml
kubectl get pods -n wso2-dpdp -o wide -w

# 3. Route
kubectl apply -n wso2-dpdp -f ingressroute.yaml
```

Then open `https://<IS_HOSTNAME>/console` and sign in as the admin from `values-secrets.yaml`. The
consent portal is at `/consent-portal`.

**Before step 3, check how the cluster's Traefik selects routes:**

```powershell
kubectl get deploy -A -o yaml | findstr /i "ingressclass"
```

If Traefik is started with an ingress class, `ingressroute.yaml` needs the matching
`kubernetes.io/ingress.class` annotation (it carries `kommander-traefik`). If it is not, remove the
annotation, or Traefik ignores the route.

## Run locally (Rancher Desktop)

Needs Kubernetes enabled, about 6 CPUs and 8 GB for the VM, and the two images built locally (the
`local/` files name them).

```sh
kubectl config use-context rancher-desktop
kubectl create namespace wso2-dpdp
kubectl create secret generic mysql-root -n wso2-dpdp --from-literal=password=<MYSQL_ROOT_PASSWORD>
kubectl apply -n wso2-dpdp -f local/mysql-local.yaml
helm upgrade --install wso2is ./kubernetes-is -n wso2-dpdp \
  -f values-dpdp.yaml -f local/values-local.yaml -f values-secrets.yaml
kubectl apply -n wso2-dpdp -f local/ingressroute-local.yaml
```

Traefik answers on port 443 of the Mac. If something else already holds 443, forward another port:

```sh
kubectl port-forward -n kube-system svc/traefik 8443:443
curl -k https://wso2is-127-0-0-1.sslip.io:8443/api/health-check/v1.0/health
```

## Checks

```sh
kubectl get pods,svc,pvc,pdb -n wso2-dpdp -o wide
kubectl logs -n wso2-dpdp deploy/wso2is-identity-server | grep -E "ERROR|Member joined|Carbon started"
curl -k https://<IS_HOSTNAME>/oauth2/token/.well-known/openid-configuration
kubectl rollout restart deploy/wso2is-identity-server -n wso2-dpdp     # rolling redeploy
```

Traefik routes by hostname, so a request to the IP alone returns 404. To call it by IP, send the
hostname yourself:

```sh
curl -k -H "Host: <IS_HOSTNAME>" https://<INGRESS_IP>/api/health-check/v1.0/health
```

## Known limits

- MySQL is a single instance. If its node goes down uncleanly, Kubernetes does not move the pod on
  its own. Taint the node `node.kubernetes.io/out-of-service=nodeshutdown:NoExecute` to release
  the volume. A production deployment would use an external, highly available database.
- The keystores and the `[encryption]` key are the chart's defaults: fine for a test cluster, not
  for production.
- TLS at Traefik uses its default certificate. Issue a proper one (for example with cert-manager)
  if browsers must trust it.
