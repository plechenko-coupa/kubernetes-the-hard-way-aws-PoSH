# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table and routes

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network and use that to create route table entries.

In production workloads this functionality will be provided by CNI plugins like flannel, calico, amazon-vpc-cni-k8s. Doing this by hand makes it easier to understand what those plugins do behind the scenes.

Print the internal IP address and Pod CIDR range for each worker instance and create route table entries:

```powershell
$WorkerIPs = @{}

foreach ($i in (0..2)) {
  $InstanceName = "worker-$i"
  $Instance = (Get-Ec2Instance -ProfileName $env:AWS_PROFILE `
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
  ).Instances[0]

  $InstanceId = $Instance.InstanceId
  $InstanceIp = $Instance.PrivateIpAddress
  
  $InstanceUserData = Get-Ec2InstanceAttribute -ProfileName $env:AWS_PROFILE `
    -InstanceId $InstanceId `
    -Attribute userData
  $PodCidr = ([System.Text.Encoding]::ASCII.GetString(
      [System.Convert]::FromBase64String($InstanceUserData.UserData)
    ) -split '\|' `
    | Where-Object {$_ -like 'pod-cidr*'}) -split '=' `
    | Select-Object -Last 1

  Write-Host "$InstanceIp $PodCidr"

  $RouteTable = (Get-EC2RouteTable -ProfileName $env:AWS_PROFILE `
    -Filter @(
      @{
        Name = 'tag:Name'
        Values = 'kubernetes'
      }
    )
  )

  New-EC2Route -ProfileName $env:AWS_PROFILE `
    -RouteTableId $RouteTable.RouteTableId `
    -DestinationCidrBlock $PodCidr `
    -InstanceId $InstanceId

}
```

> output

```output
172.16.1.20 176.17.0.0/24
True
172.16.1.21 176.17.1.0/24
True
172.16.1.22 176.17.2.0/24
True
```

## Validate Routes

Validate network routes for each worker instance:

```powershell
Get-EC2RouteTable -ProfileName $env:AWS_PROFILE `
  -Filter @(
    @{
      Name = 'tag:Name'
      Values = 'kubernetes'
    }
  ) `
  | Format-Table

```

> output

```output
CarrierGatewayId DestinationCidrBlock DestinationIpv6CidrBlock DestinationPrefixListId EgressOnlyInternetGatewayId GatewayId             InstanceId          InstanceOwnerId LocalGatewayId NatGatewayId
---------------- -------------------- ------------------------ ----------------------- --------------------------- ---------             ----------          --------------- -------------- ------------
                 176.17.0.0/24                                                                                                           i-09b469bbe25deff9a 660200843256
                 176.17.1.0/24                                                                                                           i-0a20f395802486a24 660200843256
                 172.16.0.0/16                                                                                     local
                 0.0.0.0/0                                                                                         igw-0ee0f75fa0a1bc4db
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
