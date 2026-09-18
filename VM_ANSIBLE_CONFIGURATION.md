# Configuring VM with Ansible

## Step 1: Generate inventory

Generate the inventory from the template using the `VM_SSH_ADDR` environment variable set during [VM provisioning](VM_PROVISIONING.md):

```bash
envsubst < ansible/inventory.yaml.template > ansible/inventory.yaml
```

## Step 2: Run the playbook

```bash
export ISTIO_PROXY_RPM_URL=<your-repo-url>
ansible-playbook -i ansible/inventory.yaml ansible/playbook.yaml
```
