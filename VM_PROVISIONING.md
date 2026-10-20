# VM Provisioning with KubeVirt

This guide applies to any external VM workload regardless of how it is provisioned. In this particular test environment, we used KubeVirt running in a separate OpenShift cluster to provision the VMs. A sample KubeVirt VM definition is available in [vm.yaml](vm.yaml).

The lab needs two VMs:

| VM | Joins the mesh as | Discovered through |
|---|---|---|
| `httpbin-internal-vm` | `httpbin` in namespace `internal` | a Kubernetes `Service` that also selects in-cluster pods |
| `httpbin-external-vm` | `httpbin` in namespace `external` | a `ServiceEntry` with the custom hostname `httpbin.external.vm` |

> [!Note]
> KubeVirt VMs support direct console access via `virtctl ssh`/`virtctl console`, which does not require SSH key setup. However, this lab configures standard SSH access to simulate a real external VM that is not running on OpenShift.

## Step 1: Generate an SSH key pair

Generate an SSH key pair for connecting to the VMs:

```bash
mkdir -p ssh
ssh-keygen -t ed25519 -C "test@example.com" -f ./ssh/vm-key -N ""
```

## Step 2: Create the VMs

Create both VMs, injecting the public key into cloud-init:

```bash
kubectl create ns vms
for VM in httpbin-internal-vm httpbin-external-vm; do
  sed -e "s|__VM_NAME__|${VM}|g" \
      -e "s|__SSH_PUBLIC_KEY__|$(cat ssh/vm-key.pub)|" vm.yaml \
    | kubectl apply -n vms -f -
done
```

## Step 3: Expose the VMs with LoadBalancer services

Each VM needs an address that is reachable **from outside its own cluster**, on two ports:

- `22` — SSH, used to configure the VM
- `8080` — the port the mesh sends inbound traffic to; the Istio iptables rules inside the guest redirect it to Envoy on `15006`

Port `8080` is what makes service discovery work in the mesh-to-VM direction. The address assigned here is the address that ends up in the auto-registered `WorkloadEntry`, so in-cluster clients dial it directly.

```bash
for VM in httpbin-internal-vm httpbin-external-vm; do
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${VM}-lb
  namespace: vms
spec:
  type: LoadBalancer
  ports:
  - name: ssh
    port: 22
    targetPort: 22
    protocol: TCP
  - name: mesh-inbound
    port: 8080
    targetPort: 8080
    protocol: TCP
  selector:
    vm: ${VM}
EOF
done
```

Wait for the load balancer addresses to be assigned and export them:

```bash
export INTERNAL_VM_ADDR=$(kubectl get svc httpbin-internal-vm-lb -n vms -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export EXTERNAL_VM_ADDR=$(kubectl get svc httpbin-external-vm-lb -n vms -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "internal VM: ${INTERNAL_VM_ADDR}, external VM: ${EXTERNAL_VM_ADDR}"
```

> [!IMPORTANT]
> These must be IP addresses, not hostnames - they are written into the `WorkloadEntry` `address` field, which only accepts IPs for statically resolved endpoints.
> If your load balancer publishes a hostname instead (e.g. on AWS), resolve it first:
>
> ```bash
> export INTERNAL_VM_ADDR=$(getent hosts "$(kubectl get svc httpbin-internal-vm-lb -n vms -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')" | awk '{print $1}' | head -1)
> ```

## Step 4: Verify SSH connectivity

```bash
ssh -i ./ssh/vm-key "admin@${INTERNAL_VM_ADDR}" hostname
ssh -i ./ssh/vm-key "admin@${EXTERNAL_VM_ADDR}" hostname
```

Expected output:

```
httpbin-internal-vm
httpbin-external-vm
```
