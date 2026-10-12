# Istio Installation and Configuration

Overall installation for Istio for all environments (i.e. locally, sandbox and production).

## Prerequisites

Download from the main Istio.io website the version you intend to run. This repo ships the
`istio-1.30.2` and `istio-1.31.0` bundles, and the cluster currently runs 1.31.x (the HelmReleases
track a semver range, so check `kubectl -n istio-system get deploy istiod -o jsonpath='{.spec.template.spec.containers[0].image}'`
for what is actually deployed).
```shell
curl -L https://istio.io/downloadIstio | sh -
```

Install the Authentication Plugin from the GCloud tooling and validate:
```shell
gcloud components install gke-gcloud-auth-plugin

gke-gcloud-auth-plugin version
```


## Local

To install Istio locally, execute the following commands.

```shell
kubectl apply -f namespace.yaml

istioctl install -f local.istio-operator.yaml -y --set meshConfig.accessLogFile=/dev/stdout
# similar installation to the line below, but with a YAML manifest to specify the profile.
istioctl install --set profile=demo -y  --set meshConfig.accessLogFile=/dev/stdout # Do NOT use; given as a reference only.

# install the local configuration
kubectl apply -f local/local.ingress-gateway.yaml
kubectl apply -f base/ingressclass.yaml
kubectl apply -f local/local.ext-authz-oauth2-proxy.yaml
kubectl apply -f local/local.requestauthentication.yaml
kubectl apply -f base/peerauthentication.yaml
#kubectl apply -f local.domain-traffic-rerouting.yaml

kubectl label namespace default istio-injection=enabled
```

After an upgrade, the injected namespaces need a restart for their sidecars to pick up the
new proxy. Restart StatefulSets as well as Deployments: a Deployment-only restart leaves
databases and Redis on the old proxy.
```shell
kubectl rollout restart deployment,statefulset --namespace default
kubectl rollout restart deployment,statefulset --namespace keycloak
kubectl rollout restart deployment,statefulset --namespace oauth2-proxy
kubectl rollout restart deployment --namespace istio-system -l 'app in (istio-ingressgateway,istio-egressgateway)'
```
Check with `istio-release-qualification/run.sh --only IBT-001`: a WARN lists any workload
still on the previous proxy.


## Sandbox


```shell
export BOUCIO_CLSTR_FULLNAME=<cluster_FULL_name>
export BOUCIO_CLSTR_SHORTNAME=<cluster_SHORT_name>
kubectl apply -f namespace.yaml --cluster $BOUCIO_CLSTR_FULLNAME

istioctl install -f sandbox.istio-operator.yaml -y --set meshConfig.accessLogFile=/dev/stdout --context $BOUCIO_CLSTR_FULLNAME

kubectl apply -f sandbox/sandbox.ingress-gateway.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f base/ingressclass.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f sandbox/sandbox.ext-authz-oauth2-proxy.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f sandbox/sandbox.requestauthentication.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f base/peerauthentication.yaml --cluster $BOUCIO_CLSTR_FULLNAME

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
helm upgrade -install istio-base ./istio-1.31.0/manifests/charts/base -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istiod ./istio-1.31.0/manifests/charts/istio-control/istio-discovery -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istio-ingress ./istio-1.31.0/manifests/charts/gateways/istio-ingress -n istio-system --set global.jwtPolicy=first-party-jwt
helm upgrade -install istio-egress ./istio-1.31.0/manifests/charts/gateways/istio-egress -n istio-system --set global.jwtPolicy=first-party-jwt
```


## Mesh mTLS

There are no DestinationRules in this repo any more. mTLS comes from two things:

1. `base/peerauthentication.yaml` (STRICT), which makes sidecars in its namespace reject plaintext.
2. Istio **auto-mTLS**, on by default and not overridden anywhere here. A sidecar sends mTLS to any
   destination whose endpoints have a sidecar, and plaintext to anything without one.

That covers both in-mesh traffic and the exceptions. Sidecar-less destinations (the hostNetwork
OpenTelemetry collector) and external hosts (cloud LLM providers, MCP servers, `registry.ollama.ai`)
get plaintext or plain TLS passthrough without needing a rule.

A blanket `boucio-enable-mtls` DestinationRule (`host: "*.local"`, `ISTIO_MUTUAL`) used to force
mTLS by hand, with a growing list of per-host `DISABLE` exceptions beside it. It was removed on
2026-10-03: it duplicated auto-mTLS, it broke a traffic path twice (external egress, then the OTel
collector), and Kiali flagged it as KIA0205 because a mesh-wide DestinationRule needs a mesh-wide
PeerAuthentication in `istio-system`, which this setup does not have. Do not reintroduce one: scope
any new DestinationRule to a single host.

## FluxCD GitOps

These days Istio is delivered by FluxCD rather than manual kubectl/istioctl. The Flux repo pulls this directory as a
Git submodule and reconciles manifests through environment-specific Kustomizations:

- `components/istio/base/` – mesh-wide resources (PeerAuthentication, IngressClass). The mesh
  config itself (`meshConfig`, including `outboundTrafficPolicy`) is not here: it is single-sourced
  in the istiod HelmRelease values, because a raw ConfigMap alongside the HelmRelease got clobbered
  on every reconcile.
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

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
