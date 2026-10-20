# Integrating External VMs with Istio on OpenShift

Integrate external virtual machines into the Istio service mesh. Each VM runs an Istio sidecar proxy, reaches the control plane through an OpenShift passthrough Route, and reaches in-mesh workloads through an east-west gateway. This lab uses KubeVirt to provision the VMs in a dedicated OpenShift cluster, but the mesh is agnostic to how the VMs are provisioned.

On top of the basic integration, the lab demonstrates the two ways a VM workload can be made discoverable to the rest of the mesh:

| Approach | Hostname | Backed by |
|---|---|---|
| Kubernetes `Service` | `httpbin.internal.svc.cluster.local` | in-cluster pods **and** a `WorkloadEntry` for `httpbin-internal-vm` |
| `ServiceEntry` | `httpbin.external.vm` | a `WorkloadEntry` for `httpbin-external-vm` only |

A `curl` client in the `client` namespace calls both. The `Service` case shows that an ordinary Kubernetes `Service` selects pods and VMs alike, so a VM can be added to an existing service without the clients knowing. The `ServiceEntry` case shows how to publish a VM-only service under a hostname that does not exist in cluster DNS at all.

Both VMs run the same httpbin container image as `samples/httpbin/httpbin.yaml` from the Istio repository, started with podman, listening on port `8080` and published in the mesh on port `8000`.

## Prerequisites

- OpenShift cluster with OpenShift Service Mesh 3 operator installed
- `kubectl` and `istioctl` CLI tools installed
- Two external VMs, or another OpenShift cluster with OpenShift Virtualization installed
- The VMs can resolve and reach names under the cluster's ingress domain (see [Step 2](#step-2-export-the-cluster-base-domain))
- The mesh cluster can reach each VM on TCP port `8080` (see [Step 3 of VM provisioning](VM_PROVISIONING.md#step-3-expose-the-vms-with-loadbalancer-services))

## Mesh setup

### Step 1: Install IstioCNI

Create the `istio-cni` namespace and deploy the `IstioCNI` resource:

```bash
kubectl create namespace istio-cni
kubectl apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: IstioCNI
metadata:
  name: default
spec:
  version: v1.30-latest
  namespace: istio-cni
EOF
```

Verify IstioCNI is ready:

```bash
kubectl get istiocni default
```

### Step 2: Export the cluster base domain

The control plane is exposed to the VMs through an OpenShift Route, so the cluster's default ingress domain is needed in several of the following steps:

```bash
export BASE_DOMAIN=$(kubectl get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "Base domain: ${BASE_DOMAIN}"
```

> [!IMPORTANT]
> This setup assumes the VM's DNS resolver can resolve names under the cluster's base domain, and that the resolved address is reachable from the VM.
> That normally holds without any extra work: the OpenShift ingress domain is backed by a wildcard `*.${BASE_DOMAIN}` record in public DNS pointing at the router's load balancer.
>
> This assumption is what removes the need to manage `/etc/hosts` on the VM.
> The usual VM integration pins the in-cluster name `istiod.istio-system.svc` to the gateway's IP with a generated `hosts` file,
> because that name does not exist in any DNS the VM can query. Here the VM reaches istiod under a name that does resolve,
> so there is nothing to pin and nothing to re-pin when the gateway address changes.
>
> Once the VMs exist, confirm it before configuring the proxy:
>
> ```bash
> ssh -i ./ssh/vm-key "admin@${INTERNAL_VM_ADDR}" "getent hosts istiod.istio-system.svc.${BASE_DOMAIN}"
> ```
>
> If this returns nothing, the VM is on a private resolver that does not forward to public DNS. Point it at a resolver that does,
> or add a single `/etc/hosts` entry mapping the Route hostname to the router's address.

### Step 3: Install Istio control plane

Create the `istio-system` namespace with the network topology label, then deploy the Istio resource configured for VM integration:

```bash
kubectl create namespace istio-system
kubectl label namespace istio-system topology.istio.io/network=cluster-network istio-discovery=enabled
```

```bash
kubectl apply -f - <<EOF
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  version: v1.30-latest
  namespace: istio-system
  updateStrategy:
    type: InPlace
  values:
    meshConfig:
      accessLogFile: /dev/stdout
      defaultConfig:
        proxyMetadata:
          ISTIO_META_DNS_CAPTURE: "true"
      discoverySelectors:
      - matchLabels:
          istio-discovery: enabled
      extensionProviders:
      - name: vm-file-logger
        envoyFileAccessLog:
          path: /var/log/istio/access.log
    global:
      network: cluster-network
    pilot:
      env:
        ISTIOD_CUSTOM_HOST: istiod.istio-system.svc.${BASE_DOMAIN}
EOF
```

> [!NOTE]
> `ISTIOD_CUSTOM_HOST` adds the Route hostname to the SAN list of the certificate istiod serves on port 15012.
> It only adds to the defaults, so the in-cluster names (`istiod.istio-system.svc`) keep working for the injection and validation webhooks.
>
> This is required because the VM proxy uses the host part of its discovery address as both the TLS SNI and the name it
> verifies istiod's certificate against. The SNI has to be the Route hostname for the OpenShift router to pick the right Route,
> so that same hostname must also be present in istiod's certificate.

> [!NOTE]
> `ISTIO_META_DNS_CAPTURE` turns on the sidecar DNS proxy mesh-wide. It is required for the `ServiceEntry` part of this lab:
> `httpbin.external.vm` is not a Kubernetes `Service`, so cluster DNS knows nothing about it and a client pod cannot resolve it.
> With DNS capture enabled, the sidecar answers the query itself from the service registry, using the address istiod
> auto-allocates for address-less `ServiceEntry` resources (from `240.240.0.0/16` by default).
>
> VMs already get this setting through `cluster.env`; the mesh-wide setting is what extends it to in-cluster pods.

> [!IMPORTANT]
> The control plane setting `meshConfig.accessLogFile: /dev/stdout` configures access logging for all proxies in the mesh. This works for containerized workloads where stdout is captured by the container runtime, but on VMs the istio-proxy runs as a systemd service where `/dev/stdout` is a symlink to `/proc/self/fd/1`. When Envoy tries to open this file it logs error for every listener like below:
>
> ```
> 0.0.0.0_8000: unable to open file '/dev/stdout': No such device or address
> virtualOutbound: unable to open file '/dev/stdout': No such device or address
> virtualInbound: unable to open file '/dev/stdout': No such device or address
> ```
>
> The extension provider `vm-file-logger` enables configuring custom access log file per proxy with `Telemetry` API.


Verify the control plane is ready:

```bash
kubectl get istio default
```

### Step 4: Deploy the east-west gateway

Create a Kubernetes Gateway API resource to expose cross-network traffic to the VMs. The gateway only carries data-plane traffic — the control plane is exposed separately in the next step:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eastwestgateway
  namespace: istio-system
  labels:
    topology.istio.io/network: cluster-network
spec:
  gatewayClassName: istio
  listeners:
    - name: cross-network
      hostname: "*.local"
      port: 15443
      protocol: TLS
      tls:
        mode: Passthrough
        options:
          gateway.istio.io/listener-protocol: auto-passthrough
EOF
```

Verify the gateway is ready and has an external address:

```bash
kubectl get gateway eastwestgateway -n istio-system
```

> [!NOTE]
> This gateway carries traffic in one direction only: VM to cluster. Pod IPs are not routable from the VMs, so the VM
> proxies send cross-network traffic to the gateway, which forwards it by SNI.
>
> There is no gateway registered for `vm-network`, and that is deliberate. When a network has no gateway, istiod hands out
> its endpoints directly instead of filtering them out, so in-cluster clients dial the `WorkloadEntry` address itself.
> That is why the address registered for each VM has to be reachable from the mesh cluster — see
> [Step 10](#step-10-generate-vm-configuration-files).

### Step 5: Expose the control plane to the VMs

Expose istiod with an OpenShift passthrough Route instead of routing control plane traffic through the east-west gateway.
The reason for preferring a Route here is DNS: a Route publishes istiod under a real hostname in the cluster's ingress domain,
which the VM can resolve on its own (see [Step 2](#step-2-export-the-cluster-base-domain)). The gateway only offers a load balancer address,
which the VM has to map to the in-cluster name `istiod.istio-system.svc` through a generated `/etc/hosts` entry,
which needs to be updated if the IP changes.

```bash
kubectl apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: istiod-15012
  namespace: istio-system
spec:
  host: istiod.istio-system.svc.${BASE_DOMAIN}
  to:
    kind: Service
    name: istiod
    weight: 100
  port:
    targetPort: 15012
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: None
EOF
```

Verify the route was created with the evaluated hostname:

```bash
kubectl get routes -n istio-system
```

Expected output:

```
NAME           HOST/PORT                                PATH   SERVICES   PORT    TERMINATION   WILDCARD
istiod-15012   istiod.istio-system.svc.${BASE_DOMAIN}          istiod     15012   passthrough   None
```

Because the Route is passthrough, the TLS session terminates at istiod itself. Extract the Istio root CA and confirm from outside the cluster that istiod answers on port 443 with a certificate valid for the Route hostname:

```bash
kubectl get configmap istio-ca-root-cert -n istio-system -o jsonpath='{.data.root-cert\.pem}' > istio-ca.crt
```

```bash
openssl s_client -connect "istiod.istio-system.svc.${BASE_DOMAIN}:443" \
  -servername "istiod.istio-system.svc.${BASE_DOMAIN}" \
  -CAfile istio-ca.crt </dev/null 2>/dev/null | grep -E "Verify return code|subject="
```

Expected output:

```
Verify return code: 0 (ok)
```

A verification failure here means `ISTIOD_CUSTOM_HOST` is not set correctly - check that the hostname in Step 3 matches the Route created above.

## Workloads

### Step 6: Deploy the in-cluster workloads

Create the three namespaces. All of them need the Istio discovery label, because the control plane in
[Step 3](#step-3-install-istio-control-plane) restricts discovery to namespaces carrying it. Only the namespaces
that run pods need sidecar injection:

```bash
kubectl create namespace client
kubectl label namespace client istio-discovery=enabled istio-injection=enabled

kubectl create namespace internal
kubectl label namespace internal istio-discovery=enabled istio-injection=enabled

kubectl create namespace external
kubectl label namespace external istio-discovery=enabled
```

Deploy the `curl` client and the in-cluster half of `httpbin.internal`, both from the upstream Istio samples:

```bash
kubectl apply -n client -f https://raw.githubusercontent.com/istio/istio/master/samples/curl/curl.yaml
kubectl apply -n internal -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
```

The `external` namespace has no pods at all — its only workload is the VM. It still needs a service account, because
that is the identity the VM proxy presents to istiod:

```bash
kubectl create serviceaccount httpbin -n external
```

Verify the pods are running with a sidecar:

```bash
kubectl get pods -n client
kubectl get pods -n internal
```

> [!NOTE]
> The `httpbin` Service created by the sample selects `app: httpbin` and maps port `8000` to target port `8080`.
> Nothing in it is specific to pods — that is exactly why the VM can join it in the next step.

### Step 7: Register the VMs with WorkloadGroups

A `WorkloadGroup` is the template istiod uses to auto-register each VM as a `WorkloadEntry` when its proxy connects.

The VM joining `httpbin.internal` carries `app: httpbin`, the label the sample `Service` selects on, plus `version: vm`
to tell it apart from the in-cluster pods (which carry `version: v1`):

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: httpbin
  namespace: internal
spec:
  metadata:
    labels:
      app: httpbin
      version: vm
  template:
    serviceAccount: httpbin
    network: vm-network
    ports:
      http: 8080
EOF
```

The VM behind the `ServiceEntry` is registered the same way, in the `external` namespace:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: httpbin
  namespace: external
spec:
  metadata:
    labels:
      app: httpbin
      version: vm
  template:
    serviceAccount: httpbin
    network: vm-network
    ports:
      http: 8080
EOF
```

> [!NOTE]
> `template.ports` is what makes inbound traffic work on the VM. `istioctl` turns it into `ISTIO_INBOUND_PORTS=8080`
> in `cluster.env`, and `istio-iptables` uses that to redirect traffic arriving on port `8080` to Envoy on `15006`.
> Without it the value defaults to `*`, which captures every inbound port except the few `istio-start.sh` excludes
> (`22`, `15020`, `15021`, `15090`).

### Step 8: Publish the external VM with a ServiceEntry

The VM in the `external` namespace is not backed by a Kubernetes `Service`, so it needs a `ServiceEntry` to give it a
hostname and a port. `workloadSelector` ties the hostname to the `WorkloadEntry` that istiod auto-registers for the VM:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin
  namespace: external
spec:
  hosts:
  - httpbin.external.vm
  location: MESH_INTERNAL
  resolution: STATIC
  ports:
  - number: 8000
    name: http
    protocol: HTTP
    targetPort: 8080
  workloadSelector:
    labels:
      app: httpbin
EOF
```

> [!NOTE]
> The port block mirrors the sample `Service`: clients call port `8000`, and `targetPort: 8080` is the port httpbin
> actually listens on inside the VM.
>
> `location: MESH_INTERNAL` is what makes this a mesh workload rather than an external dependency — clients use mTLS
> and the VM's identity (`spiffe://cluster.local/ns/external/sa/httpbin`) applies to policy as it would for a pod.

### Step 9: Configure access logging for the proxies running in the VMs

The `vm-file-logger` provider from [Step 3](#step-3-install-istio-control-plane) is applied to the VM workloads only,
selected by the `version: vm` label, so the in-cluster httpbin pods keep logging to stdout:

```bash
for ns in internal external; do
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: vm-access-logging
  namespace: ${ns}
spec:
  selector:
    matchLabels:
      version: vm
  accessLogging:
  - providers:
    - name: vm-file-logger
EOF
done
```

## VM provisioning (optional)

This step is optional. If you already have two VMs, skip to [Step 10](#step-10-generate-vm-configuration-files) — it only
needs their addresses in `INTERNAL_VM_ADDR` and `EXTERNAL_VM_ADDR`. For instructions on provisioning the VMs with KubeVirt,
see [VM_PROVISIONING.md](VM_PROVISIONING.md).

## VM setup

### Step 10: Generate VM configuration files

Use `istioctl` to generate the files each VM needs to join the mesh. The two VMs get separate output directories,
because each one joins a different namespace with a different token:

```bash
mkdir -p internal-vm-config external-vm-config
```

```bash
istioctl x workload entry configure \
  -o internal-vm-config \
  --autoregister \
  --name httpbin \
  --namespace internal \
  --externalIP "${INTERNAL_VM_ADDR}"
```

```bash
istioctl x workload entry configure \
  -o external-vm-config \
  --autoregister \
  --name httpbin \
  --namespace external \
  --externalIP "${EXTERNAL_VM_ADDR}"
```

Each directory gets:
- `cluster.env` — environment variables (namespace, service account, inbound ports, network CIDR)
- `istio-token` — Kubernetes token for certificate retrieval
- `mesh.yaml` — proxy configuration with discovery address
- `root-cert.pem` — root certificate for mTLS
- `hosts` — DNS entries for istiod resolution (not used in this setup, see below)

> [!IMPORTANT]
> `--externalIP` writes `ISTIO_SVC_IP` into `cluster.env`. `istio-start.sh` passes it to the agent as `INSTANCE_IP`,
> which is the address istiod records in the auto-registered `WorkloadEntry` — and therefore the address in-cluster
> clients dial.
>
> Without it the VM reports whatever `hostname --all-ip-addresses` returns first. On a KubeVirt VM with masquerade
> networking that is `10.0.2.2`, an address that exists only inside the guest. Discovery would still look correct
> (`kubectl get workloadentries` shows an entry) but every request from the cluster would fail to connect.
>
> This flag is only needed when the VM's own IP is not routable from the mesh cluster. A VM with a routable address
> can be registered without it.

> [!NOTE]
> `--ingressIP` is intentionally omitted. That flag exists only to generate the `hosts` file, which pins the in-cluster istiod name to the gateway IP. The VM reaches istiod through a Route hostname that resolves over public DNS, so no `/etc/hosts` entry is needed and the generated `hosts` file is left unused. `istioctl` prints a warning about this, which is safe to ignore:
>
> ```
> Could not auto-detect IP for istiod.istio-system.svc/istio-system. Use --ingressIP to manually specify the Gateway address to reach istiod from the VM.
> ```

Patch the generated `mesh.yaml` of both VMs to set the correct Envoy binary path and point the discovery address at the
Route, and append `CA_ADDR` to `cluster.env` so the CA endpoint goes through the Route as well:

```bash
for dir in internal-vm-config external-vm-config; do
  sed -i '/defaultConfig:/a\  binaryPath: /usr/bin/envoy' "${dir}/mesh.yaml"
  sed -i "s|discoveryAddress: .*|discoveryAddress: istiod.istio-system.svc.${BASE_DOMAIN}:443|" "${dir}/mesh.yaml"
  echo "CA_ADDR=istiod.istio-system.svc.${BASE_DOMAIN}:443" >> "${dir}/cluster.env"
done
```

> [!IMPORTANT]
> `discoveryAddress` alone is not enough. The istio-agent uses two separate connections to istiod — xDS and the CA (CSR) — and only the xDS one reads `discoveryAddress` from `mesh.yaml`. `istio-start.sh` derives the CA endpoint on its own and defaults it to the in-cluster name:
>
> ```sh
> DEFAULT_PILOT_ADDRESS="istiod.${ISTIO_SYSTEM_NAMESPACE}.svc:15012"
> CA_ADDR=${CA_ADDR:-${CUSTOM_PILOT_ADDRESS:-${DEFAULT_PILOT_ADDRESS}}}
> ```
>
> That name used to resolve only because of the `/etc/hosts` entry. Without `CA_ADDR` set, xDS connects over the Route but certificate signing fails:
>
> ```
> warn   ca              ca request failed, starting attempt 1 in 93.91323ms
> error  citadelclient   failed to sign CSR: create certificate: rpc error: code = Unavailable desc = connection error
> ```
>
> `istio-start.sh` sources `cluster.env` before applying that default, so a value set there wins.

### Step 11: Configure the VMs

Install the sidecar and run httpbin on both VMs:

- [Manual configuration](VM_MANUAL_CONFIGURATION.md)
- [Ansible configuration](VM_ANSIBLE_CONFIGURATION.md)

## Testing service discovery

### Verify the VMs registered

Both VMs should appear as auto-registered `WorkloadEntry` resources, with the addresses passed as `--externalIP`:

```bash
kubectl get workloadentries -A
```

Expected output:

```
NAMESPACE   NAME                                       AGE   ADDRESS
external    httpbin-192.0.2.11-vm-network              2m    192.0.2.11
internal    httpbin-192.0.2.10-vm-network              2m    192.0.2.10
```

### Service approach: httpbin.internal

The sample `Service` now has two endpoints — the in-cluster pod and the VM — even though nothing about the `Service`
changed:

```bash
istioctl proxy-config endpoints deploy/curl.client --cluster "outbound|8000||httpbin.internal.svc.cluster.local"
```

Expected output:

```
ENDPOINT             STATUS      OUTLIER CHECK     CLUSTER
10.128.2.31:8080     HEALTHY     OK                outbound|8000||httpbin.internal.svc.cluster.local
192.0.2.10:8080      HEALTHY     OK                outbound|8000||httpbin.internal.svc.cluster.local
```

Send a few requests:

```bash
kubectl exec -n client deploy/curl -c curl -- \
  sh -c 'for i in $(seq 10); do curl -s httpbin.internal:8000/headers -o /dev/null -w "%{http_code} "; done; echo'
```

Expected output:

```
200 200 200 200 200 200 200 200 200 200
```

The responses are identical whichever backend served them, so check the client sidecar's access log to see the requests
being spread over the pod and the VM:

```bash
kubectl logs -n client deploy/curl -c istio-proxy --tail=20 \
  | grep 'httpbin.internal.svc' \
  | grep -oE '"[0-9.]+:8080"' | sort | uniq -c
```

Expected output:

```
      6 "10.128.2.31:8080"
      4 "192.0.2.10:8080"
```

To watch the VM leave and rejoin the `Service`, stop its proxy and list the endpoints again — istiod removes the
auto-registered `WorkloadEntry` once the proxy disconnects, leaving only the pod:

```bash
ssh -i ./ssh/vm-key "admin@${INTERNAL_VM_ADDR}" "sudo systemctl stop istio-proxy"
kubectl get workloadentries -n internal
```

```bash
ssh -i ./ssh/vm-key "admin@${INTERNAL_VM_ADDR}" "sudo systemctl start istio-proxy"
```

### ServiceEntry approach: httpbin.external.vm

`httpbin.external.vm` is not a Kubernetes name, so istiod auto-allocates an address for it and the sidecar DNS proxy
serves that address to clients:

```bash
kubectl get serviceentry httpbin -n external -o jsonpath='{.status.addresses}' | jq
```

Expected output:

```json
[{"host":"httpbin.external.vm","value":"240.240.0.1"}]
```

The cluster for that hostname resolves to the external VM:

```bash
istioctl proxy-config endpoints deploy/curl.client --cluster "outbound|8000||httpbin.external.vm"
```

Expected output:

```
ENDPOINT            STATUS      OUTLIER CHECK     CLUSTER
192.0.2.11:8080     HEALTHY     OK                outbound|8000||httpbin.external.vm
```

Call it from the client pod:

```bash
kubectl exec -n client deploy/curl -c curl -- curl -sS httpbin.external.vm:8000/headers
```

The `X-Forwarded-Client-Cert` header in the response confirms the request reached the VM over mTLS with the client's
identity:

```
"X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/external/sa/httpbin;...;URI=spiffe://cluster.local/ns/client/sa/curl"
```

### Verify the VM-to-cluster direction

Traffic the other way goes through the east-west gateway. From either VM:

```bash
ssh -i ./ssh/vm-key "admin@${EXTERNAL_VM_ADDR}" "curl -s httpbin.internal.svc:8000/headers"
```

### Verify VM access logs

Confirm that the VM proxies are writing access logs to the file configured by the `Telemetry` resources:

```bash
ssh -i ./ssh/vm-key "admin@${INTERNAL_VM_ADDR}" "cat /var/log/istio/access.log"
```

Expected output — an inbound entry for a request the client pod sent to the VM, and an outbound entry for a request the
VM sent to the mesh:

```
[2026-10-20T16:40:45.630Z] "GET /headers HTTP/1.1" 200 - via_upstream - "-" 0 558 2 1 "-" "curl/8.16.0" "0b1f2e6c-..." "httpbin.internal:8000" "127.0.0.1:8080" inbound|8080|| 127.0.0.1:35412 10.0.2.2:8080 10.128.2.44:0 outbound_.8000_._.httpbin.internal.svc.cluster.local default
[2026-10-20T16:41:02.118Z] "GET /headers HTTP/1.1" 200 - via_upstream - "-" 0 558 30 27 "-" "curl/7.76.1" "d5ec4032-..." "httpbin.internal.svc:8000" "10.0.190.182:15443" outbound|8000||httpbin.internal.svc.cluster.local 10.0.2.2:43208 172.30.105.168:8000 10.0.2.2:41218 - default
```
