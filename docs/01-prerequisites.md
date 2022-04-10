# Prerequisites

## Amazon Web Services

This tutorial leverages the [Amazon Web Services](https://aws.amazon.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. It would cost less then $2 for a 24 hour period that would take to complete this exercise.

> The compute resources required for this tutorial exceed the Amazon Web Services free tier. Make sure that you clean up the resource at the end of the activity to avoid incurring unwanted costs.

## OpenSSH Client for Windows

To install OpenSSH using PowerShell, run PowerShell as an Administrator. Then, install the client components as needed:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

## Amazon Web Services CLI

### Install the AWS CLI

Follow the AWS CLI [documentation](https://aws.amazon.com/cli/) to install and configure the `aws` command line utility.

Verify the AWS CLI version using:

```sh
aws --version
```

### Install the AWS Tools for PowerShell

Follow the AWS Tools for PowerShell [documentation](https://aws.amazon.com/powershell/).

Verify the AWS Tools for PowerShell version using:

```powershell
Get-AWSPowerShellVersion -ListServiceVersionInfo
```

### Set AWS Profile, Compute Region and Zone

This tutorial assumes a default compute region and zone have been configured.

Go ahead and set a default compute region:

```powershell
$env:AWS_PROFILE = 'default'
$env:AWS_REGION = 'us-east-1'
```

### Define networks used by scripts

You have to define the network prefixes:

```powershell
# CIDR prefix of AWS Subnet for Private IPs of nodes
$K8sNodesCidrPrefix = '172.20'

# CIDR prefix of k8s Cluster internal network
$K8sClusterCidrPrefix = '172.21'

# CIDR prefix of k8s POD network
$K8sPodCidrPrefix = '172.22'
```

Next: [Installing the Client Tools](02-client-tools.md)
