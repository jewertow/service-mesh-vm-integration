# Integrating an External VM with Istio on OpenShift

Integrate an external virtual machine into the Istio service mesh. The VM runs an Istio sidecar proxy and connects to the control plane through an east-west gateway. This lab uses KubeVirt to provision the VM in a dedicated OpenShift cluster, but the mesh is agnostic to how the VM is provisioned.

## Prerequisites

- OpenShift cluster with OpenShift Service Mesh 3 operator installed
- `kubectl` and `istioctl` CLI tools installed
- An external VM or another OpenShift cluster with OpenShift Virtualization installed
- `virtctl` CLI installed (if using OpenShift Virtualization)

## Step 1: Install IstioCNI

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

## Step 2: Install Istio control plane

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
      extensionProviders:
      - name: vm-file-logger
        envoyFileAccessLog:
          path: /var/log/istio/access.log
      discoverySelectors:
      - matchLabels:
          istio-discovery: enabled
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: cluster-network
EOF
```

Verify the control plane is ready:

```bash
kubectl get istio default
```

## Step 3: Deploy the east-west gateway

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

## Step 4: Expose the control plane to the VM

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
EOF
```
```bash
kubectl apply -f - <<EOF
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

## TODO: either remove Istio Gateway config or remove 15012 and 15017 ports from K8S Gateway config

## Step 5: Deploy httpbin

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

## Step 6: Configure the VM namespace and WorkloadGroup

Create a namespace for the VM workload, label it for Istio discovery, and create a service account:

```bash
kubectl create namespace curl-external
kubectl label namespace curl-external istio-discovery=enabled
kubectl create serviceaccount curl -n curl-external
```

Create a `WorkloadGroup` to define the VM identity in the mesh:

```bash
cat <<EOF > workloadgroup.yaml
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

```bash
kubectl apply -f workloadgroup.yaml
```

## Step 7: Configure access logging for the VM

The control plane setting `meshConfig.accessLogFile: /dev/stdout` configures Envoy access logging for all proxies in the mesh. This works for containerized workloads where stdout is captured by the container runtime, but on VMs the istio-proxy runs as a systemd service where `/dev/stdout` is not available. Without this step, the VM proxy logs errors on every listener:

```
0.0.0.0_8000: unable to open file '/dev/stdout': No such device or address
virtualOutbound: unable to open file '/dev/stdout': No such device or address
virtualInbound: unable to open file '/dev/stdout': No such device or address
```

A `Telemetry` resource scoped to the VM namespace overrides the legacy `meshConfig.accessLogFile` setting for proxies in that namespace only. In-cluster sidecars and gateways continue using `/dev/stdout` as before.

Create a `Telemetry` resource that directs the VM proxy to write access logs to a local file using the `vm-file-logger` extension provider defined in the Istio control plane configuration:

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

> **Note:** The `vm-file-logger` extension provider is defined in the Istio resource (Step 2) under `meshConfig.extensionProviders`. It writes to `/var/log/istio/access.log`, a directory that the `istio-proxy` RPM already creates on the VM. When a `Telemetry` resource with `accessLogging` applies to a proxy, it completely replaces the legacy `meshConfig.accessLogFile` setting for that proxy — the two do not coexist.

## Step 8: Generate VM configuration files

Use `istioctl` to generate the files the VM needs to join the mesh:

```bash
WORK_DIR=vm-config
mkdir -p "${WORK_DIR}"

INGRESS_IP=$(kubectl get svc eastwestgateway-istio -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

istioctl x workload entry configure \
  -o "${WORK_DIR}" \
  --clusterID cluster1 \
  --autoregister \
  --ingressIP "${INGRESS_IP}" \
  -f workloadgroup.yaml
```

This generates:
- `cluster.env` — environment variables (namespace, service account, network CIDR)
- `istio-token` — Kubernetes token for certificate retrieval
- `mesh.yaml` — proxy configuration with discovery address
- `root-cert.pem` — root certificate for mTLS
- `hosts` — DNS entries for istiod resolution

Patch the generated `mesh.yaml` to set the correct Envoy binary path:

```bash
sed -i '/defaultConfig:/a\  binaryPath: /usr/bin/envoy' "${WORK_DIR}"/mesh.yaml
```

## (Optional) VM provisioning

This guide applies to any external VM workload regardless of how it is provisioned. In this particular test environment, we used KubeVirt running in a separate OpenShift cluster to provision the VM. A sample KubeVirt VM definition with pre-configured yum repos for installing the Istio sidecar proxy is available in [vm.yaml](vm.yaml).

> **Note:** KubeVirt VMs support direct console access via `virtctl ssh`/`virtctl console`, which does not require SSH key setup. However, this lab configures standard SSH access to simulate a real external VM that is not running on OpenShift.

Generate an SSH key pair for connecting to the VM:

```bash
mkdir -p ssh
ssh-keygen -t ed25519 -C "test@example.com" -f ./ssh/vm-key -N ""
```

Create the VM, injecting the public key into cloud-init:

```bash
kubectl create ns curl
sed "s|__SSH_PUBLIC_KEY__|$(cat ssh/vm-key.pub)|" vm.yaml | kubectl apply -n curl -f -
```

Expose the VM with a LoadBalancer service for SSH access:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: rhel9-ssh-lb
  namespace: curl
spec:
  type: LoadBalancer
  ports:
  - name: ssh
    port: 22
    targetPort: 22
    protocol: TCP
  selector:
    vm: curl
EOF
```

Wait for the load balancer IP to be assigned:

```bash
export VM_SSH_ADDR=$(kubectl get svc rhel9-ssh-lb -n curl -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

Verify SSH connectivity:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" hostname
```

## Step 9: Configure the VM

> **Prerequisite:** The VM must have the `baseos` and `appstream` repositories enabled. These provide core dependencies required by the `istio-proxy` RPM.

Install the Istio sidecar:

```bash
export ISTIO_PROXY_RPM_URL=<your-repo-url>
```
```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo yum-config-manager --add-repo ${ISTIO_PROXY_RPM_URL} && sudo yum install -y --nogpgcheck --setopt=sslverify=0 istio-proxy.x86_64"
```

Transfer the generated configuration files to the VM:

```bash
scp -i ./ssh/vm-key "${WORK_DIR}"/cluster.env "${WORK_DIR}"/istio-token "${WORK_DIR}"/mesh.yaml "${WORK_DIR}"/root-cert.pem "${WORK_DIR}"/hosts "admin@${VM_SSH_ADDR}":~
```

Install the root certificate:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo cp root-cert.pem /etc/certs/root-cert.pem"
```

Install the token:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo mkdir -p /var/run/secrets/tokens && sudo cp istio-token /var/run/secrets/tokens/istio-token"
```

Deploy the configuration files:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo cp cluster.env /var/lib/istio/envoy/cluster.env && sudo cp mesh.yaml /etc/istio/config/mesh && sudo sh -c 'cat hosts >> /etc/hosts'"
```

Set ownership:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo mkdir -p /etc/istio/proxy && sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets"
```

Start the Istio agent:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo systemctl start istio-proxy"
```

Verify the agent started successfully:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo systemctl status istio-proxy"
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "cat /var/log/istio/istio.log"
```

> **Note:** A successful startup should also be visible in the east-west gateway logs. You should see entries like:
> ```
> [2026-09-09T16:23:41.927Z] "- - -" 0 - - - "-" 52316 135070 687977 - "-" "-" "-" "-" "10.131.0.36:15012" outbound|15012||istiod.istio-system.svc.cluster.local 10.128.2.22:53452 10.128.2.22:15012 100.64.0.2:43315 istiod.istio-system.svc -
> ```

## Step 10: Verify connectivity

From the VM, test connectivity to httpbin running in the mesh:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "curl -v httpbin.httpbin.svc:8000/headers"
```

## Step 11: Verify VM access logs

Confirm that the VM proxy is writing access logs to the file configured in Step 7:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "cat /var/log/istio/access.log"
```

Expected output:

```
[2026-09-09T16:40:45.630Z] "GET /headers HTTP/1.1" 200 - via_upstream - "-" 0 558 30 27 "-" "curl/7.76.1" "d5ec4032-8011-4487-901a-571f5ac07c0e" "httpbin.httpbin.svc:8000" "10.0.190.182:15443" outbound|8000||httpbin.httpbin.svc.cluster.local 10.0.2.2:43208 172.30.105.168:8000 10.0.2.2:41218 - default
```

