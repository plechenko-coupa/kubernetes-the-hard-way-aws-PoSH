# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this lab must be run on each worker instance: `worker-0`, `worker-1`, and `worker-2`. Login to each worker instance using the `ssh` command. Example:

```powershell
$WorkerIPs = @{}

foreach ($i in (0..2)) {
  $InstanceName = "worker-$i"
  $InstanceExtIp = (Get-Ec2Instance -ProfileName $env:AWS_PROFILE `
    -Filter @(
      @{
        Name = 'tag:Name'
        Values = $InstanceName
      }
      @{
        Name = 'instance-state-name'
        Values = 'running'
      }      
    )
  ).Instances.PublicIpAddress

  $WorkerIPs[$InstanceName] = $InstanceExtIp

  Write-Host "ssh -i kubernetes.id_rsa ubuntu@$InstanceExtIp"
}
```

Now ssh into each one of the IP addresses received in last step.

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```powershell
$SshScript = @'
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
'@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> The socat binary enables support for the `kubectl port-forward` command.

### Disable Swap

By default the kubelet will fail to start if [swap](https://help.ubuntu.com/community/SwapFaq) is enabled. It is [recommended](https://github.com/kubernetes/kubernetes/issues/7294) that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.

Disable swap immediately:

```powershell
$SshScript = @'
sudo swapon --show
sudo swapoff -a
sudo swapon --show
'@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> To ensure swap remains off after reboot consult your Linux distro documentation.

### Download and Install Worker Binaries

```powershell
$SshScript = @'
# Download binaries
wget --no-verbose --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet

# Create the installation directories
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

# Install the worker binaries:

mkdir containerd
tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc 
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
'@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure CNI Networking

```powershell
$SshScript = @'
# Retrieve the Pod CIDR range for the current compute instance
POD_CIDR=$(curl -s http://169.254.169.254/latest/meta-data/tags/instance/pod_cidr)
echo ${POD_CIDR}

# Create the `bridge` network configuration file
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    \"cniVersion\": \"0.4.0\",
    \"name\": \"bridge\",
    \"type\": \"bridge\",
    \"bridge\": \"cnio0\",
    \"isGateway\": true,
    \"ipMasq\": true,
    \"ipam\": {
        \"type\": \"host-local\",
        \"ranges\": [
          [{\"subnet\": \"${POD_CIDR}\"}]
        ],
        \"routes\": [{\"dst\": \"0.0.0.0/0\"}]
    }
}
EOF

# Create the `loopback` network configuration file:
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    \"cniVersion\": \"0.4.0\",
    \"name\": \"lo\",
    \"type\": \"loopback\"
}
EOF
'@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure containerd

```powershell
$SshScript = @'
# Create the `containerd` configuration file
sudo mkdir -p /etc/containerd/

# Create the containerd config file
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = \"overlayfs\"
    [plugins.cri.containerd.default_runtime]
      runtime_type = \"io.containerd.runtime.v1.linux\"
      runtime_engine = \"/usr/local/bin/runc\"
      runtime_root = \"\"
EOF

# Create the `containerd.service` systemd unit file

cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
'@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the Kubelet

```powershell
$SshScript = @"
# Retrieve worker node name
WORKER_NAME=`$(curl -s http://169.254.169.254/latest/meta-data/tags/instance/Name)
echo `$WORKER_NAME

# Move certificates to their locations
sudo mv \"`${WORKER_NAME}-key.pem\" \"`${WORKER_NAME}.pem\" /var/lib/kubelet/
sudo mv \"`${WORKER_NAME}.kubeconfig\" /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/

# Create the kubelet-config.yaml configuration file
# > The resolvConf configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`. 
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: \"/var/lib/kubernetes/ca.pem\"
authorization:
  mode: Webhook
clusterDomain: \"cluster.local\"
clusterDNS:
  - \"${K8sPodCidrPrefix}.0.10\"
podCIDR: \"`${POD_CIDR}\"
resolvConf: \"/run/systemd/resolve/resolv.conf\"
runtimeRequestTimeout: \"15m\"
tlsCertFile: \"/var/lib/kubelet/`${WORKER_NAME}.pem\"
tlsPrivateKeyFile: \"/var/lib/kubelet/`${WORKER_NAME}-key.pem\"
EOF

# Create the kubelet.service systemd unit file
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
"@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the Kubernetes Proxy

```powershell
$SshScript = @"
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

# Create the kube-proxy-config.yaml configuration file:
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: \"/var/lib/kube-proxy/kubeconfig\"
mode: \"iptables\"
clusterCIDR: \"${K8sClusterCidrPrefix}.0.0/16\"
EOF

# Create the kube-proxy.service systemd unit file:
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
"@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Start the Worker Services

```powershell
$SshScript = @"
# Add controllers and workers into the `/etc/hosts`
for i in 0 1 2
do
  hostsline=\"$K8sNodesCidrPrefix.1.1`$i ip-$($K8sNodesCidrPrefix.Replace('.','-'))-1-1`$i\"
  grep \"`$hostsline\" /etc/hosts || (echo \"`$hostsline\" | sudo tee -a /etc/hosts)
  hostsline=\"$K8sNodesCidrPrefix.1.2`$i ip-$($K8sNodesCidrPrefix.Replace('.','-'))-1-2`$i\"
  grep \"`$hostsline\" /etc/hosts || (echo \"`$hostsline\" | sudo tee -a /etc/hosts)
done

# Start Worker services
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl restart containerd kubelet kube-proxy
sudo systemctl status containerd kubelet kube-proxy
"@
foreach ($i in $WorkerIPs.Keys) {
  $ip = $WorkerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```powershell
$SshScript = @'
kubectl get nodes --kubeconfig admin.kubeconfig
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> output

```output
NAME             STATUS   ROLES    AGE   VERSION
ip-172-16-1-20   Ready    <none>   3m13s   v1.21.0
ip-172-16-1-21   Ready    <none>   16s     v1.21.0
ip-172-16-1-22   Ready    <none>   15s     v1.21.0
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
