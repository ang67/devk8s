# devk8s

## Introduction
Here in this lesson we will install kubernetes 1.22 on centos 9

### Requirements
- VM
- 
#### Check required ports
```sh
nc 127.0.0.1 6443
```
#### Installing a container runtime (containerd)
Typically, you will have to install runc and CNI plugins from their official sites too.
- Download binary
```sh
wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz
```
-install containerd
```sh
sudo tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz
```
As the Kubernetes CRI feature has been already included in containerd-<VERSION>-<OS>-<ARCH>.tar.gz, you do not need to download the cri-containerd-.... archives to use CRI.
- start containerd
We will start containerd bia systemd, o let dowload container.service unit file and put it in /usr/local/lib/systemd/system/containerd.service

```sh
wget https://github.com/containerd/containerd/blob/main/containerd.service
cp containerd.service /usr/local/lib/systemd/system/
```
```sh
systemctl daemon-reload
systemctl enable --now containerd
```

- Install runc (runc 1.1.4)
_why???_
Download the runc. binary from https://github.com/opencontainers/runc/releases , and install it as /usr/local/sbin/runc.

```sh
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

- Installing CNI plugins
Download the cni-plugins-<OS>-<ARCH>-<VERSION>.tgz archive from https://github.com/containernetworking/plugins/releases , vand extract it under /opt/cni/bin:
```sh
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-arm64-v1.1.1.tgz

sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.1.1.tgz

/
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

Interacting with containerd via CLI using ctr or critcl
- post install
Manage Docker as a non-root user
```sh
sudo chown "$USER":"$USER" /run/containerd/ -R
sudo chmod g+rwx /run/containerd/ -R
```
Once you've created a valid configuration file, config.toml

```
version = 2

root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = 0
imports = ["/etc/containerd/runtime_*.toml", "./debug.toml"]

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0

[debug]
  address = "/run/containerd/debug.sock"
  uid = 0
  gid = 0
  level = "info"

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[plugins]
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = 0
    startup_delay = "100ms"
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = true
  [plugins."io.containerd.service.v1.tasks-service"]
    blockio_config_file = ""
    rdt_config_file = ""
```

##### Configuring the systemd cgroup driver (facultatif*)
To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
If you apply this change, make sure to restart containerd:
```sh
sudo systemctl restart containerd
```

Let configure manually cgroup driver for kubelet as we are using kubeadm 
```yaml
# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.25.0
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```
## Install tools
### Installing kubectl
```sh
curl -LO https://dl.k8s.io/release/v1.25.0/bin/linux/amd64/kubectl

chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl

kubectl version --client
```
Install CNI plugins (required for most pod network):
```sh
CNI_PLUGINS_VERSION="v1.1.1"
ARCH="amd64"
DEST="/opt/cni/bin"
sudo mkdir -p "$DEST"
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz
```

```sh
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"
```
**Install crictl (required for kubeadm / Kubelet Container Runtime Interface (CRI))**
```sh
CRICTL_VERSION="v1.25.0"
ARCH="amd64"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
```
**Install kubeadm, kubelet, kubectl and add a kubelet systemd service:**

```sh
RELEASE="v1.25.2"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
sudo chmod +x {kubeadm,kubelet}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
````

Enable and start kubelet:
```sh
systemctl enable --now kubelet
```

/usr is mounted read-only on nodes  troubleshoot
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/#usr-mounted-read-only


next https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configuring-a-cgroup-driver
