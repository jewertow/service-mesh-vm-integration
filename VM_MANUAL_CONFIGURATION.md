# Configuring VM manually

These steps assume the environment variables exported during [mesh setup](README.md#mesh-setup) and [VM provisioning](VM_PROVISIONING.md) are still set: `VM_SSH_ADDR`, `EW_GATEWAY_ADDR` and `ZTUNNEL_IMAGE`.

> **Prerequisite:** The VM must have the `baseos` and `appstream` repositories enabled, plus the Docker CE repository. [vm.yaml](vm.yaml) configures all three via cloud-init.

## Step 1: Install Docker

If the VM was not provisioned from [vm.yaml](vm.yaml), add the Docker CE repository first:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo"
```

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "sudo dnf install -y --nogpgcheck --allowerasing docker-ce docker-ce-cli containerd.io"
```

> [!NOTE]
> `--allowerasing` is needed because `containerd.io` conflicts with the `runc` package that ships with podman on RHEL and CentOS Stream.

Start the daemon and enable it so the ztunnel container comes back after a reboot:

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo systemctl enable --now docker"
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker version"
```

## Step 2: Transfer configuration files

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo mkdir -p /etc/ztunnel && sudo chmod 755 /etc/ztunnel"
scp -i ./ssh/vm-key ztunnel-vm-config/root-cert.pem ztunnel-vm-config/istio-token "admin@${VM_SSH_ADDR}":~
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" \
  "sudo install -m 0644 root-cert.pem /etc/ztunnel/root-cert.pem && sudo install -m 0600 istio-token /etc/ztunnel/istio-token && rm -f root-cert.pem istio-token"
```

## Step 3: Pull the ztunnel image

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker pull ${ZTUNNEL_IMAGE}"
```

## Step 4: Run ztunnel

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker run -d \
  --name ztunnel \
  --restart=always \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --add-host istiod.istio-system.svc:${EW_GATEWAY_ADDR} \
  -v /etc/ztunnel:/etc/ztunnel:ro,Z \
  -e PROXY_MODE=dedicated \
  -e PROXY_WORKLOAD_INFO=external-vm/vm-1/vm \
  -e XDS_ADDRESS=https://istiod.istio-system.svc:15012 \
  -e CA_ADDRESS=https://istiod.istio-system.svc:15012 \
  -e XDS_ROOT_CA=/etc/ztunnel/root-cert.pem \
  -e CA_ROOT_CA=/etc/ztunnel/root-cert.pem \
  -e AUTH_TOKEN=/etc/ztunnel/istio-token \
  -e CLUSTER_ID=Kubernetes \
  -e NETWORK=vm-network \
  -e POD_NAMESPACE=external-vm \
  -e POD_NAME=vm-1 \
  -e INSTANCE_IP=${VM_IP} \
  -e RUST_LOG=info \
  ${ZTUNNEL_IMAGE}"
```

What each part is doing:

| Setting | Why |
|---|---|
| `--network host` | ztunnel's listener ports (15001 outbound, 15006 inbound plaintext, 15008 inbound HBONE) are hardcoded on a wildcard bind and are not configurable. Host networking is what lets the cluster reach 15008 at the VM's own address, and lets the admin API on `localhost:15000` be reachable from the VM. |
| `--add-host istiod...` | The east-west gateway routes by SNI, so ztunnel must dial istiod by name. See the note in [Step 5 of the mesh setup](README.md#step-5-expose-the-control-plane-to-the-vm). |
| `-v /etc/ztunnel:...:ro,Z` | The whole directory is mounted, not individual files, so that rewriting the token in place is visible inside the container. `Z` applies the SELinux label the container needs on RHEL. |
| `PROXY_MODE=dedicated` | Serves exactly one workload and skips CA impersonation, which is what makes this work outside Kubernetes. |
| `PROXY_WORKLOAD_INFO` | `<namespace>/<name>/<service-account>` — must match the `WorkloadEntry` exactly. |
| `XDS_ROOT_CA` / `CA_ROOT_CA` | The defaults are **relative** paths (`./var/run/secrets/...`). Always set them explicitly. |
| `AUTH_TOKEN` | Interpreted as a file if the path exists, **otherwise as a literal static token**. A typo'd path is not an error — it silently becomes a garbage bearer token. |
| `CLUSTER_ID` | Sent to istiod in the `clusterid` gRPC header. `Kubernetes` is the default cluster ID; change it if yours differs. |
| `POD_NAME` / `POD_NAMESPACE` / `INSTANCE_IP` | Only used to build the XDS node ID (`ztunnel~<ip>~<name>.<ns>~<ns>.svc.cluster.local`), which makes istiod's logs readable. |
| `NODE_NAME` **absent** | Deliberate. A `WorkloadEntry` has no node, and ztunnel only indexes node-local workloads by identity — unless `NODE_NAME` is unset, in which case it indexes all of them. |
| `--cap-add NET_ADMIN/NET_RAW` | Needed once the data path is in use, for freebind/transparent source-IP preservation. Not required to reach istiod; ztunnel degrades to a warning without them. |

## Step 5: Verify

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker logs ztunnel 2>&1 | tail -30"
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "curl -sf localhost:15021/healthz/ready && echo READY"
```

Continue with [Verifying the integration](README.md#verifying-the-integration).

## Refreshing the token

The service account token expires. Rewrite it in place — ztunnel reads the file on every request, so no restart is needed:

```bash
kubectl create token vm -n external-vm --audience=istio-ca --duration=24h > ztunnel-vm-config/istio-token
scp -i ./ssh/vm-key ztunnel-vm-config/istio-token "admin@${VM_SSH_ADDR}":~
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo install -m 0600 istio-token /etc/ztunnel/istio-token && rm -f istio-token"
```

## Cleanup

```bash
ssh -i ./ssh/vm-key "admin@${VM_SSH_ADDR}" "sudo docker rm -f ztunnel && sudo rm -rf /etc/ztunnel"
```
