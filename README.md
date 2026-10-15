# Integrating an External VM with Istio on OpenShift

Integrate an external virtual machine into the Istio service mesh. The VM runs an Istio sidecar proxy, reaches the control plane through OpenShift passthrough Routes, and reaches in-mesh workloads through an east-west gateway. This lab uses KubeVirt to provision the VM in a dedicated OpenShift cluster, but the mesh is agnostic to how the VM is provisioned.

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

### Step 2: Export the cluster base domain

The control plane is exposed to the VM through OpenShift Routes, so the cluster's default ingress domain is needed in several of the following steps:

```bash
export BASE_DOMAIN=$(kubectl get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "Base domain: ${BASE_DOMAIN}"
```

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
        ISTIOD_CUSTOM_HOST: istiod.istio-system.svc.${BASE_DOMAIN},istiod-webhook.istio-system.svc.${BASE_DOMAIN}
EOF
```

> [!IMPORTANT]
> `ISTIOD_CUSTOM_HOST` adds the Route hostnames to the SAN list of the certificate istiod serves on ports 15012 and 15017. It only adds to the defaults, so the in-cluster names (`istiod.istio-system.svc`) keep working for the injection and validation webhooks.
>
> This is required because the VM proxy uses the host part of its discovery address as both the TLS SNI and the name it verifies istiod's certificate against. The SNI has to be the Route hostname for the OpenShift router to pick the right Route, so that same hostname must also be present in istiod's certificate.

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

### Step 4: Deploy the east-west gateway

Create a Kubernetes Gateway API resource to expose cross-network traffic to the VM. The gateway only carries data-plane traffic — the control plane is exposed separately in the next step:

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

### Step 5: Expose the control plane to the VM

Expose istiod with OpenShift passthrough Routes instead of routing control plane traffic through the east-west gateway. The OpenShift router listens on port 443 and selects the backend by SNI, so each istiod port needs its own hostname:

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
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: istiod-15017
  namespace: istio-system
spec:
  host: istiod-webhook.istio-system.svc.${BASE_DOMAIN}
  to:
    kind: Service
    name: istiod
    weight: 100
  port:
    targetPort: 15017
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: None
EOF
```

Verify the routes were created with the evaluated hostnames:

```bash
kubectl get routes -n istio-system
```

Expected output:

```
NAME           HOST/PORT                                            PATH   SERVICES   PORT    TERMINATION   WILDCARD
istiod-15012   istiod.istio-system.svc.${BASE_DOMAIN}                      istiod     15012   passthrough   None
istiod-15017   istiod-webhook.istio-system.svc.${BASE_DOMAIN}              istiod     15017   passthrough   None
```

Because the Routes are passthrough, the TLS session terminates at istiod itself. Extract the Istio root CA and confirm from outside the cluster that istiod answers on port 443 with a certificate valid for the Route hostname:

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

A verification failure here means `ISTIOD_CUSTOM_HOST` is not set correctly — check that the Route hostnames in Step 3 match the ones created above.

### Step 6: Deploy httpbin

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

### Step 7: Configure the VM namespace and WorkloadGroup

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

### Step 8: Configure access logging for the proxy running in VM

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

### Step 9: Generate VM configuration files

Use `istioctl` to generate the files the VM needs to join the mesh:

```bash
mkdir -p curl-vm-config
istioctl x workload entry configure \
  -o curl-vm-config \
  --autoregister \
  --name curl \
  --namespace curl-external
```

This generates:
- `cluster.env` — environment variables (namespace, service account, network CIDR)
- `istio-token` — Kubernetes token for certificate retrieval
- `mesh.yaml` — proxy configuration with discovery address
- `root-cert.pem` — root certificate for mTLS
- `hosts` — DNS entries for istiod resolution (not used in this setup, see below)

> [!NOTE]
> `--ingressIP` is intentionally omitted. That flag exists only to generate the `hosts` file, which pins the in-cluster istiod name to the gateway IP. The VM reaches istiod through a Route hostname that resolves over public DNS, so no `/etc/hosts` entry is needed and the generated `hosts` file is left unused. `istioctl` prints a warning about this, which is safe to ignore:
>
> ```
> Could not auto-detect IP for istiod.istio-system.svc/istio-system. Use --ingressIP to manually specify the Gateway address to reach istiod from the VM.
> ```

Patch the generated `mesh.yaml` to set the correct Envoy binary path and point the discovery address at the Route:

```bash
sed -i '/defaultConfig:/a\  binaryPath: /usr/bin/envoy' curl-vm-config/mesh.yaml
sed -i "s|discoveryAddress: .*|discoveryAddress: istiod.istio-system.svc.${BASE_DOMAIN}:443|" curl-vm-config/mesh.yaml
```

Point the CA endpoint at the Route as well, by appending `CA_ADDR` to `cluster.env`:

```bash
echo "CA_ADDR=istiod.istio-system.svc.${BASE_DOMAIN}:443" >> curl-vm-config/cluster.env
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
