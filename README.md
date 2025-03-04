# Rancher Turtles AWS Cluster API Provisioning Guide

This README provides a step-by-step guide on how to provision a Kubernetes cluster on AWS using Cluster API (CAPI) with Rancher Turtles.

## Prerequisites

- A Kubernetes cluster with Rancher and Turtles installed
- `kubectl` configured to connect to your management cluster
- AWS CLI configured
- `clusterawsadm` installed

## Step 1: Create AWS IAM Resources

First, create the necessary IAM resources in your AWS account using `clusterawsadm`:

```bash
clusterawsadm bootstrap iam create-cloudformation-stack --region eu-central-1
```

This command creates the required IAM roles and policies that Cluster API will use to provision AWS resources.

## Step 2: Create Required Namespaces

Create the namespaces needed for the AWS provider and your clusters:

```bash
kubectl create ns capa-system
kubectl create ns capi-clusters
```

## Step 3: Label the Namespace for Auto-Import

Label the `capi-clusters` namespace to enable automatic importing of provisioned clusters into Rancher:

```bash
kubectl label namespace capi-clusters cluster-api.cattle.io/rancher-auto-import=true
```

## Step 4: Prepare AWS Credentials

Base64 encode your AWS credentials:

```bash
cat ~/.aws/credentials | base64
```

Create a Kubernetes secret with your AWS access keys:

```bash
kubectl create secret generic aws-credentials -n capi-clusters \
  --from-literal=AccessKeyID=<accesskey> \
  --from-literal=SecretAccessKey=<secretkey>
```

## Step 5: Install the AWS Infrastructure Provider

Create a file named `aws-provider.yaml` with the following content:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-variables
  namespace: capa-system
type: Opaque
stringData:
  AWS_B64ENCODED_CREDENTIALS: <encoded_credentials>
  ExternalResourceGC: "true"
---
apiVersion: turtles-capi.cattle.io/v1alpha1
kind: CAPIProvider
metadata:
  name: aws
  namespace: capa-system
spec:
  name: aws
  type: infrastructure
  version: v2.7.1
  configSecret:
    name: aws-variables
  variables:
    EXP_MACHINE_POOL: "true"
    EXP_EXTERNAL_RESOURCE_GC: "true"
    CAPA_LOGLEVEL: "4"
  manager:
    syncPeriod: "5m"
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSClusterStaticIdentity
metadata:
  name: aws-identity
  namespace: capi-clusters
spec:
  secretRef: aws-credentials
  allowedNamespaces:
    list:
    - capi-clusters
```

Apply this configuration:

```bash
kubectl apply -f aws-provider.yaml
```

## Understanding the Provider Configuration

The provider configuration sets up several important components:

1. **aws-variables Secret**: Contains the base64-encoded AWS credentials and enables external resource garbage collection.

2. **CAPIProvider Resource**: This is the Rancher Turtles way to install and configure Cluster API Providers:
   - `name: aws` - Specifies the AWS provider
   - `type: infrastructure` - Defines it as an infrastructure provider
   - `version: v2.7.1` - Specifies the version to install
   - `configSecret` - References the secret containing credentials and variables
   - `variables` - Enables experimental features like machine pools and external resource garbage collection
   - `manager.syncPeriod: "5m"` - Sets the controller reconciliation period

3. **AWSClusterStaticIdentity**: Creates an identity resource that:
   - References your AWS credentials secret
   - Restricts its use to the `capi-clusters` namespace

## Step 6: Create a Cluster

Create an AWS cluster definition using a template from the documentation examples or create your own definition. Apply it to your management cluster:

```bash
kubectl apply -f aws-cluster.yaml
```

## Step 7: Monitor Cluster Provisioning

Watch the cluster provisioning process:

```bash
kubectl get cluster -n capi-clusters
kubectl get machine -n capi-clusters
```

## Step 8: Access Your Cluster

Once provisioned, your cluster will automatically be imported into Rancher thanks to the auto-import label. You can access it through the Rancher UI or get the kubeconfig directly:

```bash
kubectl get secret -n capi-clusters <cluster-name>-kubeconfig -o jsonpath='{.data.value}' | base64 -d > kubeconfig
export KUBECONFIG=./kubeconfig
```

## Additional Information

- Rancher Turtles provides seamless integration between Rancher and CAPI
- The auto-import process automatically deploys the Rancher agent to your CAPI clusters
- You can customize the cluster templates based on your requirements
- For air-gapped environments, refer to the Turtles documentation for additional configuration

For more details, refer to the official Rancher Turtles documentation (https://turtles.docs.rancher.com/turtles/stable/en/index.html).
