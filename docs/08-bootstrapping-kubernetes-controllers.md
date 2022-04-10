# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. Login to each controller instance using the `ssh` command. Example:

```powershell
$ControllerIPs = @{}

foreach ($i in (0..2)) {
  $InstanceName = "controller-$i"
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

  $ControllerIPs[$InstanceName] = $InstanceExtIp

  Write-Host "ssh -i kubernetes.id_rsa ubuntu@$InstanceExtIp"
}
```

Now ssh into each one of the IP addresses received in last step.

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory

Download the official Kubernetes release binaries

Install the Kubernetes binaries

```powershell
$SshScript = @'
K8S_VER=v1.21.0

DOWNLOAD_URL=\"https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/amd64\"

#Create the Kubernetes configuration directory
sudo mkdir -p /etc/kubernetes/config

#Download the official Kubernetes release binaries
wget --no-verbose --https-only --timestamping \
  \"${DOWNLOAD_URL}/kube-apiserver\" \
  \"${DOWNLOAD_URL}/kube-controller-manager\" \
  \"${DOWNLOAD_URL}/kube-scheduler\" \
  \"${DOWNLOAD_URL}/kubectl\"

#Install the Kubernetes binaries
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the Kubernetes API Server

```powershell
$SshScript = @'
sudo mkdir -p /var/lib/kubernetes/

sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster.

Create the `kube-apiserver.service` systemd unit file:

```powershell
$KubernetesPublicAddress = (Get-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE -Name 'kubernetes').DNSName

$SshScript = @"
INTERNAL_IP=`$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
INTERNAL_CIDR_PREFIX=`$(echo `${INTERNAL_IP} | cut -d'.' -f 1,2 -s)

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=`${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://`${INTERNAL_CIDR_PREFIX}.1.30:2379,https://`${INTERNAL_CIDR_PREFIX}.1.31:2379,https://`${INTERNAL_CIDR_PREFIX}.1.32:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KubernetesPublicAddress}:443 \\
  --service-cluster-ip-range=${K8sPodCidrPrefix}.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
"@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into `/var/lib/kubernetes/`.

Create the `kube-controller-manager.service` systemd unit file

```powershell
$SshScript = @"
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=${K8sClusterCidrPrefix}.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=${K8sPodCidrPrefix}.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
"@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into `/var/lib/kubernetes/`

Create the `kube-scheduler.yaml` configuration file

Create the `kube-scheduler.service` systemd unit file

```powershell
$SshScript = @'
# Move the `kube-scheduler` kubeconfig into `/var/lib/kubernetes/`
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/

# Create the `kube-scheduler.yaml` configuration file
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: \"/var/lib/kubernetes/kube-scheduler.kubeconfig\"
leaderElection:
  leaderElect: true
EOF

# Create the `kube-scheduler.service` systemd unit file
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Start the Controller Services

```powershell
$SshScript = @'
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl restart kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl status kube-apiserver kube-controller-manager kube-scheduler
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Verification

```powershell
$SshScript = @'
kubectl cluster-info --kubeconfig admin.kubeconfig
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

```output
Kubernetes control plane is running at https://127.0.0.1:6443
```

> Remember to run the above command on each controller node: `controller-0`, `controller-1`, and `controller-2`.

### Add Host File Entries

In order for `kubectl exec` commands to work, the controller nodes must each
be able to resolve the worker hostnames.  This is not set up by default in
AWS.  The workaround is to add manual host entries on each of the controller
nodes with this command:

```powershell
$SshScript = @"
for i in 0 1 2
do
  hostsline=\"$K8sNodesCidrPrefix.1.2`$i ip-$($K8sNodesCidrPrefix.Replace('.','-'))-1-2`$i\"
  grep \"`$hostsline\" /etc/hosts || (echo \"`$hostsline\" | sudo tee -a /etc/hosts)
done
"@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> If this step is missed, the [DNS Cluster Add-on](12-dns-addon.md) testing will
fail with an error like this: `Error from server: error dialing backend: dial tcp: lookup ip-10-0-1-22 on 127.0.0.53:53: server misbehaving`

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

The commands in this section will effect the entire cluster and only need to be run once from one of the controller nodes.

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```powershell
$SshScript = @'
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: \"true\"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - \"\"
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - \"*\"
EOF

kubectl --kubeconfig admin.kubeconfig describe ClusterRole system:kube-apiserver-to-kubelet
'@
$ip = $ControllerIPs['controller-0']
Write-Host "Invoking script on $i ($ip)"
ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```powershell
$SshScript = @'
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: \"\"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl --kubeconfig admin.kubeconfig describe ClusterRoleBinding system:kube-apiserver
'@
$ip = $ControllerIPs['controller-0']
Write-Host "Invoking script on $i ($ip)"
ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
```

### Verification of cluster public endpoint

> The compute instances created in this tutorial will not have permission to complete this section. **Run the following commands from the same machine used to create the compute instances**.

Make an HTTPS request for the Kubernetes version info:

```powershell
$KubernetesPublicAddress = (Get-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE -Name 'kubernetes').DNSName
curl.exe --cacert ca.pem "https://$KubernetesPublicAddress/version" -k
```

> output

```output
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
