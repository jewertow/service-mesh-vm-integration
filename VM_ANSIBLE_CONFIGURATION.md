# Configuring VM with Ansible

> **Prerequisite:** The VM must be registered with Red Hat Subscription Management - it provides the repository with the `istio-proxy` RPM and the repositories with its dependencies.
>
> If the VM is already registered, skip this - the playbook leaves the registration untouched.
>
> Otherwise, export your credentials:
>
> ```bash
> export RHSM_USERNAME=<your-username>
> export RHSM_PASSWORD=<your-password>
> ```
>
> The playbook picks them up and registers the VM when [you run it](#step-2-run-the-playbook). To register it upfront instead:
>
> ```bash
> ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo subscription-manager register --username ${RHSM_USERNAME} --password ${RHSM_PASSWORD}"
> ```

## Step 1: Generate inventory

Generate the inventory from the template using the `VM_SSH_ADDR` environment variable set during [VM provisioning](VM_PROVISIONING.md):

```bash
envsubst < ansible/inventory.yaml.template > ansible/inventory.yaml
```

## Step 2: Run the playbook

```bash
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml
```
