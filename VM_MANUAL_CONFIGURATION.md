# Configuring VM manually

> **Prerequisite:** The VM must have the `baseos` repository enabled - it provides core dependencies required by the `istio-proxy` RPM.
>
> If the `baseos` repository is not enabled, add it:
>
> ```bash
> ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo yum-config-manager --add-repo https://mirror.stream.centos.org/9-stream/BaseOS/x86_64/os/"
> ```

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
```

