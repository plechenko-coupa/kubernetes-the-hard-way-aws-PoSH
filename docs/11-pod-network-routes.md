# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table and routes

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network and use that to create route table entries.

In production workloads this functionality will be provided by CNI plugins like flannel, calico, amazon-vpc-cni-k8s. Doing this by hand makes it easier to understand what those plugins do behind the scenes.

Print the internal IP address and Pod CIDR range for each worker instance and create route table entries:

```powershell
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
  $PodCidr = $Instance.Tags.Where({$_.Key -eq 'pod_cidr'}).Value

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
(Get-EC2RouteTable -ProfileName $env:AWS_PROFILE `
  -Filter @(
    @{
      Name = 'tag:Name'
      Values = 'kubernetes'
    }
  ) `
).Routes | Format-Table -Property DestinationCidrBlock, InstanceId, GatewayId

```

> output

```output
DestinationCidrBlock InstanceId          GatewayId
-------------------- ----------          ---------
172.22.0.0/24        i-0e895e8ab1fdd8b12
172.22.1.0/24        i-07c5347a9b6829462
172.22.2.0/24        i-05f5c186fff2d58ea
172.20.0.0/16                            local
0.0.0.0/0                                igw-0c1d4a66f1314aeb9
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
