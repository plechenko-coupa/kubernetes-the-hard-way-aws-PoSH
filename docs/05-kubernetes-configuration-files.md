# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.

### Kubernetes Public DNS Address

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes-the-hard-way` DNS address:

```powershell
$KubernetesPublicAddress = (Get-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE -Name 'kubernetes').DNSName
```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for each worker node:

```powershell
foreach ($i in (0..2)) {
  $InstanceName = "worker-$i"
  kubectl config set-cluster kubernetes-the-hard-way `
    --certificate-authority='ca.pem' `
    --embed-certs=true `
    --server="https://${KubernetesPublicAddress}:443" `
    --kubeconfig="$InstanceName.kubeconfig"

  kubectl config set-credentials "system:node:$InstanceName" `
    --client-certificate="$InstanceName.pem" `
    --client-key="$InstanceName-key.pem" `
    --embed-certs=true `
    --kubeconfig="$InstanceName.kubeconfig"

  kubectl config set-context default `
    --cluster=kubernetes-the-hard-way `
    --user="system:node:$InstanceName" `
    --kubeconfig="$InstanceName.kubeconfig"

  kubectl config use-context default --kubeconfig="$InstanceName.kubeconfig"
}

Get-ChildItem '*.kubeconfig'
```

Results:

```output
Cluster "kubernetes-the-hard-way" set.
User "system:node:worker-0" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes-the-hard-way" set.
User "system:node:worker-1" set.
Context "default" created.
Switched to context "default".
Cluster "kubernetes-the-hard-way" set.
User "system:node:worker-2" set.
Context "default" created.
Switched to context "default".

---

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:02 PM           6451 worker-0.kubeconfig
-a----         4/10/2022   4:02 PM           6447 worker-1.kubeconfig
-a----         4/10/2022   4:02 PM           6447 worker-2.kubeconfig
```

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```powershell
kubectl config set-cluster kubernetes-the-hard-way `
  --certificate-authority=ca.pem `
  --embed-certs=true `
  --server="https://${KubernetesPublicAddress}:443" `
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy `
  --client-certificate=kube-proxy.pem `
  --client-key=kube-proxy-key.pem `
  --embed-certs=true `
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default `
  --cluster=kubernetes-the-hard-way `
  --user=system:kube-proxy `
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

Get-ChildItem 'kube-proxy.kubeconfig'
```

Results:

```output
Cluster "kubernetes-the-hard-way" set.
User "system:kube-proxy" set.
Context "default" created.
Switched to context "default".

---

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:03 PM           6369 kube-proxy.kubeconfig
```

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```powershell
kubectl config set-cluster kubernetes-the-hard-way `
  --certificate-authority=ca.pem `
  --embed-certs=true `
  --server=https://127.0.0.1:6443 `
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager `
  --client-certificate=kube-controller-manager.pem `
  --client-key=kube-controller-manager-key.pem `
  --embed-certs=true `
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default `
  --cluster=kubernetes-the-hard-way `
  --user=system:kube-controller-manager `
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

Get-ChildItem 'kube-controller-manager.kubeconfig'
```

Results:

```output
Cluster "kubernetes-the-hard-way" set.
User "system:kube-controller-manager" set.
Context "default" modified.
Switched to context "default".

---

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:05 PM           6387 kube-controller-manager.kubeconfig


```

### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```powershell
kubectl config set-cluster kubernetes-the-hard-way `
  --certificate-authority=ca.pem `
  --embed-certs=true `
  --server=https://127.0.0.1:6443 `
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler `
  --client-certificate=kube-scheduler.pem `
  --client-key=kube-scheduler-key.pem `
  --embed-certs=true `
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default `
  --cluster=kubernetes-the-hard-way `
  --user=system:kube-scheduler `
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

Get-ChildItem 'kube-scheduler.kubeconfig'
```

Results:

```output
Cluster "kubernetes-the-hard-way" set.
User "system:kube-scheduler" set.
Context "default" created.
Switched to context "default".

---

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:06 PM           6337 kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```powershell
kubectl config set-cluster kubernetes-the-hard-way `
  --certificate-authority=ca.pem `
  --embed-certs=true `
  --server=https://127.0.0.1:6443 `
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin `
  --client-certificate=admin.pem `
  --client-key=admin-key.pem `
  --embed-certs=true `
  --kubeconfig=admin.kubeconfig

kubectl config set-context default `
  --cluster=kubernetes-the-hard-way `
  --user=admin `
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig

Get-ChildItem 'admin.kubeconfig'
```

Results:

```output
Cluster "kubernetes-the-hard-way" set.
User "admin" set.
Context "default" created.
Switched to context "default".

---

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:06 PM           6261 admin.kubeconfig
```

## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```powershell
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

  Write-Host "Copying config files to ${InstanceName}:"
  scp -i kubernetes.id_rsa ca.pem "$InstanceName.kubeconfig" kube-proxy.kubeconfig "ubuntu@${InstanceExtIp}:~/"
}
```

Result:

```output
Copying config files to worker-0:
ca.pem
worker-0.kubeconfig
kube-proxy.kubeconfig
Copying config files to worker-1:
ca.pem
worker-1.kubeconfig
kube-proxy.kubeconfig
Copying config files to worker-2:
ca.pem
worker-2.kubeconfig
kube-proxy.kubeconfig
```

Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```powershell
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

  Write-Host "Copying config files to ${InstanceName}:"
  scp -i kubernetes.id_rsa admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig "ubuntu@${InstanceExtIp}:~/"
}
```

Result:

```output
Copying config files to controller-0:
admin.kubeconfig
kube-controller-manager.kubeconfig
kube-scheduler.kubeconfig
Copying config files to controller-1:
admin.kubeconfig
kube-controller-manager.kubeconfig
kube-scheduler.kubeconfig
Copying config files to controller-2:
admin.kubeconfig
kube-controller-manager.kubeconfig
kube-scheduler.kubeconfig
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)
