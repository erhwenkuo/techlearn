# 在 RKE2 Kubernetes 集群上安裝 Nvidia GPU Operator

原文: [Install a Nvidia GPU Operator on RKE2 Kubernetes Cluster](https://thenewstack.io/install-a-nvidia-gpu-operator-on-rke2-kubernetes-cluster/)

在本教程中，我們將引導您完成在 Rancher 的 RKE2 Kubernetes 發行版上安裝 NVIDIA GPU Operator 的步驟。

## Step 1: Install RKE2

SSH 進入 Ubuntu 機器並使用以下內容創建文件 `/etc/rancher/rke2/config.yaml`：

```bash title="rke2-config.bash"
sudo mkdir /etc/rancher/rke2 -p

cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"
write-kubeconfig: "/root/.kube/config"
cni: "calico"
tls-san:
  - jani-ai-testbed
  - 10.148.0.59
  - 34.87.135.216
EOF
```

此文件包含 RKE2 服務器所需的配置。不要忘記用 Ubuntu 的主機名、內部 IP 和外部 IP 地址替換 `tls-san` 配置。

下載並運行 RKE2 的安裝腳本。完成後，激活並啟用該服務以在啟動時啟動。

```bash title="install-rke2.sh"
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
./install.sh

# Enable and activate RKE2 server
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

檢查:

```bash
sudo systemctl status rke2-server
```

進程: **/usr/local/bin/rke2 server**

進程: **containerd**

- `-c /var/lib/rancher/rke2/agent/etc/containerd/config.toml`
- `-a /run/k3s/containerd/containerd.sock`
- `--state /run/k3s/containerd`
- `--root /var/lib/rancher/rke2/agent/containerd`

進程: **kublet**

- `--volume-plugin-dir=/var/lib/kubelet/volumeplugins`
- `--file-check-frequency=5s`
- `--sync-frequency=30s`
- `--address=0.0.0.0`
- `--alsologtostderr=false`
- `--anonymous-auth=false`
- `--authentication-token-webhook=true`
- `--authorization-mode=Webhook`
- `--cgroup-driver=systemd`
- `--client-ca-file=/var/lib/rancher/rke2/agent/client-ca.crt`
- `--cloud-provider=external`
- `--cluster-dns=10.43.0.10`
- `--cluster-domain=cluster.local`
- `--container-runtime-endpoint=unix:///run/k3s/containerd/containerd.sock`
- `--containerd=/run/k3s/containerd/containerd.sock`
- `--eviction-hard=imagefs.available<5%,nodefs.available<5%`
- `--eviction-minimum-reclaim=imagefs.available=10%,nodefs.available=10%`
- `--fail-swap-on=false`
- `--healthz-bind-address=127.0.0.1`
- `--hostname-override=dxlab-dt-01`
- `--kubeconfig=/var/lib/rancher/rke2/agent/kubelet.kubeconfig`
- `--log-file=/var/lib/rancher/rke2/agent/logs/kubelet.log`
- `--log-file-max-size=50`
- `--logtostderr=false`
- `--node-labels=`
- `--pod-infra-container-image=index.docker.io/rancher/pause:3.6`
- `--pod-manifest-path=/var/lib/rancher/rke2/agent/pod-manifests`
- `--read-only-port=0`
- `--resolv-conf=/run/systemd/resolve/resolv.conf`
- `--serialize-image-pulls=false`
- `--stderrthreshold=FATAL`
- `--tls-cert-file=/var/lib/rancher/rke2/agent/serving-kubelet.crt`
- `--tls-private-key-file=/var/lib/rancher/rke2/agent/serving-kubelet.key`

將包含 Kubernetes binaries 的目錄添加到路徑中，然後運行 `kubectl` 命令來檢查服務器的狀態。

```bash title="config-rke2.sh"
# add RKE2 binaries to path
export PATH=$PATH:/var/lib/rancher/rke2/bin
echo "export PATH=$PATH:/var/lib/rancher/rke2/bin" >> ~/.bashrc

# copy RKE2 kubeconfig file to the default location
mkdir ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config

# verify the configuration
kubectl get nodes
```

## Step 2: Install Helm and Patch Containerd Configuration

由於我們要通過 Helm Chart 來部署 GPU operator，所以我們先安裝 Helm 3。

```bash title="install-helm3.sh"
# install Helm3
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

下一步對於部署 Nvidia GPU Operator 最為關鍵。我們將修補配置文件以啟用對 containerd v2 支持，否則 Nvidia Container Toolkit 將無法運行。

```bash
if grep -q 'data-dir' /etc/rancher/rke2/config.yaml; then
  DATA_DIR=$(grep 'data-dir' /etc/rancher/rke2/config.yaml | awk '{print $2}')
else
  DATA_DIR=/var/lib/rancher/rke2
fi

CONTAINERD_CONFIG=$DATA_DIR/agent/etc/containerd/config.toml.tmpl

cat <<EOF | sudo tee $CONTAINERD_CONFIG
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    enable_selinux = false
    sandbox_image = "index.docker.io/rancher/pause:3.2"
    stream_server_address = "127.0.0.1"
    stream_server_port = "10010"
    [plugins."io.containerd.grpc.v1.cri".containerd]
      disable_snapshot_annotations = true
      snapshotter = "overlayfs"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.internal.v1.opt"]
    path = "/data/rancher/rke2/agent/containerd"
EOF

CONTAINERD_SOCKET=/run/k3s/containerd/containerd.sock
```


## Step 3: Deploy Nvidia GPU Operator on RKE2

```bash
# add NVIDIA repo and refresh Helm
helm repo add nvidia https://nvidia.github.io/gpu-operator && helm repo update

# install GPU operator Helm chart
helm install --wait --generate-name \
nvidia/gpu-operator $HELM_OPTIONS \
--set operator.defaultRuntime=containerd \
--set toolkit.env[0].name=CONTAINERD_CONFIG \
--set toolkit.env[0].value=$CONTAINERD_CONFIG \
--set toolkit.env[1].name=CONTAINERD_SOCKET \
--set toolkit.env[1].value=$CONTAINERD_SOCKET \
--set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
--set toolkit.env[2].value=nvidia \
--set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
--set-string toolkit.env[3].value=true
```

```bash
# add NVIDIA repo and refresh Helm
helm repo add nvidia https://nvidia.github.io/gpu-operator && helm repo update

helm upgrade --install \
gpu-operator nvidia/gpu-operator \
--create-namespace --namespace gpu-operator \
--set operator.defaultRuntime="containerd" \
--set driver.enabled=true \
--set toolkit.enabled=true \
--set toolkit.env[0].name=CONTAINERD_CONFIG \
--set toolkit.env[0].value=/var/lib/rancher/rke2/agent/etc/containerd/config.toml \
--set toolkit.env[1].name=CONTAINERD_SOCKET \
--set toolkit.env[1].value=/run/k3s/containerd/containerd.sock \
--set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
--set toolkit.env[2].value=nvidia \
--set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
--set-string toolkit.env[3].value=true
```
