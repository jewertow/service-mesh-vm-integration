# Configuring VM manually

> **Prerequisite:** The VM must be registered with Red Hat Subscription Management - it provides the repository with the `istio-proxy` RPM and the repositories with its dependencies.
>
> If the VM is not registered yet, export your credentials:
>
> ```bash
> export RHSM_USERNAME=<your-username>
> export RHSM_PASSWORD=<your-password>
> ```
>
> and register it:
>
> ```bash
> ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo subscription-manager register --username ${RHSM_USERNAME} --password ${RHSM_PASSWORD}"
> ```

## Step 1: Install the Istio sidecar

Enable the OpenShift Service Mesh repository:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo subscription-manager repos --enable ossm-3-for-rhel-9-x86_64-rpms"
```

Install the `istio-proxy` package:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo dnf install -y istio-proxy"
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
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo cp cluster.env /var/lib/istio/envoy/cluster.env && sudo cp mesh.yaml /etc/istio/config/mesh"
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

