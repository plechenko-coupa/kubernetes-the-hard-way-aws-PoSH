# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: `etcd-0`, `etcd-1`, and `etcd-2`. Login to each controller instance using the `ssh` command. Example:

```powershell
$EtcdIPs = @{}

foreach ($i in (0..2)) {
  $InstanceName = "etcd-$i"
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

  $EtcdIPs[$InstanceName] = $InstanceExtIp

  Write-Host "ssh -i kubernetes.id_rsa ubuntu@$InstanceExtIp"
}
```

Now ssh into each one of the IP addresses received in last step.

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
foreach ($i in $EtcdIPs.Keys) {
  $ip = $EtcdIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Configure the etcd Server

```powershell
$SshScript = @'
# Copy certificates
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/

# The instance internal IP address will be used to serve client requests and communicate with etcd cluster peers. 
# Retrieve the internal IP address and CIDR prefix for the current compute instance
INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
INTERNAL_CIDR_PREFIX=$(echo ${INTERNAL_IP} | cut -d'.' -f 1,2 -s)

# Each etcd member must have a unique name within an etcd cluster. 
# Set the etcd name to match the hostname of the current compute instance
ETCD_NAME=$(curl -s http://169.254.169.254/latest/meta-data/tags/instance/Name)
echo $ETCD_NAME

# Create the `etcd.service` systemd unit file
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
  --initial-cluster-state new \\
  --initial-cluster etcd-0=https://${INTERNAL_CIDR_PREFIX}.1.30:2380,etcd-1=https://${INTERNAL_CIDR_PREFIX}.1.31:2380,etcd-2=https://${INTERNAL_CIDR_PREFIX}.1.32:2380 \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
'@
foreach ($i in $EtcdIPs.Keys) {
  $ip = $EtcdIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

### Start the etcd Server

```powershell
$SshScript = @'
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl restart etcd
sudo systemctl status etcd
'@
foreach ($i in $EtcdIPs.Keys) {
  $ip = $EtcdIPs[$i]
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
foreach ($i in $EtcdIPs.Keys) {
  $ip = $EtcdIPs[$i]
  Write-Host "Invoking script on $i ($ip)"
  ssh -i kubernetes.id_rsa ubuntu@$ip $SshScript
}
```

> output

```output
89d87cd647c4ffc5, started, etcd-0, https://172.20.1.30:2380, https://172.20.1.30:2379, false
90af86bf69c20137, started, etcd-1, https://172.20.1.31:2380, https://172.20.1.31:2379, false
1fef96cbcdcf6806, started, etcd-2, https://172.20.1.32:2380, https://172.20.1.32:2379, false
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
