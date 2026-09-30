# Configuring VM with Ansible

The playbook installs Docker on the VM and runs ztunnel in dedicated mode as a container. It is the automated equivalent of [VM_MANUAL_CONFIGURATION.md](VM_MANUAL_CONFIGURATION.md) — read that file for what each ztunnel setting does.

## Prerequisites

- `ansible-core` 2.15 or later (the playbook uses the `dnf` module's `allowerasing` option)
- The `community.docker` collection:

  ```bash
  ansible-galaxy collection install community.docker
  ```

## Step 1: Generate inventory

Generate the inventory from the template using the `VM_SSH_ADDR` environment variable set during [VM provisioning](VM_PROVISIONING.md):

```bash
envsubst < ansible/inventory.yaml.template > ansible/inventory.yaml
```

## Step 2: Run the playbook

The playbook reads `ZTUNNEL_IMAGE`, `EW_GATEWAY_ADDR` and `VM_IP` from the environment — all three are exported during [mesh setup](README.md#mesh-setup). It fails immediately if any is missing.

```bash
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml
```

The final task polls ztunnel's readiness endpoint, so a successful run means ztunnel reached istiod and completed its initial XDS sync. Continue with [Verifying the integration](README.md#verifying-the-integration).

## Refreshing the token

The service account token is short-lived. Regenerate it and re-run the playbook; ztunnel re-reads the file on every request, so the container is not restarted needlessly:

```bash
kubectl create token vm -n external-vm --audience=istio-ca --duration=24h > ztunnel-vm-config/istio-token
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml
```

## Overriding the workload identity

The workload triple is exposed as playbook variables, so it can be changed without editing the playbook:

```bash
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml \
  -e ztunnel_namespace=external-vm \
  -e ztunnel_workload_name=vm-1 \
  -e ztunnel_service_account=vm \
  -e ztunnel_network=vm-network \
  -e ztunnel_cluster_id=Kubernetes
```

These must stay consistent with the `WorkloadEntry` and with the service account the token was issued for.
