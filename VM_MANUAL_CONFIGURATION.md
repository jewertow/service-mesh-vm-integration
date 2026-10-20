# Configuring VMs manually

Both VMs are configured the same way - only the address and the generated configuration directory differ. Run the steps below once per VM, selecting the VM first:

```bash
# internal VM
export VM_ADDR="${INTERNAL_VM_ADDR}"
export VM_CONFIG=internal-vm-config
```

```bash
# external VM
export VM_ADDR="${EXTERNAL_VM_ADDR}"
export VM_CONFIG=external-vm-config
```

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
> ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo subscription-manager register --username ${RHSM_USERNAME} --password ${RHSM_PASSWORD}"
> ```

## Step 1: Install the Istio sidecar

Enable the OpenShift Service Mesh repository:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo subscription-manager repos --enable ossm-3-for-rhel-9-x86_64-rpms"
```

Install the `istio-proxy` package:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo dnf install -y istio-proxy"
```

## Step 2: Transfer configuration files

Transfer the generated configuration files to the VM:

```bash
scp -i ./ssh/vm-key "${VM_CONFIG}"/* "admin@${VM_ADDR}":~
```

## Step 3: Install the root certificate

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo cp root-cert.pem /etc/certs/root-cert.pem"
```

## Step 4: Install the token

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo mkdir -p /var/run/secrets/tokens && sudo cp istio-token /var/run/secrets/tokens/istio-token"
```

## Step 5: Deploy configuration files

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo cp cluster.env /var/lib/istio/envoy/cluster.env && sudo cp mesh.yaml /etc/istio/config/mesh"
```

## Step 6: Set ownership

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets"
```

## Step 7: Start the Istio agent

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo systemctl enable --now istio-proxy"
```

Verify the agent started successfully:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo systemctl status istio-proxy"
```

## Step 8: Run httpbin

Install podman:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo dnf install -y podman"
```

Create a Quadlet unit so systemd manages the container. It uses the same image and the same port (`8080`) as the container in `samples/httpbin/httpbin.yaml` from the Istio repository, which the mesh publishes on port `8000`.

The podman package does not ship the Quadlet drop-in directory, so create it first:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo mkdir -p /etc/containers/systemd"
```

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo tee /etc/containers/systemd/httpbin.container > /dev/null" <<'EOF'
[Unit]
Description=httpbin
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/mccutchen/go-httpbin:v2.15.0
Exec=/bin/go-httpbin -port 8080
Network=host

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
EOF
```

> [!IMPORTANT]
> The container runs on the host network instead of publishing port `8080`.
> A published port is DNATed by podman from `nat/PREROUTING`, and podman inserts its jump at the head of that chain - ahead of the `ISTIO_INBOUND` jump that `istio-iptables` appends.
> Inbound mesh traffic would then reach the container directly instead of being redirected to Envoy on `15006`, and the container would be handed raw mTLS bytes.
> On the host network the Istio redirect stays in front, and Envoy forwards the decrypted request to `127.0.0.1:8080`.

Start the service:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "sudo systemctl daemon-reload && sudo systemctl start httpbin"
```

> [!NOTE]
> Quadlet units are generated, so they cannot be enabled with `systemctl enable`. The `[Install]` section in the `.container` file is what makes the service start on boot.
>
> `Exec=` replaces the image's `CMD` instead of appending to it, and the go-httpbin image declares no `ENTRYPOINT`, so the binary has to be named explicitly. Dropping `/bin/go-httpbin` leaves podman trying to exec `-port`, and the service fails on startup and is restarted until systemd gives up:
>
> ```
> httpbin.service: Failed with result 'exit-code'.
> httpbin.service: Start request repeated too quickly.
> ```

Verify httpbin answers locally:

```bash
ssh -i ./ssh/vm-key "admin@${VM_ADDR}" "curl -s localhost:8080/status/200 -o /dev/null -w '%{http_code}\n'"
```
