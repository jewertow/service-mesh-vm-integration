# Integrating an External VM with an Ambient Mesh on OpenShift

Connect an external virtual machine to an Istio ambient mesh. The VM runs **ztunnel in dedicated mode inside a Docker container** and connects to the control plane through an east-west gateway. No application runs on the VM — the goal of this lab is to show a ztunnel outside Kubernetes authenticating to istiod, obtaining a workload certificate, and receiving the mesh state over XDS.

This lab uses KubeVirt to provision the VM in a dedicated OpenShift cluster, but the mesh is agnostic to how the VM is provisioned.

## How it works

ztunnel has two proxy modes. The default `shared` mode is inseparable from in-pod operation: it serves many workloads by binding its (hardcoded) listener ports inside each workload's network namespace, and it authenticates to the CA by *impersonating* the workload identity. That impersonation path is guarded by istiod's node authorizer, which is Pod-shaped and cannot be satisfied from a VM.

`dedicated` mode serves exactly one workload, declared up front via `PROXY_WORKLOAD_INFO=<namespace>/<name>/<service-account>`. It does **not** use impersonation, so istiod never consults the node authorizer — an ordinary service account token is enough. That is what makes running ztunnel on a VM possible.

Two consequences shape the setup below:

- **The certificate SAN comes from the token, not from `PROXY_WORKLOAD_INFO`.** istiod derives the SPIFFE ID from the authenticated caller. You must present a token for the *workload's* service account, and set the third field of `PROXY_WORKLOAD_INFO` to that same service account, or L4 authorization policies will silently not match.
- **One ztunnel serves one identity.** Anything that reaches its outbound port is attributed to the declared workload, regardless of who actually sent it. There is no per-source demultiplexing in dedicated mode.

> [!NOTE]
> Dedicated mode is documented upstream as a local-development convenience and is not covered by Istio's integration tests — every ambient suite calls `SkipVMs()`. Expect rough edges.

## Scope

In scope: ztunnel on the VM authenticates to istiod, is issued a workload certificate, and receives workloads and services over XDS.

Out of scope: the VM data path. Nothing on the VM sends traffic through ztunnel, no traffic redirection (iptables) is configured, and the cluster-to-VM HBONE path over the east-west gateway is not set up. See [Beyond this lab](#beyond-this-lab).

## Prerequisites

- OpenShift cluster with OpenShift Service Mesh 3 operator installed
- `kubectl` CLI tool installed
- An external VM or another OpenShift cluster with OpenShift Virtualization installed

## Mesh setup

### Step 1: Install IstioCNI

Create the `istio-cni` namespace and deploy the `IstioCNI` resource with the `ambient` profile:

```bash
kubectl create namespace istio-cni
kubectl apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: IstioCNI
metadata:
  name: default
spec:
  profile: ambient
  version: v1.30-latest
  namespace: istio-cni
EOF
```

Verify IstioCNI is ready:

```bash
kubectl get istiocni default
```

### Step 2: Install the Istio control plane

Create the `istio-system` and `ztunnel` namespaces with the network topology and discovery labels:

```bash
kubectl create namespace istio-system
kubectl label namespace istio-system topology.istio.io/network=cluster-network istio-discovery=enabled
kubectl create namespace ztunnel
kubectl label namespace ztunnel istio-discovery=enabled
```

> [!NOTE]
> When `discoverySelectors` is in use, both the `Istio` and the `ZTunnel` namespaces must carry the discovery label *before* the components are installed, otherwise ztunnel hits a [TLS signing error](https://github.com/istio/istio/issues/52057).

Deploy the `Istio` resource with the `ambient` profile:

```bash
kubectl apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  profile: ambient
  version: v1.30-latest
  namespace: istio-system
  updateStrategy:
    type: InPlace
  values:
    pilot:
      trustedZtunnelNamespace: ztunnel
    meshConfig:
      accessLogFile: /dev/stdout
      discoverySelectors:
      - matchLabels:
          istio-discovery: enabled
    global:
      network: cluster-network
EOF
```

Verify the control plane is ready:

```bash
kubectl get istio default
```

### Step 3: Install the in-cluster ztunnel

```bash
kubectl apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: ZTunnel
metadata:
  name: default
spec:
  version: v1.30-latest
  namespace: ztunnel
  targetRef:
    kind: Istio
    name: default
EOF
```

Verify ztunnel is ready:

```bash
kubectl get ztunnel default
```

### Step 4: Deploy the east-west gateway

Create a Kubernetes Gateway API resource to expose the control plane to the VM:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eastwestgateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
    - name: tls-istiod
      port: 15012
      protocol: TLS
      tls:
        mode: Passthrough
EOF
```

Verify the gateway is ready and has an external address:

```bash
kubectl get gateway eastwestgateway -n istio-system
```

### Step 5: Expose the control plane to the VM

TLSRoute is not supported in OCP 4.22, so we use a mix of APIs: the Kubernetes Gateway API deploys the actual gateway workload (Step 4), while the Istio Gateway and VirtualService APIs configure routing to istiod.

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: eastwestgateway
  namespace: istio-system
spec:
  selector:
    gateway.networking.k8s.io/gateway-name: eastwestgateway
  servers:
  - port:
      name: tls-istiod
      number: 15012
      protocol: tls
    tls:
      mode: PASSTHROUGH
    hosts:
    - istiod.istio-system.svc
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: istiod-vs
  namespace: istio-system
spec:
  hosts:
  - istiod.istio-system.svc
  gateways:
  - eastwestgateway
  tls:
  - match:
    - port: 15012
      sniHosts:
      - istiod.istio-system.svc
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 15012
EOF
```

> [!IMPORTANT]
> Routing here is by SNI, so the VM's ztunnel must connect to the name `istiod.istio-system.svc`, not to the gateway's IP address. The VM configuration maps that name to the gateway address in the container's `/etc/hosts`. Pointing `XDS_ADDRESS` straight at an IP sends no SNI at all and the gateway will drop the connection.

### Step 6: Deploy httpbin into the ambient mesh

This gives the VM's ztunnel something to discover over XDS. Nothing on the VM will call it.

```bash
kubectl create namespace httpbin
kubectl label namespace httpbin istio-discovery=enabled istio.io/dataplane-mode=ambient
kubectl apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
```

Verify httpbin is running:

```bash
kubectl get pods -n httpbin
```

### Step 7: Register the VM as a workload

Create a namespace for the VM workload, label it for Istio discovery, and create the service account whose identity the VM will assume:

```bash
kubectl create namespace external-vm
kubectl label namespace external-vm istio-discovery=enabled
kubectl create serviceaccount vm -n external-vm
```

Export the VM's IP address as seen on its own network. With the KubeVirt masquerade networking used in [VM_PROVISIONING.md](VM_PROVISIONING.md) this is `10.0.2.2`:

```bash
export VM_IP=10.0.2.2
```

> [!NOTE]
> If the VM does not exist yet, jump to [VM provisioning](#vm-provisioning-optional) and come back — Step 5 there reads the address off the VM and exports `VM_IP`.

Create the `WorkloadEntry`:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: WorkloadEntry
metadata:
  name: vm-1
  namespace: external-vm
  labels:
    app: vm
    networking.istio.io/tunnel: http
spec:
  address: ${VM_IP}
  network: vm-network
  serviceAccount: vm
EOF
```

Two details matter:

- **`networking.istio.io/tunnel: http`** is what makes istiod mark the workload `NativeTunnel`, i.e. HBONE-capable, so in-cluster ztunnels would dial the VM's own address on 15008. Auto-registration will not add this label for you — that path only fires when a *sidecar* registers itself, and a ztunnel connects as node type `Ztunnel`.
- **The `WorkloadEntry` name, namespace and service account must match `PROXY_WORKLOAD_INFO`** exactly. ztunnel looks the workload up by that triple and refuses traffic if it does not arrive within 5 seconds.

> [!NOTE]
> A `WorkloadEntry`-derived workload carries no node name, so the VM's ztunnel must leave `NODE_NAME` **unset**. ztunnel only indexes workloads by identity for its *local node*; with `NODE_NAME` unset it indexes all of them, and the lookup succeeds. Setting `NODE_NAME` to anything non-empty makes the lookup fail with `UnknownSourceWorkload`.

### Step 8: Generate the VM configuration

Resolve the east-west gateway address:

```bash
export EW_GATEWAY_ADDR=$(kubectl get svc eastwestgateway-istio -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If the load balancer provides a hostname instead of an IP (e.g. on AWS), resolve it to an address — the VM needs a literal IP for the `/etc/hosts` entry:

```bash
export EW_GATEWAY_ADDR=$(getent hosts "$(kubectl get svc eastwestgateway-istio -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')" | awk '{print $1; exit}')
```

Write the two files the VM needs — the mesh root certificate and a service account token:

```bash
mkdir -p ztunnel-vm-config

kubectl get configmap istio-ca-root-cert -n external-vm \
  -o jsonpath='{.data.root-cert\.pem}' > ztunnel-vm-config/root-cert.pem

kubectl create token vm -n external-vm --audience=istio-ca --duration=24h \
  > ztunnel-vm-config/istio-token
```

> [!IMPORTANT]
> The token must be issued for the **`vm` service account**, because istiod derives the certificate SAN from the authenticated caller. A token for some other service account yields a certificate for that other identity and no error.
>
> The token is short-lived; the API server may also cap `--duration` below what you asked for. ztunnel re-reads the file on every request, so refreshing it is a matter of rewriting `/etc/ztunnel/istio-token` on the VM — no restart needed. Check the granted lifetime with:
>
> ```bash
> jq -R 'split(".")[1] | @base64d | fromjson | .exp | todate' ztunnel-vm-config/istio-token
> ```

Pick the ztunnel image. Matching the in-cluster version is the simplest way to stay consistent:

```bash
export ZTUNNEL_IMAGE=$(kubectl get daemonset ztunnel -n ztunnel -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "${ZTUNNEL_IMAGE}"
```

> [!NOTE]
> If the VM cannot pull from the cluster's registry (e.g. `registry.redhat.io` without credentials), use the upstream image instead: `export ZTUNNEL_IMAGE=registry.istio.io/release/ztunnel:1.30.1`.

## VM provisioning (optional)

This step is optional. If you already have a VM, skip to [Configuring VM](#configuring-vm). For instructions on provisioning a VM with KubeVirt, see [VM_PROVISIONING.md](VM_PROVISIONING.md).

## Configuring VM

- [Manual configuration](VM_MANUAL_CONFIGURATION.md)
- [Ansible configuration](VM_ANSIBLE_CONFIGURATION.md)

## Verifying the integration

### ztunnel is running and ready

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker ps --filter name=ztunnel"
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "curl -sf localhost:15021/healthz/ready && echo READY"
```

### ztunnel connected to istiod and received the mesh state

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker logs ztunnel 2>&1 | tail -30"
```

A healthy startup ends with the proxy binding its listeners and the XDS stream reporting an initial sync:

```
2026-09-30T10:14:02.118731Z  INFO ztunnel::app: proxy mode enabled
2026-09-30T10:14:02.301884Z  INFO ztunnel::xds::client: received initial xds state
2026-09-30T10:14:02.302511Z  INFO ztunnel::proxy::inbound: listener established address=[::]:15008
2026-09-30T10:14:02.302644Z  INFO ztunnel::proxy::outbound: listener established address=[::]:15001
```

Confirm the mesh state actually arrived — httpbin from Step 6 should be there:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "curl -s localhost:15000/config_dump | jq -r '.workloads[] | \"\(.namespace)/\(.name)\"'"
```

Expected output includes the httpbin pod, the in-cluster ztunnels, and the VM's own `WorkloadEntry`:

```
external-vm/vm-1
httpbin/httpbin-5cbd68b9c7-2xj6f
istio-system/eastwestgateway-istio-5f7b9d6c84-nmk2q
ztunnel/ztunnel-7pk9d
```

If `external-vm/vm-1` is missing, ztunnel cannot resolve its own identity — recheck `PROXY_WORKLOAD_INFO` against the `WorkloadEntry`, and that `NODE_NAME` is unset.

### istiod issued a workload certificate

ztunnel fetches its certificate lazily, on the first connection it has to secure, so `config_dump` shows no certificates until something exercises the data path. Open a TLS connection to the inbound HBONE port to force it:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "openssl s_client -connect localhost:15008 -alpn h2 </dev/null 2>/dev/null | openssl x509 -noout -text | grep -A1 'Subject Alternative Name'"
```

Expected output:

```
X509v3 Subject Alternative Name: critical
    URI:spiffe://cluster.local/ns/external-vm/sa/vm
```

The handshake itself fails afterwards — ztunnel requires a client certificate and `s_client` presents none — but the server certificate is sent first, which is all we need. This is the end-to-end proof: the CSR went to istiod over the east-west gateway, was authenticated with the service account token, and came back signed with the workload's SPIFFE ID.

The certificate is now in ztunnel's cache:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "curl -s localhost:15000/config_dump | jq '.certificates'"
```

### The gateway saw the connection

```bash
kubectl logs -n istio-system -l gateway.networking.k8s.io/gateway-name=eastwestgateway --tail=20
```

You should see a long-lived connection to istiod:

```
[2026-09-30T10:14:02.101Z] "- - -" 0 - - - "-" 52316 135070 687977 - "-" "-" "-" "-" "10.131.0.36:15012" outbound|15012||istiod.istio-system.svc.cluster.local 10.128.2.22:53452 10.128.2.22:15012 100.64.0.2:43315 istiod.istio-system.svc -
```

## Beyond this lab

Making the VM actually carry traffic needs three more things, none of which are configured here:

1. **Outbound redirection on the VM.** Dedicated mode has no netns to capture traffic from, so redirection is UID-based — the same `--uid-owner` technique `istio-iptables` uses for VM sidecars. Run the application as a different user from ztunnel and redirect that user's traffic to port 15001. Port 15008 (inbound HBONE) must never be redirected.
2. **Inbound reachability.** The cluster's ztunnels dial the VM's address on 15008 directly. The VM's firewall must allow it, and the address in the `WorkloadEntry` must be routable from the cluster — which, with KubeVirt masquerade networking, it is not.
3. **A cross-network path.** The VM is on `vm-network` and the cluster on `cluster-network`. Ambient cross-network traffic goes HBONE over an east-west gateway on port 15008; this lab only exposes 15012.

A note on scaling this: **one dedicated ztunnel serves one identity, and pointing two workloads at it does not fail loudly — it gives both the same certificate.** If you need per-workload identity on a VM, run one ztunnel per workload in its own network namespace. If you do not, a single `WorkloadEntry` for the whole VM is by far the least machinery.

Worth weighing as an alternative: a VM running the ordinary Istio **sidecar** with `ISTIO_META_ENABLE_HBONE=true` already participates in an ambient mesh as a native HBONE peer, on a tested code path, and auto-registration applies the tunnel label for you. It costs an Envoy per VM but requires no unsupported configuration.
