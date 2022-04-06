# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

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

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [etcd GitHub project](https://github.com/etcd-io/etcd) or [Google Storage](https://storage.googleapis.com/etcd)

Extract and install the `etcd` server and the `etcdctl` command line utility

```powershell
$SshScript = @'
ETCD_VER=v3.5.2

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download && mkdir -p /tmp/etcd-download

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

sudo mv /tmp/etcd-download/etcd* /usr/local/bin/

etcd --version
etcdctl version
etcdutl version

rm -rf /tmp/etcd-download
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the etcd Server

```powershell
$SshScript = @'
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. Retrieve the internal IP address for the current compute instance:

```sh
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
```

Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:

```sh
ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
  | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
echo "${ETCD_NAME}"
```

Create the `etcd.service` systemd unit file:

```sh
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://172.16.1.10:2380,controller-1=https://172.16.1.11:2380,controller-2=https://172.16.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

```sh
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

Below is the full script:

```powershell
$SshScript = @'
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ | tr '|' '\n' | grep '^name' | cut -d= -f2)
echo "${ETCD_NAME}"

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://172.16.1.10:2380,controller-1=https://172.16.1.11:2380,controller-2=https://172.16.1.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

## Verification

List the etcd cluster members:

```powershell
$SshScript = @'
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
'@
foreach ($i in $ControllerIPs.Keys) {
  $ip = $ControllerIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> output

```
bbeedf10f5bbaa0c, started, controller-2, https://10.0.1.12:2380, https://10.0.1.12:2379, false
f9b0e395cb8278dc, started, controller-0, https://10.0.1.10:2380, https://10.0.1.10:2379, false
eecdfcb7e79fc5dd, started, controller-1, https://10.0.1.11:2380, https://10.0.1.11:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
