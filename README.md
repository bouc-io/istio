# Istio Installation and Configuration

Overall installation for Istio for all environments (i.e. locally, sandbox and production).

## Pre-requisites

Download from the main Istio.io website the latest version (note, the current version tested is 1.10.0).
```shell
curl -L https://istio.io/downloadIstio | sh -
```

Install the Authentication Plugin from the GCloud tooling and validate:
```shell
gcloud components install gke-gcloud-auth-plugin

gke-gcloud-auth-plugin version
```


## Local (manual install)

To install Istio locally, execute the following commands.

```shell
kubectl apply -f namespace.yaml

istioctl install -f local.istio-operator.yaml -y --set meshConfig.accessLogFile=/dev/stdout
# similar installation to the line below, but with a YAML manifest to specify the profile.
istioctl install --set profile=demo -y  --set meshConfig.accessLogFile=/dev/stdout # Do NOT use; given as a reference only.

# install the local configuration
kubectl apply -f local.ingress-gateway.yaml
kubectl apply -f ingressclass.yaml
kubectl apply -f configmap.yaml
kubectl apply -f local.ext-authz-oauth2-proxy.yaml
kubectl apply -f local.requestauthentication.yaml
kubectl apply -f destinationrule.yaml
kubectl apply -f peerauthentication.yaml
#kubectl apply -f local.domain-traffic-rerouting.yaml

kubectl label namespace default istio-injection=enabled
```

For reference, during the upgrade of Istio, the current namespace requiring a restart to have the Istio upgrade along with the specifi commands are:
```shell
kubectl rollout restart deployment --namespace kubernetes-dashboard
kubectl rollout restart deployment --namespace default
kubectl rollout restart deployment --namespace keycloak
kubectl rollout restart deployment --namespace oauth2-proxy
kubectl rollout restart deployment --namespace metrics-server
kubectl rollout restart deployment --namespace prometheus
kubectl rollout restart deployment --namespace grafana

```


## Sandbox (manual install)


```shell
export BOUCIO_CLSTR_FULLNAME=<cluster_FULL_name>
export BOUCIO_CLSTR_SHORTNAME=<cluster_SHORT_name>
kubectl apply -f namespace.yaml --cluster $BOUCIO_CLSTR_FULLNAME

istioctl install -f sandbox.istio-operator.yaml -y --set meshConfig.accessLogFile=/dev/stdout --context $BOUCIO_CLSTR_FULLNAME

kubectl apply -f sandbox.ingress-gateway.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f ingressclass.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f configmap.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f sandbox.ext-authz-oauth2-proxy.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f sandbox.requestauthentication.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f destinationrule.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f peerauthentication.yaml --cluster $BOUCIO_CLSTR_FULLNAME
#kubectl apply -f sandbox.domain-traffic-rerouting.yaml --cluster $BOUCIO_CLSTR_FULLNAME

kubectl label namespace default istio-injection=enabled --cluster $BOUCIO_CLSTR_FULLNAME
```


Due to some constraints on a private GKE (see #13 below), the following commands are required:

To obtain the firewall rule name:
```shell
gcloud compute firewall-rules list --filter="name~gke-${BOUCIO_CLSTR_SHORTNAME}-[0-9a-z]*-master"
```

To add the port to the cluster:
```shell
gcloud compute firewall-rules update <firewall-rule-name> --allow tcp:10250,tcp:443,tcp:15017
```

To add the admin role to the cluster:
```shell
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account) \
    --cluster $CLUSTER_FULLNAME
```

Via Helm:

Note, the method is still valid but I personally highly recommend to use the istioctl tool available.

```shell
helm upgrade -install istio-base ./istio-1.10.3/manifests/charts/base -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istiod ./istio-1.10.3/manifests/charts/istio-control/istio-discovery -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istio-ingress ./istio-1.10.3/manifests/charts/gateways/istio-ingress -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istio-egress ./istio-1.10.3/manifests/charts/gateways/istio-egress -n istio-system --set global.jwtPolicy=first-party-jwt
```


## FluxCD GitOps (current setup)

These days Istio is delivered by FluxCD rather than manual kubectl/istioctl. The Flux repo pulls this directory as a
Git submodule and reconciles manifests through environment-specific Kustomizations:

- `components/istio/base/` – mesh-wide resources (mesh ConfigMap, default DestinationRules, PeerAuthentication, IngressClass).
- `components/istio/local/` – local-only overlays (Gateway, VirtualServices, oauth2-proxy ext-auth, etc.).
- `components/istio/sandbox/` – sandbox overlays (same idea with sandbox hosts and policies).

Flux wires them like this (local example):

```
clusters/local/flux-system/istio-base-kustomization.yaml    -> ./components/istio/base
clusters/local/flux-system/istio-local-kustomization.yaml   -> ./components/istio/local
```

Both Kustomizations depend on `local-infra` (which installs the Istio Helm charts), and the overlay depends on the base
layer. Sandbox mirrors the same pattern. This keeps platform pieces (Gateways, mesh defaults) central while application
Helm charts own their `VirtualService`/`DestinationRule` templates and reference the shared `istio-system` gateway.

If you want to apply these YAMLs manually, the sections below cover the kubectl/istioctl approach.


### References

Refer to:
1. https://istio.io/latest/docs/setup/getting-started/
2. https://istio.io/latest/docs/examples/microservices-istio/
3. https://istio.io/latest/docs/setup/install/helm/
4. https://cloud.google.com/istio/docs/istio-on-gke/installing
5. https://www.youtube.com/watch?v=voAyroDb6xk&list=WL&index=3
6. https://istio.io/latest/docs/setup/upgrade/in-place/
7. https://medium.com/@senthilrch/api-access-control-using-istio-ingress-gateway-44be659a087e
8. https://medium.com/@senthilrch/api-authentication-using-istio-ingress-gateway-oauth2-proxy-and-keycloak-a980c996c259
9. https://medium.com/@senthilrch/api-authentication-using-istio-ingress-gateway-oauth2-proxy-and-keycloak-part-2-of-2-dbb3fb9cd0d0
10. https://oauth2-proxy.github.io/oauth2-proxy/docs/
11. https://bitnami.com/stack/oauth2-proxy/helm
12. https://github.com/bitnami/charts/tree/master/bitnami/oauth2-proxy/#installing-the-chart
13. https://istio.io/latest/docs/setup/platform-setup/gke/ (Addtional config required for private GKE cluster)
14. https://istio.io/latest/docs/ops/common-problems/injection/ (To debug Istio Sidecar injection problem, pod annotation for safe to evict, prioritize the proxy network connectivity if the app requires checks while bootup)
15. https://cloud.google.com/container-registry/docs/access-control?&_ga=2.135761393.-1706233676.1695819138#public
16. https://www.linkedin.com/pulse/integrating-istio-gcp-load-balancer-serdar-yıldırım/
17. https://istio.io/latest/docs/reference/config/networking/virtual-service/#CorsPolicy
18. https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/
19. https://github.com/istio/istio/issues/27643#issuecomment-1588249182
20. https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#PodDisruptionBudgetSpec
21. https://chrishaessig.medium.com/keycloak-with-istio-and-oauth2-proxy-65227a383c15 ** PERFECT EXAMPLE!!
22. https://istio.io/latest/docs/setup/additional-setup/cni/#prerequisites
23. https://istio.io/latest/docs/setup/platform-setup/gke/
