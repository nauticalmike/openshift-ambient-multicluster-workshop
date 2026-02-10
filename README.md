# Openshift Specific Ambient Multi-Cluster Workshop

In this workshop, you will set up Istio Ambient in a multi-cluster OpenShift environment, deploy a sample Bookinfo app, and explore how Solo.io Ambient enables secure service-to-service communication in all directions.

![High Level Architecture](./images/high-level-architecture-1.png)

# Objectives
- Adjust OpenShift related configuration
- Deploy Istio on Openshift with Helm based install
- Deploy sample microservices on two different namespaces (bookinfo app)
- Install Istio Ambient Mesh
- Enable Ambient Mesh for app namespaces (bookinfo-frontends and bookinfo-backends)
- Setup Ingress Gateway to access the bookinfo frontend (productpage)
- Link the two clusters
- Configure a globally available service using labels (productpage)
- Reconfigure ingress to global service hostname (*.<namespace>.mesh.internal)
- Establish zero-trust security using mesh access control policies
- Egress Control
- Install Gloo Mesh control plane for multicluster UI

# Use Cases
- Zero Trust (mTLS)
- Ingress
- Observability
- Multi-cluster routing
- Global service discovery
- High Availability
- Failover

## Validated on
- OpenShift 4.18.21
- Istio 1.28.3-patch0-solo
- Solo Enterprise for Istio 2.11

## Prereqs

- Two OpenShift clusters
- A valid license key
- CLI tools like Helm, istioctl, meshctl
- A kubernetes version >1.29

### Env

1. Create two OpenShift clusters and set the env vars below to their context

```bash
export CLUSTER1=openshift_cluster_one # UPDATE THIS
export CLUSTER2=openshift_cluster_two # UPDATE THIS
export GLOO_MESH_LICENSE_KEY=<update>  # UPDATE THIS

export ISTIO_VERSION=1.28.3-patch0
export REPO_KEY=594e990587b9
export ISTIO_IMAGE=${ISTIO_VERSION}-solo
export REPO=us-docker.pkg.dev/gloo-mesh/istio-${REPO_KEY}
export HELM_REPO=us-docker.pkg.dev/gloo-mesh/istio-helm-${REPO_KEY}
```

2. Download Solo's `istioctl` Binary:
```bash
OS=$(uname | tr '[:upper:]' '[:lower:]' | sed -E 's/darwin/osx/')
ARCH=$(uname -m | sed -E 's/aarch/arm/; s/x86_64/amd64/; s/armv7l/armv7/')

mkdir -p ~/.istioctl/bin
curl -sSL https://storage.googleapis.com/istio-binaries-${REPO_KEY}/${ISTIO_VERSION}-solo/istioctl-${ISTIO_VERSION}-solo-${OS}-${ARCH}.tar.gz | tar xzf - -C ~/.istioctl/bin
chmod +x ~/.istioctl/bin/istioctl

export PATH=${HOME}/.istioctl/bin:${PATH}
```

3. Verify using `istioctl version`

```sh
istioctl version --remote=false
```

### Clean up previous Istio installations

```bash
kubectl --context=$CLUSTER1 delete smc --all
kubectl --context=$CLUSTER2 delete smc --all
istioctl uninstall --purge -y --context $CLUSTER1
istioctl uninstall --purge -y --context $CLUSTER2
```

### OpenShift OVN

Keep in mind OCP OVN-k (CNI) has a default routing mode `shared` and we need to make sure it gets switched to `local`:

```sh
oc patch networks.operator.openshift.io cluster --type=merge -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"gatewayConfig":{"routingViaHost": true}}}}}'
```

More information can be found [here](https://istio.io/latest/docs/ambient/install/platform-prerequisites/#red-hat-openshift).

## Deploy Bookinfo Application across two namespaces on both clusters

Openshift SCC:

```bash
oc --context ${CLUSTER1} adm policy add-scc-to-group anyuid system:serviceaccounts:bookinfo-frontends
oc --context ${CLUSTER1} adm policy add-scc-to-group anyuid system:serviceaccounts:bookinfo-backends
oc --context ${CLUSTER2} adm policy add-scc-to-group anyuid system:serviceaccounts:bookinfo-frontends
oc --context ${CLUSTER2} adm policy add-scc-to-group anyuid system:serviceaccounts:bookinfo-backends
```

Deploy bookinfo frontends in bookinfo-frontends namespace:

```bash
kubectl apply -f bookinfo/bookinfo-frontends.yaml --context $CLUSTER1
kubectl apply -f bookinfo/bookinfo-frontends.yaml --context $CLUSTER2
```

Deploy bookinfo backends in bookinfo-backends namespace:

```bash
kubectl apply -f bookinfo/bookinfo-backends.yaml --context $CLUSTER1
kubectl apply -f bookinfo/bookinfo-backends.yaml --context $CLUSTER2
```

Wait for the applications to be deployed:

```bash
for deploy in $(kubectl get deploy -n bookinfo-frontends --context $CLUSTER1 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for frontend deployment '$deploy' to be ready in $CLUSTER1..."
  kubectl rollout status deploy/"$deploy" -n bookinfo-frontends --watch --timeout=90s --context $CLUSTER1
  done

for deploy in $(kubectl get deploy -n bookinfo-backends --context $CLUSTER1 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for backend deployment '$deploy' to be ready in $CLUSTER1..."
  kubectl rollout status deploy/"$deploy" -n bookinfo-backends --watch --timeout=90s --context $CLUSTER1
  done

for deploy in $(kubectl get deploy -n bookinfo-frontends --context $CLUSTER2 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for frontend deployment '$deploy' to be ready in $CLUSTER2..."
  kubectl rollout status deploy/"$deploy" -n bookinfo-frontends --watch --timeout=90s --context $CLUSTER2
  done

for deploy in $(kubectl get deploy -n bookinfo-backends --context $CLUSTER2 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for backend deployment '$deploy' to be ready in $CLUSTER2..."
  kubectl rollout status deploy/"$deploy" -n bookinfo-backends --watch --timeout=90s --context $CLUSTER2
  done
```

Update the reviews service to display which cluster it is coming from:

```bash
kubectl --context ${CLUSTER1} -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=CLUSTER1
kubectl --context ${CLUSTER1} -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=CLUSTER1
kubectl --context ${CLUSTER1} -n bookinfo-backends set env deploy/reviews-v3 CLUSTER_NAME=CLUSTER1
kubectl --context ${CLUSTER2} -n bookinfo-backends set env deploy/reviews-v1 CLUSTER_NAME=CLUSTER2
kubectl --context ${CLUSTER2} -n bookinfo-backends set env deploy/reviews-v2 CLUSTER_NAME=CLUSTER2
kubectl --context ${CLUSTER2} -n bookinfo-backends set env deploy/reviews-v3 CLUSTER_NAME=CLUSTER2
```

Port forward to productpage in bookinfo-frontends namespace to validate application is working:

```bash
kubectl port-forward svc/productpage -n bookinfo-frontends 9080:9080 --context $CLUSTER1
```

Navigate to http://localhost:9080/productpage:

Do the same on another terminal session:

```bash
kubectl port-forward svc/productpage -n bookinfo-frontends 9080:9080 --context $CLUSTER2
```

Navigate to http://localhost:9080/productpage

## Istio Install

Normally Istio OSS recommends to use `platform: openshift` profile when installing Istio components. This helps set the proper configuration needed for CNI and zTunnel when installed in the `kube-system` namespace as the OSS recommends. The problem is that most enterprises consider this a risk and prevent any workloads running on the `kube-system` namespace. This installation guide has the needed configuration to run Istio Ambient on OpenShift using the `istio-system` namespace and is not considering any k8s NetworkPolicies in place. More information can be found [here](https://istio.io/latest/docs/ambient/usage/networkpolicy/).

- Create istio-system namespace and shared root trust secret in cluster1

```bash
kubectl create namespace istio-system --context $CLUSTER1
kubectl label namespace istio-system topology.istio.io/network=cluster1 --context $CLUSTER1 --overwrite
kubectl apply -f shared-root-trust-secret.yaml --context $CLUSTER1
kubectl create namespace istio-system --context $CLUSTER2
kubectl label namespace istio-system topology.istio.io/network=cluster2 --context $CLUSTER2 --overwrite
kubectl apply -f shared-root-trust-secret.yaml --context $CLUSTER2
```

- Install Istio using Helm

Install Kubernetes Gateway CRDs:

```bash
kubectl get crd gateways.gateway.networking.k8s.io --context $CLUSTER1 &> /dev/null || \
  { kubectl --context $CLUSTER1 apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml; }
kubectl get crd gateways.gateway.networking.k8s.io --context $CLUSTER2 &> /dev/null || \
  { kubectl --context $CLUSTER2 apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml; }
```

Install istio:

```sh
export CLUSTER_NAME1=cluster1
export CLUSTER_NAME2=cluster2
```

```sh
# istio-base
helm upgrade -i istio-base oci://us-docker.pkg.dev/soloio-img/istio-helm/base                       \
             --namespace istio-system                                                               \
             --create-namespace                                                                     \
             --set defaultRevision=main                                                             \
             --version ${ISTIO_VERSION}-solo                                                        \
             --wait --kube-context $CLUSTER1

helm upgrade -i istio-base oci://us-docker.pkg.dev/soloio-img/istio-helm/base                       \
             --namespace istio-system                                                               \
             --create-namespace                                                                     \
             --set defaultRevision=main                                                             \
             --version ${ISTIO_VERSION}-solo                                                        \
             --wait --kube-context $CLUSTER2

# Istiod
helm upgrade --kube-context $CLUSTER1 -i istiod oci://us-docker.pkg.dev/soloio-img/istio-helm/istiod\
        --namespace istio-system                                                                    \
        --version ${ISTIO_VERSION}-solo                                                             \
        --wait                                                                                      \
        --values - <<EOF
        profile: ambient
        trustedZtunnelNamespace: istio-system
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        env:
          PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
          PILOT_ENABLE_ALPHA_GATEWAY_API: "true"
        global:
          hub: ${REPO}
          tag: ${ISTIO_IMAGE}
          meshID: mesh1
          multiCluster:
            clusterName: ${CLUSTER_NAME1}
            enabled: true
          network: ${CLUSTER_NAME1}
        meshConfig:
          defaultConfig:
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
          trustDomain: "${CLUSTER_NAME1}"
        platforms:
          peering:
            enabled: true
        license:
          value: $GLOO_MESH_LICENSE_KEY
EOF

helm upgrade --kube-context $CLUSTER2 -i istiod oci://us-docker.pkg.dev/soloio-img/istio-helm/istiod\
        --namespace istio-system                                                                    \
        --version ${ISTIO_VERSION}-solo                                                             \
        --wait                                                                                      \
        --values - <<EOF
        profile: ambient
        trustedZtunnelNamespace: istio-system
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        env:
          PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
          PILOT_ENABLE_ALPHA_GATEWAY_API: "true"
        global:
          hub: ${REPO}
          tag: ${ISTIO_IMAGE}
          meshID: mesh1
          multiCluster:
            clusterName: ${CLUSTER_NAME2}
            enabled: true
          network: ${CLUSTER_NAME2}
        meshConfig:
          defaultConfig:
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
          trustDomain: "${CLUSTER_NAME2}"
        platforms:
          peering:
            enabled: true
        license:
          value: $GLOO_MESH_LICENSE_KEY
EOF

# Istio-cni
helm upgrade --kube-context $CLUSTER1 -i istio-cni oci://us-docker.pkg.dev/soloio-img/istio-helm/cni \
        --namespace istio-system                                                                     \
        --version ${ISTIO_VERSION}-solo                                                              \
        --wait                                                                                       \
        --values - << EOF
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        profile: ambient
        ambient:
          dnsCapture: true
        excludeNamespaces:
          - istio-system
          - kube-system
        global:
          platform: openshift
          hub: us-docker.pkg.dev/soloio-img/istio
          tag: ${ISTIO_IMAGE}
          variant: distroless
EOF

helm upgrade --kube-context $CLUSTER2 -i istio-cni oci://us-docker.pkg.dev/soloio-img/istio-helm/cni \
        --namespace istio-system                                                                     \
        --version ${ISTIO_VERSION}-solo                                                              \
        --wait                                                                                       \
        --values - << EOF
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        profile: ambient
        ambient:
          dnsCapture: true
        excludeNamespaces:
          - istio-system
          - kube-system
        global:
          platform: openshift
          hub: us-docker.pkg.dev/soloio-img/istio
          tag: ${ISTIO_IMAGE}
          variant: distroless
EOF

# Ztunnel
helm upgrade --kube-context $CLUSTER1 -i ztunnel oci://us-docker.pkg.dev/soloio-img/istio-helm/ztunnel \
        --namespace istio-system                                                                       \
        --version ${ISTIO_VERSION}-solo                                                                \
        --wait                                                                                         \
        --values - << EOF
        profile: ambient
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        global:
          platform: openshift
          hub: us-docker.pkg.dev/soloio-img/istio
          tag: ${ISTIO_IMAGE}
          variant: distroless
        resources:
          requests:
            cpu: 500m
            memory: 2048Mi
        istioNamespace: istio-system
        env:
          L7_ENABLED: "true"
          SKIP_VALIDATE_TRUST_DOMAIN: "true"
        multiCluster:
          clusterName: ${CLUSTER_NAME1}
        network: ${CLUSTER_NAME1}
EOF

helm upgrade --kube-context $CLUSTER2 -i ztunnel oci://us-docker.pkg.dev/soloio-img/istio-helm/ztunnel \
        --namespace istio-system                                                                       \
        --version ${ISTIO_VERSION}-solo                                                                \
        --wait                                                                                         \
        --values - << EOF
        profile: ambient
        hub: us-docker.pkg.dev/soloio-img/istio
        tag: ${ISTIO_IMAGE}
        global:
          platform: openshift
          hub: us-docker.pkg.dev/soloio-img/istio
          tag: ${ISTIO_IMAGE}
          variant: distroless
        resources:
          requests:
            cpu: 500m
            memory: 2048Mi
        istioNamespace: istio-system
        env:
          L7_ENABLED: "true"
          SKIP_VALIDATE_TRUST_DOMAIN: "true"
        multiCluster:
          clusterName: ${CLUSTER_NAME2}
        network: ${CLUSTER_NAME2}
EOF
```

Use solo `istioctl` to create e/w peering gateways:

```bash
kubectl create namespace istio-gateways --context $CLUSTER1
istioctl multicluster expose --namespace istio-gateways --context $CLUSTER1
kubectl create namespace istio-gateways --context $CLUSTER2
istioctl multicluster expose --namespace istio-gateways --context $CLUSTER2
```

Wait for the e/w gateways to be deployed

```bash
for deploy in $(kubectl get deploy -n istio-gateways --context $CLUSTER1 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for e/w gateway deployment '$deploy' to be ready in $CLUSTER1..."
  kubectl rollout status deploy/"$deploy" -n istio-gateways --watch --timeout=90s --context $CLUSTER1
  done

for deploy in $(kubectl get deploy -n istio-gateways --context $CLUSTER2 -o jsonpath='{.items[*].metadata.name}'); do
  echo "Waiting for e/w gateway deployment '$deploy' to be ready in $CLUSTER2..."
  kubectl rollout status deploy/"$deploy" -n istio-gateways --watch --timeout=90s --context $CLUSTER2
  done
```

Link the e/w gateways gateways

```bash
istioctl multicluster link --contexts=$CLUSTER1,$CLUSTER2 --namespace istio-gateways
```

Make sure you check the status on the peered gateway to validate it was programmed.

Tail the logs of istiod for peering status:

```sh
kubectl logs -n istio-system deployments/istiod --context $CLUSTER1 | grep "peering"
```

Now look for a log entry like `info    peering cluster cluster1 sync complete`.

Now on the other cluster:

```sh
kubectl logs -n istio-system deployments/istiod --context $CLUSTER2 | grep "peering"
```

Now look for a log entry like `info    peering cluster cluster1 sync complete`.

Finally, check the status for both clusters:

```sh
istioctl multicluster check --contexts=$CLUSTER1,$CLUSTER2
```

Expect a response similar to:

```sh
=== Cluster: default/api-ocp-demo-7-mgt-openshift-cse-solo-io:6443/kube:admin ===

✅ Incompatible Environment Variable Check: all relevant environment variables are valid
✅ License Check: license is valid for multicluster
✅ Pod Check (istiod): all pods healthy
✅ Pod Check (ztunnel): all pods healthy
✅ Pod Check (eastwest gateway): all pods healthy
✅ Gateway Check: all eastwest gateways programmed
✅ Peers Check: all clusters connected


=== Cluster: default/api-ocp-demo-7-cl2-openshift-cse-solo-io:6443/kube:admin ===

✅ Incompatible Environment Variable Check: all relevant environment variables are valid
✅ License Check: license is valid for multicluster
✅ Pod Check (istiod): all pods healthy
✅ Pod Check (ztunnel): all pods healthy
✅ Pod Check (eastwest gateway): all pods healthy
✅ Gateway Check: all eastwest gateways programmed
✅ Peers Check: all clusters connected
⚠️  Stale Workloads Check: no autogenflat workload entries found
```

## Configure a global service

Enable ambient with namespace label:

```bash
kubectl label namespace bookinfo-frontends istio.io/dataplane-mode=ambient --context $CLUSTER1
kubectl label namespace bookinfo-backends istio.io/dataplane-mode=ambient --context $CLUSTER1
kubectl label namespace bookinfo-frontends istio.io/dataplane-mode=ambient --context $CLUSTER2
kubectl label namespace bookinfo-backends istio.io/dataplane-mode=ambient --context $CLUSTER2
```

```bash
for context in ${CLUSTER1} ${CLUSTER2}; do
  kubectl --context ${context}  -n bookinfo-frontends label service productpage solo.io/service-scope=global
  kubectl --context ${context}  -n bookinfo-frontends annotate service productpage networking.istio.io/traffic-distribution=Any
done
```

We should now see an automatically generated ServiceEntry referencing our global service in each cluster:

```bash
for CTX in "$CLUSTER1" "$CLUSTER2"; do
  echo "Checking for autogen global ServiceEntry in $CTX"
  kubectl --context $CTX get serviceentry -n istio-system
done
```

Expect something along the lines of:

```sh
Checking for autogen global ServiceEntry in cluster1
NAME                                     HOSTS                                              LOCATION        RESOLUTION   AGE
autogen.bookinfo-frontends.productpage   ["productpage.bookinfo-frontends.mesh.internal"]                   STATIC       17s
dummy-route-blackhole-solo-unused        ["dummy-route.blackhole.solo.unused"]              MESH_INTERNAL   STATIC       5d22h
graphql-host-blackhole-solo-unused       ["graphql-host.blackhole.solo.unused"]             MESH_INTERNAL   STATIC       5d22h
Checking for autogen global ServiceEntry in cluster2
NAME                                     HOSTS                                              LOCATION   RESOLUTION   AGE
autogen.bookinfo-frontends.productpage   ["productpage.bookinfo-frontends.mesh.internal"]              STATIC       18s
```

## Configure Ingress Gateway in cluster1 to route to global service

Deploy an Istio Ingressgateway:

```bash
kubectl apply --context $CLUSTER1 -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: gw-options
  namespace: istio-system
data:
  service: |
    metadata:
      annotations:
        key1: value1
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: http
  namespace: istio-system
spec:
  infrastructure:
    parametersRef:
      group: ""
      kind: ConfigMap
      name: gw-options
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

Reconfigure the bookinfo application to use the global service hostname `*.<namespace>.mesh.internal`

```bash
kubectl apply --context $CLUSTER1 -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: bookinfo-http
  namespace: bookinfo-frontends
spec:
  hostnames:
  - "bookinfo.glootest.com"
  parentRefs:
    - name: http
      namespace: istio-system
  rules:
    - matches:
      - path:
          type: PathPrefix
          value: /
      backendRefs:
      - kind: Hostname
        group: networking.istio.io
        name: productpage.bookinfo-frontends.mesh.internal
        port: 9080
EOF
```

Assuming that you have DNS properly configured to the LoadBalancer IP address of the ingress gateway, you should now be able to access the application at http://bookinfo.glootest.com/productpage

Otherwise you can curl it with a host header

```bash
SVC=$(kubectl -n istio-system get svc http-istio --context $CLUSTER1 --no-headers | awk '{ print $4 }')
curl -H "Host: bookinfo.glootest.com" http://$SVC/productpage
```

## Failover Bookinfo on cluster1

Scale down productpage-v1 in the `bookinfo-frontends` namespace on cluster1

```bash
kubectl scale deploy/productpage-v1 -n bookinfo-frontends --replicas 0 --context $CLUSTER1
```

Retry the curl command, you should still have success accessing the bookinfo application

```bash
SVC=$(kubectl -n istio-system get svc http-istio --context $CLUSTER1 --no-headers | awk '{ print $4 }')
curl -H "Host: bookinfo.glootest.com" http://$SVC/productpage
```

Tail logs of ztunnel on `cluster2` in a new terminal to watch logs

```bash
kubectl logs ds/ztunnel -n istio-system --context $CLUSTER2 -f
```

You should see traffic going to cluster2

## Restore Bookinfo on cluster1

Scale productpage-v1 back up in the `bookinfo-frontends` namespace on cluster1

```bash
kubectl scale deploy/productpage-v1 -n bookinfo-frontends --replicas 1 --context $CLUSTER1
```

In the ztunnel logs of cluster2 we should see that traffic is no longer routing there because productpage-v1 on cluster1 is healthy

Tail logs of ztunnel on `cluster1` in a new terminal to watch logs

```bash
kubectl logs ds/ztunnel -n istio-system --context $CLUSTER1 -f
```

If you retry the curl command you should now see traffic going back to cluster1.