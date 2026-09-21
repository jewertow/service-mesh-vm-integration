# Integrating an External VM with Istio on OpenShift

Integrate an external virtual machine into the Istio service mesh. The VM runs an Istio sidecar proxy and connects to the control plane through an east-west gateway. This lab uses KubeVirt to provision the VM in a dedicated OpenShift cluster, but the mesh is agnostic to how the VM is provisioned.

## Prerequisites

- OpenShift cluster with OpenShift Service Mesh 3 operator installed
- `kubectl` and `istioctl` CLI tools installed
- An external VM or another OpenShift cluster with OpenShift Virtualization installed

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

### Step 2: Install Istio control plane

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
      discoverySelectors:
      - matchLabels:
          istio-discovery: enabled
      extensionProviders:
      - name: vm-file-logger
        envoyFileAccessLog:
          path: /var/log/istio/access.log
    global:
      network: cluster-network
EOF
```

> [!NOTE]
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

### Step 3: Deploy the east-west gateway

Create a Kubernetes Gateway API resource to expose cross-network traffic and the control plane to the VM:

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
    - name: tls-istiod
      port: 15012
      protocol: TLS
      tls:
        mode: Passthrough
    - name: tls-istiodwebhook
      port: 15017
      protocol: TLS
      tls:
        mode: Passthrough
EOF
```

Verify the gateway is ready and has an external address:

```bash
kubectl get gateway eastwestgateway -n istio-system
```

### Step 4: Expose the control plane to the VM

TLSRoute is not supported in OCP 4.22, so we use a mix of APIs: the Kubernetes Gateway API deploys the actual gateway workload (Step 3), while the Istio Gateway and VirtualService APIs configure routing to the remote istiod.

Create an Istio Gateway and VirtualService to route TLS traffic from the east-west gateway to istiod on ports 15012 (XDS) and 15017 (webhook):

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
  - port:
      name: tls-istiodwebhook
      number: 15017
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
  - match:
    - port: 15017
      sniHosts:
      - istiod.istio-system.svc
    route:
    - destination:
        host: istiod.istio-system.svc.cluster.local
        port:
          number: 443
EOF
```

### Step 5: Deploy httpbin

Create the `httpbin` namespace with the Istio discovery label and sidecar injection, then deploy httpbin from the upstream Istio samples:

```bash
kubectl create namespace httpbin
kubectl label namespace httpbin istio-discovery=enabled istio-injection=enabled
kubectl apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
```

Verify httpbin is running:

```bash
kubectl get pods -n httpbin
```

### Step 6: Configure the VM namespace and WorkloadGroup

Create a namespace for the VM workload, label it for Istio discovery, and create a service account:

```bash
kubectl create namespace curl-external
kubectl label namespace curl-external istio-discovery=enabled
kubectl create serviceaccount curl -n curl-external
```

Create a `WorkloadGroup` to define the VM identity in the mesh:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: curl
  namespace: curl-external
spec:
  metadata:
    labels:
      app: curl
  template:
    serviceAccount: curl
    network: vm-network
EOF
```

### Step 7: Configure access logging for the proxy running in VM

```bash
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: vm-access-logging
  namespace: curl-external
spec:
  accessLogging:
  - providers:
    - name: vm-file-logger
EOF
```

### Step 8: Generate VM configuration files

Use `istioctl` to generate the files the VM needs to join the mesh:

```bash
export INGRESS_IP=$(kubectl get svc eastwestgateway-istio -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If the load balancer provides a hostname instead of an IP (e.g. on AWS), use the hostname:

```bash
export INGRESS_IP=$(kubectl get svc eastwestgateway-istio -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

```bash
mkdir -p curl-vm-config
istioctl x workload entry configure \
  -o curl-vm-config \
  --autoregister \
  --ingressIP "${INGRESS_IP}" \
  --name curl \
  --namespace curl-external
```

This generates:
- `cluster.env` — environment variables (namespace, service account, network CIDR)
- `istio-token` — Kubernetes token for certificate retrieval
- `mesh.yaml` — proxy configuration with discovery address
- `root-cert.pem` — root certificate for mTLS
- `hosts` — DNS entries for istiod resolution

Patch the generated `mesh.yaml` to set the correct Envoy binary path:

```bash
sed -i '/defaultConfig:/a\  binaryPath: /usr/bin/envoy' curl-vm-config/mesh.yaml
```

## VM provisioning (optional)

This step is optional. If you already have a VM, skip to [Configuring VM](#configuring-vm). For instructions on provisioning a VM with KubeVirt, see [VM_PROVISIONING.md](VM_PROVISIONING.md).

## Configuring VM

- [Manual configuration](VM_MANUAL_CONFIGURATION.md)
- [Ansible configuration](VM_ANSIBLE_CONFIGURATION.md)

## Testing connectivity

### Install curl on the VM

The test below requires `curl`, which is not installed on a minimal VM image:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo yum install -y --nogpgcheck curl"
```

> **Prerequisite:** `curl` is provided by the `appstream` repository.
>
> If the `appstream` repository is not enabled, add it before installing:
>
> ```bash
> ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo yum-config-manager --add-repo https://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/"
> ```

### Verify connectivity to httpbin

From the VM, test connectivity to httpbin running in the mesh:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "curl -v httpbin.httpbin.svc:8000/headers"
```

### Verify VM access logs

Confirm that the VM proxy is writing access logs to the file configured in the mesh setup:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "cat /var/log/istio/access.log"
```

Expected output:

```
[2026-09-09T16:40:45.630Z] "GET /headers HTTP/1.1" 200 - via_upstream - "-" 0 558 30 27 "-" "curl/7.76.1" "d5ec4032-8011-4487-901a-571f5ac07c0e" "httpbin.httpbin.svc:8000" "10.0.190.182:15443" outbound|8000||httpbin.httpbin.svc.cluster.local 10.0.2.2:43208 172.30.105.168:8000 10.0.2.2:41218 - default
```
