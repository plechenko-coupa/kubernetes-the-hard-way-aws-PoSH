# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```powershell
$KubernetesPublicAddress = (Get-ELB2LoadBalancer -ProfileName $env:AWS_PROFILE -Name 'kubernetes').DNSName

kubectl config set-cluster kubernetes-the-hard-way `
  --certificate-authority=ca.pem `
  --embed-certs=true `
  --server=https://${KubernetesPublicAddress}:443

kubectl config set-credentials admin `
  --client-certificate=admin.pem `
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way `
  --cluster=kubernetes-the-hard-way `
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

## Verification

Check the version of the remote Kubernetes cluster:

```powershell
kubectl version
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

List the nodes in the remote Kubernetes cluster:

```powershell
kubectl get nodes
```

> output

```
NAME             STATUS   ROLES    AGE   VERSION
ip-172-16-1-20   Ready    <none>   22m   v1.21.0
ip-172-16-1-21   Ready    <none>   19m   v1.21.0
ip-172-16-1-22   Ready    <none>   19m   v1.21.0
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)
