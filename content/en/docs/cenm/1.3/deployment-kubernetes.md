---
aliases:
- /deployment-kubernetes.html
date: '2020-01-08T09:59:25Z'
menu:
  cenm-1-3:
    identifier: cenm-1-3-deployment-kubernetes
    parent: cenm-1-3-operations
tags:
- config
- kubernetes
title: CENM Deployment with Docker, Kubernetes, and Helm charts
weight: 20
---

# CENM Deployment with Docker, Kubernetes, and Helm charts

## Introduction

This deployment guide provides a set of simple steps for deploying Corda Enterprise Network Manager (CENM)
on a Kubernetes cluster in Azure Cloud.
The deployment uses Bash scripts and Helm templates provided with CENM Docker images.

### Who is this deployment guide for?

This deployment guide is intended for use by either of the following types of CENM users:

* Any user with a moderate understanding of Kubernetes who wants to create a CENM network using default configurations.
* Software developers who wish to run a representative network in their development cycle.

### Prerequisites

The reference deployment for Corda Enterprise Network Manager runs on [Kubernetes](https://kubernetes.io/) hosted on Microsoft Azure Cloud.
Microsoft Azure provides a dedicated service to deploy a Kubernetes cluster - [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/).
You must have an active Azure subscription to be able to deploy CENM.
The next section [Deploy your network](#Deploy-your-network) contains links to the official Microsoft installation guide.
The Kubernetes cluster must have access to a private Docker repository to obtain CENM Docker images.

Your local machine operating system should be Linux, Mac OS, or a Unix-compatible environment for Windows
(for example, [Cygwin](https://www.cygwin.com/)) as the deployment uses Bash scripts.
The deployment process is driven from your local machine using a Bash script and several third-party tools:
[Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest),
[Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [Helm](https://helm.sh/).
The following section [Deploy your network](#Deploy-your-network) provides links to official installation guides of the required tools.
In addition, the CENM Command-Line Interface (CLI) tool is required so you can connect to, and manage CENM (however, this is not required for deployment).

### Compatibility

The deployment scripts are compatible with Corda Enterprise Network Manager version 1.3 only.
The deployed network runs on Kubernetes minimum version 1.8 and Helm minimum version 3.1.1.

## Deployment

### Deployment overview

The provided deployment runs all CENM services run inside a single, dedicated Kubernetes namespace (default name:`cenm`).
Each service runs in its own dedicated Kubernetes pod.

The CENM network is bootstrapped with PKI certificates, and sample X.500 subject names are provided as defaults
(for example, the Identity Manager Service certificate subject is
“CN=Test Identity Manager Service Certificate, OU=HQ, O=HoldCo LLC, L=New York, C=US”).
These can be configured in the Signer Helm chart. For more information about Signer
Helm chart refer to [Signer](deployment-kubernetes-signer.md).

There are two ways of bootstrapping a new CENM environment:

- Scripted (`bootstrap.cenm`) with **allocating new** public IP addresses.
- Scripted (`bootstrap.cenm`) with  **reusing** already allocated public IP addresses.

Use the first method for the initial bootstrap process, where there are no allocated endpoints for the services.
The bootstrapping process uses default values stored in the `values.yaml` file of each Helm chart.

{{< note >}}
The Identity Manager Service requires its public IP address or hostname to be known in advance of certificate generation, as its URL is set as the endpoint for the CRL in certificates.
{{< /note >}}

It could take a few minutes to allocate a new IP address. For subsequent deployments, you should be able to reuse existing external IP addresses.

The Network Map Service and the Signing Services have their public IP addresses allocated while bootstrapping and they do not need to be known ahead of time.

Public IP addresses are allocated as Kubernetes `LoadBalancer` services.

### Deploy your network

The deployment steps are given below:

#### 1. Install tools on your local machine

- Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

- Install [helm](https://helm.sh/docs/intro/install/)

    Ensure that the value in the `version` field for `helm version` is equal to or greater than 3.1.1,
    as shown in the example below:

    ```bash
    version.BuildInfo{Version:"v3.1.2", GitCommit:"afe70585407b420d0097d07b21c47dc511525ac8", GitTreeState:"clean", GoVersion:"go1.13.8"}
    ```
- Download CENM Command-Line Interface (CLI) tool so you can manage CENM services.

#### 2. Set up the Kubernetes cluster

- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on your machine.

- Create a cluster on Azure, following Microsoft's [quick start guide](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough).
  CENM requires a Kubernetes cluster with at least 10 GB of free memory available to all CENM services.

- Check that you have your cluster subscription [as your active subscription](https://docs.microsoft.com/en-us/cli/azure/account?view=azure-cli-latest#az-account-set).

- Connect to [your cluster](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal#connect-to-the-cluster)
  from your local machine.

#### 3. Create storage class and namespace

Run the following instruction once the previous points have been cleared:

`All the examples below use the namespace **cenm**`

```bash
kubectl apply -f deployment/k8s/cenm.yaml
export nameSpace=cenm
kubectl config set-context $(kubectl config current-context) --namespace=${nameSpace}
```

You can verify this with the command `kubectl get ns`.

#### 4. Download CENM deployment scripts

You can find the files required for the following steps in [CENM deployment repo](https://github.com/corda/cenm-deployment).

#### 5. Bootstrap CENM
**Option 1.** Bootstrap by allocating new external IP addresses

Before bootstrapping CENM, you should read the license agreement. You can do so by running `./bootstrap.cenm`.
The example below includes the `--ACCEPT_LICENSE Y` argument, which you should only specify if you accept the license agreement.

Run the following command to bootstrap a new CENM environment by allocating a new external IP:

```bash
cd network-services/deployment/k8s/helm
./bootstrap.cenm `--ACCEPT_LICENSE Y`
```

{{< note >}} The allocation of a loadbalancer to provide a public IP can take a significant amount of time (for example, even 10 minutes). {{< /note >}}

The script exits after all bootstrapping processes on the Kubernetes cluster have been started.
The process will continue to run on the cluster after the script has exited. You can monitor the completion of the deployment by running the following commands:

``` bash
kubectl get pods -o wide
```

 **Option 2.**  Bootstrap by reusing already allocated external IP addresses

If your external IPs have been already allocated you can reuse them by specifying their services names:

```bash
cd network-services/deployment/k8s/helm
./bootstrap.cenm -i idman-ip -n notary-ip
```

## Network operations

Use CENM CLI Tool to access to Farm Service:

```bash
./cenm context login -s -u <USER> -p <PASSWORD> http://<FARM-SERVICE-IP>:8080
```

The Farm Service allows you to perform all network operations on the Identity Manager Service, the Network Map Service, and the Signing Services.
The IP address is dynamically allocated for each deployment and can be found with `kubectl get svc`.
Use the following command to ensure that you are pointing at the correct namespace:

  ```bash
  kubectl config current-context && kubectl config view --minify --output 'jsonpath={..namespace}' && echo`)
  ```

### Join your network

Edit the following properties in your `node.conf` file to configure Corda node to connect to the CENM services:

```bash
networkServices {
  doormanURL="http://<IDENTITY-MANAGER-IP>:10000"
  networkMapURL="http://<NETWORK-MAP-IP>:10000"
}
```

Replacing placeholder values as follows:
  * the `doormanURL` property is the public IP address and port of the Identity Manager service
  * the `networkMapURL` is the pubic IP address and port of the Network Map service.

Next, upload the `network-root-truststore.jks` to your Corda node.
You can download it locally from the CENM Signing Service, using the following command:

```bash
kubectl cp <namespace>/<signer-pod>:DATA/trust-stores/network-root-truststore.jks network-root-truststore.jks
```

Namespace is typically `cenm` for this deployment.

Run the following command to obtain public IPs of Identity Manager and Network Map:

```bash
kubectl get svc idman-ip notary-ip
```

Run the command below to obtain the pod name for the Signer:

```bash
kubectl get pods -o wide`
```

You will find the truststore password in the `signer/files/pki.conf`, where the default value used in this Helm chart is `trust-store-password`.

{{< note >}} For more details about joining a CENM network, see:
[Joining an existing compatibility zone](https://docs.corda.net/docs/corda-os/joining-a-compatibility-zone.html).
{{< /note >}}

### Display logs

Each CENM service has a dedicated sidecar to display live logs from the ```log/``` folder.

Use the following command to display logs:

  ```bash
  kubectl logs -c logs <pod-name>
  ```

Use the following command to display live logs:

  ```bash
  kubectl logs -c logs -f <pod-name>
  ```

Display configuration files used for each CENM service

Each service stores configuration files in ```etc/``` folder in a pod.
Run the following commands to display what is in the Identity Manager Service ```etc/``` folder :

```bash
kubectl exec -it <pod name> -- ls -al etc/
Defaulting container name to main.

Use 'kubectl describe pod/idman-7699c544dc-bq9lr -n cenm' to see all of the containers in this pod.
total 10
drwxrwxrwx 2 corda corda    0 Feb 11 09:29 .
drwxr-xr-x 1 corda corda 4096 Feb 11 09:29 ..
-rwxrwxrwx 1 corda corda 1871 Feb 11 09:29 idman.conf

kubectl exec -it <pod name> -- cat etc/idman.conf
Defaulting container name to main.

Use 'kubectl describe pod/idman-7699c544dc-bq9lr -n cenm6' to see all of the containers in this pod.

address = "0.0.0.0:10000"
database {
...
```

### Update network parameters

Use CENM CLI tool command to update the network parameters.

See the official CENM documentation for more information about the list of available [network parameters](./config-network-parameters.html)
and instructions on [updating network parameters](./updating-network-parameters.html).

### Run flag day

Use the following CENM Command-Line Interface (CLI) tool command to run a Flag Day:

{{< note >}} For the changes to be advertised to the nodes, the new network map must be signed by the Signing Service.
This operation is scheduled to take place at regular intervals (by default, once every 10 seconds), as defined in the network map configuration.
{{< /note >}}

## Delete Network
There are two ways to delete your permissioned network (intended for development
environments, which are rebuilt regularly), as follows:

- delete the whole environment including IPs
- delete all CENM objects without deleting allocated external IP addresses

### Delete the whole environment including IPs

```bash
helm delete nmap notary idman signer notary-ip idman-ip
```

### Delete the whole environment without deleting IPs

If you run several ephemeral test networks in your development cycle, you might want to keep your IP addresses to speed up the process:

```bash
helm delete nmap notary idman signer
```

## Deployment Customisation

The Kubernetes scripts provided are intended to be customised depending on customer requirements.
The following sections describes how to customise various aspects of the deployment.

### Service Chart Settings

There are a number of settings provided on each Helm chart, which allow easy customisation of
common options. Each CENM service has its own dedicated page with more detailed documentation:

* [Identity Manager](deployment-kubernetes-idman.md)
* [Network Map](deployment-kubernetes-nmap.md)
* [Signer](deployment-kubernetes-signer.md)
* Farm Service - documentation not available yet
* Auth Service - documentation not available yet
* Zone Service - documentation not available yet
* [Corda Notary](deployment-kubernetes-notary.md)

### Overriding Service Configuration

The default settings used in a CENM service's configuration values can be altered as described in
[Helm guide](https://helm.sh/docs/chart_template_guide/values_files/).
In brief this can be achieved by:
* Create a separate yaml file with new values and pass it with `-f` flag: `helm install -f myvalues.yaml idman`, or;
* Override individual parameters using `--set`, such as `helm install --set foo=bar idman`, or;
* Any combination of the above, for example ```helm install -f myvalues.yaml --set foo=bar idman```

### External database support

You can configure the services to use an external database. We strongly recommend this for production deployments.
The database used by each service is configured via JDBC URL and is defined in the `values.yml` file for
the Helm chart of the respective service - for example, `idman/values.yml` for the Identity Manager Service.
In the `values.yml` file, edit the database section of the configuration to change the JDBC URL, user, and password.

The deployed service already contains JDBC drivers for PostgreSQL and SQLServer.
For an Oracle database, you need to extend the Docker images for the service by adding
the Oracle JDBC driver `.jar` file to the ` /opt/${USER}/drivers/` directory.

{{< note >}}
The bootstrap script cannot be used with an external database.
 Instead, you should run each Helm chart manually by specifying the correct database URL.
{{< /note >}}

Example settings for connection to a PostgreSQL database follow below:

```guess
database:
  driverClassName: "org.postgresql.Driver"
  url: "jdbc:postgresql://<HOST>:<PORT>/<DATABASE>"
  user: "<USER>"
  password: "<PASSWORD>"
  runMigration: "true"
```

In this example, `<HOST>` is a placeholder for the host name of the server, `<PORT>` is a placeholder for the port number the server is listening on (typically `5432` for PostgreSQL),
`<DATABASE>` is a placeholder for the database name, and `<USER>` and `<PASSWORD>` are placeholders for the access credentials of the database user.

### Memory Allocation

Default memory settings used should be adequate for most deployments, but may need to be increased for
networks with large numbers of nodes (over a thousand). The `cordaJarMx` value for each Helm chart
(in `values.yaml`) is passed to the JVM as the `-Xmx` value, and is specified in GB. Each Pod requests
memory sufficient for this value, with a limit 2GB higher than the value.

All services except the Notary use 1GB of RAM as their default `cordaJarMx`, while Notary defaults to 3GB.

## Manual bootstrap

For production deployments, you can use the bootstrap script to provide a baseline.
However, for additional flexibility you may wish to deploy each Helm chart individually.
There are several Helm commands which are used to bootstrap a new CENM environment,
where each command creates a CENM service consisting of the following:

* Signer
* Identity Manager
* Network Map
* Auth Service
* Farm Service
* Notary

They need to be run in the correct order, as shown below:

```bash
cd network-services/deployment/k8s/helm

# These Helm charts trigger public IP allocation
helm install idman-ip idman-ip
helm install notary-ip notary-ip
helm install farm-ip farm-ip

# Run these commands to display allocated public IP addresses:
kubectl get svc --namespace cenm idman-ip --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"   # step 1
kubectl get svc --namespace cenm notary-ip --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"  # step 2
kubectl get svc --namespace cenm farm-ip --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"  # step 3

# These Helm charts bootstrap CENM
helm install auth auth
helm install zone zone
helm install nmap nmap
helm install signer signer
helm install idman idman --set idmanPublicIP=[use IP from step 1]
helm install notary notary --set notaryPublicIP=[use IP from step 2]
helm install nmap nmap
helm install farm farm --set idmanPublicIP=[use IP from step 3]

# Run these commands to display allocated public IP for NetworkMap:
kubectl get svc --namespace cenm nmap --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"
```

## Appendix A: Docker Images

The Docker images used for the Kubernetes deployment are listed below for reference:

{{< table >}}

| Service           | Image Name                         | Tag |
|-------------------|------------------------------------|-----|
| Identity Manager  | acrcenm.azurecr.io/nmap/nmap       | 1.3 |
| Network Map       | acrcenm.azurecr.io/nmap/nmap       | 1.3 |
| Signer            | acrcenm.azurecr.io/signer/signer   | 1.3 |
| Zone service      | acrcenm.azurecr.io/zone/zone       | 1.3 |
| Auth service      | acrcenm.azurecr.io/auth/auth       | 1.3 |
| Farm service      | acrcenm.azurecr.io/farm/farm       | 1.3 |
| PKI Tool          | acrcenm.azurecr.io/pkitool/pkitool | 1.3 |
| Notary            | acrcenm.azurecr.io/notary/notary   | 1.3 |

{{< /table >}}