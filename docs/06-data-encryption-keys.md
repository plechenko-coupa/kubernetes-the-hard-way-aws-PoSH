# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```powershell
$EncryptionKey = (New-Guid).ToString().Replace('-','')
```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```powershell
@"
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $EncryptionKey
      - identity: {}
"@ | Out-File -Encoding ascii encryption-config.yaml
```

Copy the `encryption-config.yaml` encryption config file to each controller instance:

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

  Write-Host "Copying encryption config to ${InstanceName}:"
  scp -i kubernetes.id_rsa encryption-config.yaml "ubuntu@${InstanceExtIp}:~/"
}
```

Results:

```output
Copying encryption config to controller-0:
encryption-config.yaml
Copying encryption config to controller-1:
encryption-config.yaml
Copying encryption config to controller-2:
encryption-config.yaml
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
