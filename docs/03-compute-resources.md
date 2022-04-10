# Provisioning Compute Resources

[Guide](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md)

## Networking

### VPC

```powershell
$Vpc = New-EC2Vpc -ProfileName $env:AWS_PROFILE `
  -CidrBlock "${K8sNodesCidrPrefix}.0.0/16"
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $Vpc.VpcId `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes-the-hard-way'))
Edit-EC2VpcAttribute -ProfileName $env:AWS_PROFILE `
  -VpcId $Vpc.VpcId `
  -EnableDnsSupport $true
Edit-EC2VpcAttribute -ProfileName $env:AWS_PROFILE `
  -VpcId $Vpc.VpcId `
  -EnableDnsHostname $true

```

### Subnet

```powershell
$Subnet = New-EC2Subnet -ProfileName $env:AWS_PROFILE `
  -VpcId $Vpc.VpcId `
  -CidrBlock "${K8sNodesCidrPrefix}.1.0/24"
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $Subnet.SubnetId `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes'))

```

### Internet Gateway

```powershell
$InternetGateway = New-EC2InternetGateway -ProfileName $env:AWS_PROFILE
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $InternetGateway.InternetGatewayId `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes'))
Add-EC2InternetGateway -ProfileName $env:AWS_PROFILE `
  -VpcId $Vpc.VpcId `
  -InternetGatewayId $InternetGateway.InternetGatewayId

```

### Route Tables

```powershell
$RouteTable = New-EC2RouteTable -ProfileName $env:AWS_PROFILE `
  -VpcId $Vpc.VpcId
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $RouteTable.RouteTableId `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes'))
Register-EC2RouteTable -ProfileName $env:AWS_PROFILE `
  -RouteTableId $RouteTable.RouteTableId `
  -SubnetId $Subnet.SubnetId
New-EC2Route -ProfileName $env:AWS_PROFILE `
  -RouteTableId $RouteTable.RouteTableId `
  -DestinationCidrBlock '0.0.0.0/0' `
  -GatewayId $InternetGateway.InternetGatewayId

```

### Security Groups (aka Firewall Rules)

```powershell
# Create SG 'kubernetes'
$SecurityGroup = New-EC2SecurityGroup -ProfileName $env:AWS_PROFILE `
  -GroupName 'kubernetes' `
  -Description 'Kubernetes security group' `
  -VpcId $Vpc.VpcId
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $SecurityGroup `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes'))

# Collect the client ip of my computer
$MyIp = (Invoke-RestMethod 'https://ip.zscaler.com/index.php?json' -UseBasicParsing).clientip

# Allow access for internal networks
(Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @(
    @{IpProtocol='all'; IpRanges="${K8sNodesCidrPrefix}.0.0/16"}              # Internal traffic between nodes
    @{IpProtocol='all'; IpRanges="${K8sClusterCidrPrefix}.0.0/16"}            # Cluster internal traffic
    @{IpProtocol='all'; IpRanges="${K8sPodCidrPrefix}.0.0/16"}                # Internal traffic between pods
    @{IpProtocol='tcp'; ToPort=22; FromPort=22; IpRanges="${MyIp}/32"}        # Access to nodes via SSH from my IP address
    @{IpProtocol='tcp'; ToPort=6443; FromPort=6443; IpRanges="${MyIp}/32"}    # Access to k8s API via HTTPS from my IP address
    @{IpProtocol='tcp'; ToPort=443; FromPort=443; IpRanges="${MyIp}/32"}      # Access to ingress via HTTPS from my IP address
    @{IpProtocol='icmp'; ToPort=-1; FromPort=-1; IpRanges="${MyIp}/32"}       # Access to ICMP from my IP address
  )
).SecurityGroupRules

```

### Kubernetes Public Access - Create a Network Load Balancer

```powershell
$LoadBalancer = New-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE `
  -Name 'kubernetes' `
  -Subnet $Subnet.SubnetId `
  -Scheme internet-facing `
  -Type network
$TargetGroup = New-ELB2TargetGroup -ProfileName $env:AWS_PROFILE `
  -Name 'kubernetes' `
  -Protocol TCP `
  -Port 6443 `
  -VpcId $vpc.VpcId `
  -TargetType ip
Register-ELB2Target -ProfileName $env:AWS_PROFILE `
  -TargetGroupArn $TargetGroup.TargetGroupArn `
  -Targets @{Id = "${K8sNodesCidrPrefix}.1.10"},@{Id = "${K8sNodesCidrPrefix}.1.11"},@{Id = "${K8sNodesCidrPrefix}.1.12"}
New-ELB2Listener -ProfileName $env:AWS_PROFILE `
  -LoadBalancerArn $LoadBalancer.LoadBalancerArn `
  -Protocol TCP `
  -Port 443 `
  -DefaultAction @{Type = 'forward'; TargetGroupArn = $TargetGroup.TargetGroupArn}

```

```powershell
$KubernetesPublicAddress = $LoadBalancer.DNSName
```

## Compute Instances

### Instance Image

```powershell
$ImageUbuntu = Get-EC2Image -ProfileName $env:AWS_PROFILE `
  -Owner 099720109477 `
  -Filter @(
    @{ Name='root-device-type'; Values='ebs' }
    @{ Name='architecture'; Values='x86_64'}
    @{ Name='name'; Values='ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*'}
  ) `
  | Sort-Object -Property Name -Descending | Select-Object -First 1

$ImageWin = Get-EC2Image -ProfileName $env:AWS_PROFILE `
  -Owner 801119661308 `
  -Filter @(
    @{ Name='root-device-type'; Values='ebs' }
    @{ Name='architecture'; Values='x86_64'}
    @{ Name='name'; Values='Windows_Server-2019-English-Full-Base-*'}
  ) `
  | Sort-Object -Property Name -Descending | Select-Object -First 1  

```

### SSH Key Pair

```powershell
New-EC2KeyPair -ProfileName $env:AWS_PROFILE `
  -KeyName 'kubernetes' `
  | Select-Object -ExpandProperty KeyMaterial | Out-File -Encoding ascii kubernetes.id_rsa

# Remove Inheritance:
Icacls kubernetes.id_rsa /c /t /Inheritance:d
# Set Ownership to Owner:
Icacls kubernetes.id_rsa /c /t /Grant ${env:UserName}:F
# Remove All Users, except for Owner:
Icacls kubernetes.id_rsa /c /t /Remove Administrator BUILTIN\Administrators BUILTIN Everyone System Users 'Authenticated Users'
# Verify:
Icacls kubernetes.id_rsa

```

### Kubernetes Controllers

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process. Controller instances will be provisioned using `t3.micro` instance type.

```powershell
foreach ($i in (0..2)){
  $Instance = New-EC2Instance -ProfileName $env:AWS_PROFILE `
    -ImageId $ImageUbuntu.ImageId `
    -AssociatePublicIp $true `
    -MaxCount 1 `
    -KeyName 'kubernetes' `
    -SecurityGroupId $SecurityGroup `
    -InstanceType 't3.micro' `
    -PrivateIpAddress "${K8sNodesCidrPrefix}.1.1${i}" `
    -SubnetId $Subnet.SubnetId `
    -BlockDeviceMapping @{
      DeviceName = '/dev/sda1'
      Ebs = @{ 
        VolumeSize = 50 
      }
      NoDevice = ''
    } `
    -MetadataOptions_InstanceMetadataTag 'Enabled'

  Edit-EC2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $Instance.Instances[0].InstanceId `
    -SourceDestCheck $false

  New-EC2Tag -ProfileName $env:AWS_PROFILE `
    -Resource $Instance.Instances[0].InstanceId `
    -Tags @(
      [Amazon.EC2.Model.Tag]::new('Name',"controller-$i")
    )
  
  Write-Host "EC2 Instance 'controller-$i' ($($Instance.Instances[0].InstanceId)) created"
}

```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```powershell
foreach ($i in (0..2)){
  $Instance = New-EC2Instance -ProfileName $env:AWS_PROFILE `
    -ImageId $ImageUbuntu.ImageId `
    -AssociatePublicIp $true `
    -MaxCount 1 `
    -KeyName 'kubernetes' `
    -SecurityGroupId $SecurityGroup `
    -InstanceType 't3.micro' `
    -PrivateIpAddress "${K8sNodesCidrPrefix}.1.2$i" `
    -SubnetId $Subnet.SubnetId `
    -BlockDeviceMapping @{
      DeviceName = '/dev/sda1'
      Ebs = @{ 
        VolumeSize = 50 
      }
      NoDevice = ''
    } `
    -MetadataOptions_InstanceMetadataTag 'Enabled'

    # -UserData "name=worker-$i|os=linux|pod-cidr=${K8sPodCidrPrefix}.${i}.0/24" `
    # -EncodeUserData

  Edit-EC2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $Instance.Instances[0].InstanceId `
    -SourceDestCheck $false

  New-EC2Tag -ProfileName $env:AWS_PROFILE `
    -Resource $Instance.Instances[0].InstanceId `
    -Tags @(
      [Amazon.EC2.Model.Tag]::new('Name',"worker-$i")
      [Amazon.EC2.Model.Tag]::new('os',"linux")
      [Amazon.EC2.Model.Tag]::new('pod_cidr',"${K8sPodCidrPrefix}.${i}.0/24")
    )

  Write-Host "EC2 Instance 'worker-$i' ($($Instance.Instances[0].InstanceId)) created"
}

```

### ETCD nodes

Using `t3.micro` instances

```powershell
foreach ($i in (0..2)){
  $Instance = New-EC2Instance -ProfileName $env:AWS_PROFILE `
    -ImageId $ImageUbuntu.ImageId `
    -AssociatePublicIp $true `
    -MaxCount 1 `
    -KeyName 'kubernetes' `
    -SecurityGroupId $SecurityGroup `
    -InstanceType 't3.micro' `
    -PrivateIpAddress "${K8sNodesCidrPrefix}.1.3${i}" `
    -SubnetId $Subnet.SubnetId `
    -BlockDeviceMapping @{
      DeviceName = '/dev/sda1'
      Ebs = @{ 
        VolumeSize = 50 
      }
      NoDevice = ''
    } `
    -MetadataOptions_InstanceMetadataTag 'Enabled'

    # -UserData "name=etcd-$i" `
    # -EncodeUserData

  Edit-EC2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $Instance.Instances[0].InstanceId `
    -SourceDestCheck $false

  New-EC2Tag -ProfileName $env:AWS_PROFILE `
    -Resource $Instance.Instances[0].InstanceId `
    -Tags @(
      [Amazon.EC2.Model.Tag]::new('Name',"etcd-$i")
    )
  
  Write-Host "EC2 Instance 'etcd-$i' ($($Instance.Instances[0].InstanceId)) created"
}

```

Next: [Certificate Authority](04-certificate-authority.md)
