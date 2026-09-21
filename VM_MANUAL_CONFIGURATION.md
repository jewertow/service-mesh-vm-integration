# Configuring VM manually

> **Prerequisite:** The VM must have the `baseos` and `appstream` repositories enabled. These provide core dependencies required by the `istio-proxy` RPM.

## Step 1: Install the Istio sidecar

```bash
export ISTIO_PROXY_RPM_URL=<your-repo-url>
```
```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo yum-config-manager --add-repo ${ISTIO_PROXY_RPM_URL} && sudo yum install -y --nogpgcheck --setopt=sslverify=0 istio-proxy.x86_64"
```

## Step 2: Transfer configuration files

Transfer the generated configuration files to the VM:

```bash
scp -i ./ssh/vm-key curl-vm-config/* "admin@${VM_SSH_ADDR}":~
```

## Step 3: Install the root certificate

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo cp root-cert.pem /etc/certs/root-cert.pem"
```

## Step 4: Install the token

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo mkdir -p /var/run/secrets/tokens && sudo cp istio-token /var/run/secrets/tokens/istio-token"
```

## Step 5: Deploy configuration files

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo cp cluster.env /var/lib/istio/envoy/cluster.env && sudo cp mesh.yaml /etc/istio/config/mesh && sudo sh -c 'cat hosts >> /etc/hosts'"
```

## Step 6: Set ownership

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets"
```

## Step 7: Start the Istio agent

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo systemctl enable --now istio-proxy"
```

Verify the agent started successfully:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo systemctl status istio-proxy"
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "cat /var/log/istio/istio.log"
```

> [!Note]
> A successful startup should also be visible in the east-west gateway logs. You should see entries like:
> ```
> [2026-09-09T16:23:41.927Z] "- - -" 0 - - - "-" 52316 135070 687977 - "-" "-" "-" "-" "10.131.0.36:15012" outbound|15012||istiod.istio-system.svc.cluster.local 10.128.2.22:53452 10.128.2.22:15012 100.64.0.2:43315 istiod.istio-system.svc -
> ```
