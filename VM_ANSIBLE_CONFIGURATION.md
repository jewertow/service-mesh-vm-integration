# Configuring VMs with Ansible

The playbook configures both VMs in one run. For each host it installs and starts the Istio sidecar from the per-host configuration directory generated earlier, and runs httpbin in a podman container.

> **Prerequisite:** The VMs must be registered with Red Hat Subscription Management - it provides the repository with the `istio-proxy` RPM and the repositories with its dependencies.
>
> If the VMs are already registered, skip this - the playbook leaves the registration untouched.
>
> Otherwise, export your credentials:
>
> ```bash
> export RHSM_USERNAME=<your-username>
> export RHSM_PASSWORD=<your-password>
> ```
>
> The playbook picks them up and registers each VM when [you run it](#step-2-run-the-playbook). To register them upfront instead:
>
> ```bash
> for VM_ADDR in "${INTERNAL_VM_ADDR}" "${EXTERNAL_VM_ADDR}"; do
>   ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo subscription-manager register --username ${RHSM_USERNAME} --password ${RHSM_PASSWORD}"
> done
> ```

## Step 1: Generate inventory

Generate the inventory from the template using the `INTERNAL_VM_ADDR` and `EXTERNAL_VM_ADDR` environment variables set during [VM provisioning](VM_PROVISIONING.md):

```bash
envsubst < ansible/inventory.yaml.template > ansible/inventory.yaml
```

The template maps each host to its own generated configuration directory:

| Host | Configuration directory |
|---|---|
| `httpbin-internal-vm` | `internal-vm-config` |
| `httpbin-external-vm` | `external-vm-config` |

## Step 2: Run the playbook

```bash
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml
```

To configure a single VM, limit the run to one host:

```bash
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml --limit httpbin-internal-vm
```

## Step 3: Verify

```bash
for VM_ADDR in "${INTERNAL_VM_ADDR}" "${EXTERNAL_VM_ADDR}"; do
  ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "systemctl is-active istio-proxy httpbin && curl -s localhost:8080/status/200 -o /dev/null -w '%{http_code}\n'"
done
```

Expected output per VM:

```
active
active
200
```
