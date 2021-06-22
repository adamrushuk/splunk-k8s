<!-- omit in toc -->
# splunk-k8s

Repo for testing Splunk on Kubernetes.

<!-- omit in toc -->
## Contents

- [Install](#install)
  - [Build AKS Cluster with Terraform](#build-aks-cluster-with-terraform)
  - [Connect to AKS Cluster](#connect-to-aks-cluster)
  - [Install Splunk Operator](#install-splunk-operator)
    - [Non-admin Installation](#non-admin-installation)
  - [Install Splunk Deployment](#install-splunk-deployment)
  - [Get Admin Password](#get-admin-password)
- [Start/Stop AKS Cluster](#startstop-aks-cluster)
- [Uninstall](#uninstall)
  - [Uninstall Splunk Deployment](#uninstall-splunk-deployment)
  - [Uninstall Splunk Operator](#uninstall-splunk-operator)
  - [Destroy AKS Cluster](#destroy-aks-cluster)

## Install

### Build AKS Cluster with Terraform

Use Terraform to build the AKS cluster:

```bash
# Azure login and set Terraform env vars
source ~/path_to_azure_login_script.sh

# Init
terraform init

# Apply
terraform apply

# Outputs
terraform output
```

### Connect to AKS Cluster

The terraform output command below will return the az cli command required to get aks credentials:

```bash
# output the az cli command required to get aks credentials
terraform output aks_credentials_command
```

**Example:**  
`az aks get-credentials --resource-group <AKS_RESOURCE_GROUP_NAME> --name <AKS_CLUSTER_NAME> --overwrite-existing --admin`

### Install Splunk Operator

The [Splunk Operator for Kubernetes](https://github.com/splunk/splunk-operator) (SOK) makes it easy for Splunk
Administrators to deploy and operate Enterprise deployments in a Kubernetes infrastructure. Packaged as a container,
it uses the operator pattern to manage Splunk-specific custom resources, following best practices to manage all the
underlying Kubernetes objects for you.

Read the [Getting Started Documentation](https://github.com/splunk/splunk-operator/blob/develop/docs/README.md) for
more information.

#### Non-admin Installation

Install the Splunk Operator as a non-admin user, as the [Admin Installation for All Namespaces](https://github.com/splunk/splunk-operator/blob/develop/docs/Install.md#admin-installation-for-all-namespaces)
method has an outstanding [issue](https://github.com/splunk/splunk-operator/issues/206).

```bash
# create namespace
kubectl create namespace sok

# an admin needs to install the CRDs
kubectl apply --namespace sok -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-crds.yaml

# install splunk operator into namespace
kubectl apply --namespace sok -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-noadmin.yaml
```

### Install Splunk Deployment

First, create a ConfigMap using a license file called `enterprise.lic` (provide your own license and place in root of repo):

```bash
# create license configmap from enterprise.lic
kubectl create configmap splunk-licenses --namespace sok --from-file=enterprise.lic
```

Deploy a Splunk Validated Architecture from here: https://github.com/splunk/splunk-operator/tree/develop/deploy/examples/advanced

```bash
# deploy c1 example
kubectl apply --namespace sok -f examples/validated-arch/c1.yaml
```

### Get Admin Password

After deploying Splunk, view the admin password by running this:

```bash
kubectl get secret splunk-sok-secret --namespace sok -o jsonpath='{.data.password}' | base64 --decode
```

You can also [show all global secret values](https://github.com/splunk/splunk-operator/blob/develop/docs/Examples.md#reading-global-kubernetes-secret-object)
by running the following code:

```bash
kubectl get secret --namespace sok splunk-sok-secret -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

## Start/Stop AKS Cluster

```bash
# show current aks power state
az aks show --name <AKS_CLUSTER_NAME> --resource-group <AKS_RESOURCE_GROUP_NAME> --query "powerState"

# stop aks cluster
az aks stop --name <AKS_CLUSTER_NAME> --resource-group <AKS_RESOURCE_GROUP_NAME>

# start aks cluster
az aks start --name <AKS_CLUSTER_NAME> --resource-group <AKS_RESOURCE_GROUP_NAME>
```

## Uninstall

### Uninstall Splunk Deployment

```bash
# delete c1 example
kubectl delete --namespace sok -f examples/validated-arch/c1.yaml
```

### Uninstall Splunk Operator

```bash
# delete splunk operator
kubectl delete --namespace sok -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-noadmin.yaml

# delete CRDs
kubectl delete --namespace sok -f https://github.com/splunk/splunk-operator/releases/download/1.0.1/splunk-operator-crds.yaml

# [optional] delete namespace
kubectl delete namespace sok
```

If the namespace or other resources are stuck in a terminating state, check for remaining CRD instances, edit their
yaml and delete the `finalizers` key, eg. remove the following:

```yaml
finalizers:
  - enterprise.splunk.com/delete-pvc
```

### Destroy AKS Cluster

```bash
# destroy aks cluster
terraform destroy
```
