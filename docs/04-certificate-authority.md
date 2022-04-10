# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Generate the CA configuration file, certificate, and private key:

```powershell
@{
  signing = @{
    default = @{
      expiry = '8760h'
    }
    profiles = @{
      kubernetes = @{
        usages = @(
          'signing'
          'key encipherment'
          'server auth'
          'client auth'
        )
        expiry = '8760h'
      }
    }
  }
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii ca-config.json

@{
  CN = 'Kubernetes'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'Kubernetes'
      OU = 'CA'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii ca-csr.json 

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

Get-ChildItem 'ca*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:53 PM           1679 ca-key.pem
-a----         4/10/2022   3:53 PM           1318 ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Generate the `admin` client certificate and private key:

```powershell
@{
  CN = 'admin'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'system:masters'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii admin-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -profile='kubernetes' `
  admin-csr.json | cfssljson -bare admin

Get-ChildItem 'admin*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:55 PM           1675 admin-key.pem
-a----         4/10/2022   3:55 PM           1428 admin.pem
```

### The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:

```powershell
foreach ($i in (0..2)){
  $InstanceName="worker-$i"
  $InstanceHostname="ip-$($K8sNodesCidrPrefix.Replace('.','-'))-1-2$i"

  @{
    CN = "system:node:$InstanceHostname"
    key = @{
      algo = 'rsa'
      size = 2048
    }
    names = @(
      @{
        C = 'US'
        L = 'Portland'
        O = 'system:nodes'
        OU = 'Kubernetes The Hard Way'
        ST = 'Oregon'
      }
    )
  } | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii "$InstanceName-csr.json"

 $Instance = Get-Ec2Instance -ProfileName $env:AWS_PROFILE `
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
  
  $InstanceExternalIp = $Instance.Instances.PublicIpAddress
  $InstanceInternalIp = $Instance.Instances.PrivateIpAddress

  cfssl gencert `
    -ca='ca.pem' `
    -ca-key='ca-key.pem' `
    -config='ca-config.json' `
    -hostname="$InstanceHostname,$InstanceExternalIp,$InstanceInternalIp" `
    -profile='kubernetes' `
    "worker-$i-csr.json" | cfssljson -bare "worker-$i"
}

Get-ChildItem 'worker*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:56 PM           1679 worker-0-key.pem
-a----         4/10/2022   3:56 PM           1509 worker-0.pem
-a----         4/10/2022   3:56 PM           1675 worker-1-key.pem
-a----         4/10/2022   3:56 PM           1509 worker-1.pem
-a----         4/10/2022   3:56 PM           1675 worker-2-key.pem
-a----         4/10/2022   3:56 PM           1509 worker-2.pem
```

### The Controller Manager Client Certificate

Generate the `kube-controller-manager` client certificate and private key:

```powershell
@{
  CN = 'system:kube-controller-manager'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'system:kube-controller-manager'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii kube-controller-manager-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -profile='kubernetes' `
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

Get-ChildItem 'kube-controller-manager*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   4:05 PM           1675 kube-controller-manager-key.pem
-a----         4/10/2022   4:05 PM           1484 kube-controller-manager.pem
```

### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:

```powershell
@{
  CN = 'system:kube-proxy'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'system:node-proxier'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii kube-proxy-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -profile='kubernetes' `
  kube-proxy-csr.json | cfssljson -bare kube-proxy

Get-ChildItem 'kube-proxy*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:56 PM           1679 kube-proxy-key.pem
-a----         4/10/2022   3:56 PM           1452 kube-proxy.pem
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```powershell
@{
  CN = 'system:kube-scheduler'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'system:kube-scheduler'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii kube-scheduler-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -profile='kubernetes' `
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

Get-ChildItem 'kube-scheduler*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:57 PM           1675 kube-scheduler-key.pem
-a----         4/10/2022   3:57 PM           1460 kube-scheduler.pem
```

### The Kubernetes API Server Certificate

The `kubernetes-the-hard-way` static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Generate the Kubernetes API Server certificate and private key:

```powershell
$KubernetesPublicAddress = (Get-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE -Name 'kubernetes').DNSName

$KubernetesHostnames = @(
  "${K8sNodesCidrPrefix}.1.10"
  "${K8sNodesCidrPrefix}.1.11"
  "${K8sNodesCidrPrefix}.1.12"
  "${K8sNodesCidrPrefix}.1.30"
  "${K8sNodesCidrPrefix}.1.31"
  "${K8sNodesCidrPrefix}.1.32"
  "${K8sClusterCidrPrefix}.0.1"
  "${K8sPodCidrPrefix}.0.1"
  $KubernetesPublicAddress
  '127.0.0.1'
  'kubernetes'
  'kubernetes.default'
  'kubernetes.default.svc'
  'kubernetes.default.svc.cluster'
  'kubernetes.svc.cluster.local'
) -join ','

@{
  CN = 'kubernetes'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'Kubernetes'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii kubernetes-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -hostname="$KubernetesHostnames" `
  -profile='kubernetes' `
  kubernetes-csr.json | cfssljson -bare kubernetes

Get-ChildItem 'kubernetes*.pem'
```

> The Kubernetes API server is automatically assigned the `kubernetes` internal dns name, which will be linked to the first IP address (`172.18.0.1`) from the address range (`172.18.0.0/24`) reserved for internal cluster services during the [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) lab.

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:58 PM           1675 kubernetes-key.pem
-a----         4/10/2022   3:58 PM           1761 kubernetes.pem
```

## The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the `service-account` certificate and private key:

```powershell
@{
  CN = 'service-accounts'
  key = @{
    algo = 'rsa'
    size = 2048
  }
  names = @(
    @{
      C = 'US'
      L = 'Portland'
      O = 'Kubernetes'
      OU = 'Kubernetes The Hard Way'
      ST = 'Oregon'
    }
  )
} | ConvertTo-Json -Depth 5 | Out-File -Encoding ascii service-account-csr.json 

cfssl gencert `
  -ca='ca.pem' `
  -ca-key='ca-key.pem' `
  -config='ca-config.json' `
  -profile='kubernetes' `
  service-account-csr.json | cfssljson -bare service-account

Get-ChildItem 'service-account*.pem'
```

Results:

```output
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         4/10/2022   3:59 PM           1679 service-account-key.pem
-a----         4/10/2022   3:59 PM           1440 service-account.pem
```

## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker, controller and etcd instance:

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

  Write-Host "Copying certificates to $InstanceName"
  scp -i kubernetes.id_rsa ca.pem "$InstanceName-key.pem" "$InstanceName.pem" "ubuntu@${InstanceExtIp}:~/"

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

  Write-Host "Copying certificates to $InstanceName"
  scp -i kubernetes.id_rsa ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem "ubuntu@${InstanceExtIp}:~/"

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

  Write-Host "Copying certificates to $InstanceName"
  scp -i kubernetes.id_rsa ca.pem kubernetes-key.pem kubernetes.pem "ubuntu@${InstanceExtIp}:~/"
}
```

> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
