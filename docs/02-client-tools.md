# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson`:

Create the local `bin` directory and add it to `$Env:PATH`.

```powershell
$BinDir = Join-Path -Path $env:USERPROFILE -ChildPath bin
New-Item -Path $BinDir -ItemType Directory -Force
if ($env:Path -split ';' -notcontains $BinDir) {
  [Environment]::SetEnvironmentVariable('PATH', $Env:PATH + ";$BinDir", [EnvironmentVariableTarget]::User)
}
```

Download the latest released Windows binaries from [CFSSL Releases Page](https://github.com/cloudflare/cfssl/releases) and copy them to the local bin directory.

```powershell
$BinDir = Join-Path -Path $env:USERPROFILE -ChildPath bin
$Releases = Invoke-RestMethod -UseBasicParsing -Uri 'https://api.github.com/repos/cloudflare/cfssl/releases/latest'
@($Releases.assets | Where-Object name -Like '*_windows_amd64.exe') | ForEach-Object {
  $OutFile = Join-Path -Path $BinDir -ChildPath (($_.Name -split '_')[0] + '.exe')
  Write-Host "Downloading $($_.browser_download_url) to $OutFile"
  Invoke-WebRequest -Uri $_.browser_download_url -OutFile $OutFile
}
```

Verify `cfssl` and `cfssljson` version 1.4.1 or higher is installed:

```sh
cfssl version
```

> output

```output
Version: 1.4.1
Runtime: go1.12.12
```

```sh
cfssljson --version
```

```output
Version: 1.4.1
Runtime: go1.12.12
```

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

Follow the [official kubectl installation instruction](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/) or install Chocolatey package `kubernetes-cli`:

```powershell
choco install kubernetes-cli
```

Verify `kubectl` version 1.21.0 or higher is installed:

```sh
kubectl version --client
```

> output

```output
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
