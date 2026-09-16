# VM Provisioning with KubeVirt

This guide applies to any external VM workload regardless of how it is provisioned. In this particular test environment, we used KubeVirt running in a separate OpenShift cluster to provision the VM. A sample KubeVirt VM definition with pre-configured yum repos for installing the Istio sidecar proxy is available in [vm.yaml](vm.yaml).

> [!Note]
> KubeVirt VMs support direct console access via `virtctl ssh`/`virtctl console`, which does not require SSH key setup. However, this lab configures standard SSH access to simulate a real external VM that is not running on OpenShift.

## Step 1: Generate an SSH key pair

Generate an SSH key pair for connecting to the VM:

```bash
mkdir -p ssh
ssh-keygen -t ed25519 -C "test@example.com" -f ./ssh/vm-key -N ""
```

## Step 2: Create the VM

Create the VM, injecting the public key into cloud-init:

```bash
kubectl create ns curl
sed "s|__SSH_PUBLIC_KEY__|$(cat ssh/vm-key.pub)|" vm.yaml | kubectl apply -n curl -f -
```

## Step 3: Expose the VM with a LoadBalancer service

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

Wait for the load balancer address to be assigned:

```bash
export VM_SSH_ADDR=$(kubectl get svc rhel9-ssh-lb -n curl -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

If the load balancer provides a hostname instead of an IP (e.g. on AWS), use the hostname:

```bash
export VM_SSH_ADDR=$(kubectl get svc rhel9-ssh-lb -n curl -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

## Step 4: Verify SSH connectivity

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" hostname
```
