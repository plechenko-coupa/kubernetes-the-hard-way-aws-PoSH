# Provisioning Compute Resources

[Guide](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md)

## Networking

### VPC

```powershell
$Vpc = New-EC2Vpc -ProfileName $env:AWS_PROFILE `
  -CidrBlock '172.16.0.0/16'
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
  -CidrBlock '172.16.1.0/24'
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
$SecurityGroup = New-EC2SecurityGroup -ProfileName $env:AWS_PROFILE `
  -GroupName 'kuberbnetes' `
  -Description 'Kubernetes security group' `
  -VpcId $Vpc.VpcId
New-EC2Tag -ProfileName $env:AWS_PROFILE `
  -Resource $SecurityGroup `
  -Tags ([Amazon.EC2.Model.Tag]::new('Name','kubernetes'))
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup 
  -IpPermission @{IpProtocol='all'; IpRanges='172.16.0.0/16'}
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @{IpProtocol='all'; IpRanges='172.17.0.0/16'}
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @{IpProtocol='tcp'; ToPort=22; FromPort=22; IpRanges='0.0.0.0/0'}
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @{IpProtocol='tcp'; ToPort=6443; FromPort=6443; IpRanges='0.0.0.0/0'}
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @{IpProtocol='tcp'; ToPort=443; FromPort=443; IpRanges='0.0.0.0/0'}
Grant-EC2SecurityGroupIngress -ProfileName $env:AWS_PROFILE `
  -GroupId $SecurityGroup `
  -IpPermission @{IpProtocol='icmp'; ToPort=-1; FromPort=-1; IpRanges='0.0.0.0/0'}
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
  -Targets @{Id = '172.16.1.10'},@{Id = '172.16.1.11'},@{Id = '172.16.1.12'}
New-ELB2Listener -ProfileName $env:AWS_PROFILE `
  -LoadBalancerArn $LoadBalancer.LoadBalancerArn `
  -Protocol TCP `
  -Port 443 `
  -DefaultAction @{Type = 'forward'; TargetGroupArn = $TargetGroup.TargetGroupArn} 
```

```powershell
$KubernetesPublicAddress = $LoadBalancer.DNSName
# KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers --load-balancer-arns ${LOAD_BALANCER_ARN} --output text --query 'LoadBalancers[].DNSName')
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
```

### SSH Key Pair

```powershell
New-EC2KeyPair -ProfileName $env:AWS_PROFILE `
  -KeyName 'kubernetes' `
  | Select-Object -ExpandProperty KeyMaterial | Out-File kubernetes.id_rsa

# chmod 600 kubernetes.id_rsa
```

### Kubernetes Controllers

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
    -PrivateIpAddress "172.16.1.1$i" `
    -SubnetId $Subnet.SubnetId `
    -BlockDeviceMapping @{
      DeviceName = '/dev/sda1'
      Ebs = @{ 
        VolumeSize = 50 
      }
      NoDevice = ''
    } `
    -UserData "name=controller-$i" `
    -EncodeUserData

  Edit-EC2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $Instance.Instances[0].InstanceId `
    -SourceDestCheck $false

  New-EC2Tag -ProfileName $env:AWS_PROFILE `
    -Resource $Instance.Instances[0].InstanceId `
    -Tags ([Amazon.EC2.Model.Tag]::new('Name',"controller-$i"))
  
  Write-Host "EC2 Instance 'controller-$i' ($($Instance.Instances[0].InstanceId)) created"
}
```

### Kubernetes Workers

```powershell
foreach ($i in (0..2)){
  $Instance = New-EC2Instance -ProfileName $env:AWS_PROFILE `
    -ImageId $ImageUbuntu.ImageId `
    -AssociatePublicIp $true `
    -MaxCount 1 `
    -KeyName 'kubernetes' `
    -SecurityGroupId $SecurityGroup `
    -InstanceType 't3.micro' `
    -PrivateIpAddress "172.16.1.2$i" `
    -SubnetId $Subnet.SubnetId `
    -BlockDeviceMapping @{
      DeviceName = '/dev/sda1'
      Ebs = @{ 
        VolumeSize = 50 
      }
      NoDevice = ''
    } `
    -UserData "name=worker-$i|pod-cidr=176.17.${i}.0/24" `
    -EncodeUserData

  Edit-EC2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $Instance.Instances[0].InstanceId `
    -SourceDestCheck $false

  New-EC2Tag -ProfileName $env:AWS_PROFILE `
    -Resource $Instance.Instances[0].InstanceId `
    -Tags ([Amazon.EC2.Model.Tag]::new('Name',"worker-$i"))
  
  Write-Host "EC2 Instance 'worker-$i' ($($Instance.Instances[0].InstanceId)) created"
}
```

Next: [Certificate Authority](04-certificate-authority.md)
