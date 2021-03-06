---
title: "Troubleshoot common Azure Arc enabled Kubernetes issues (Preview)"
services: azure-arc
ms.service: azure-arc
#ms.subservice: azure-arc-kubernetes coming soon
ms.date: 05/19/2020
ms.topic: article
author: mlearned
ms.author: mlearned
description: "Troubleshooting common issues with Arc enabled Kubernetes clusters."
keywords: "Kubernetes, Arc, Azure, containers"
---

# Azure Arc enabled Kubernetes troubleshooting (Preview)

This document provides some common troubleshooting scenarios with connectivity, permissions, and agents.

## General troubleshooting

### Azure CLI set up
Before using az connectedk8s or az k8sconfiguration CLI commands, assure that az is set to work against the correct Azure subscription.

```azurecli
az account set --subscription 'subscriptionId'
az account show
```

### azure-arc agents
All agents for Azure Arc enabled Kubernetes are deployed as pods in the `azure-arc` namespace. Under normal operations all pods should be running and passing their health checks.

First, verify the Azure Arc helm release:

```console
$ helm --namespace default status azure-arc
NAME: azure-arc
LAST DEPLOYED: Fri Apr  3 11:13:10 2020
NAMESPACE: default
STATUS: deployed
REVISION: 5
TEST SUITE: None
```

If the Helm release is not found or missing, try onboarding the cluster again.

If the Helm release is present and `STATUS: deployed` determine the status of the agents using `kubectl`:

```console
$ kubectl -n azure-arc get deployments,pods
NAME                                       READY  UP-TO-DATE  AVAILABLE  AGE
deployment.apps/clusteridentityoperator     1/1       1          1       16h
deployment.apps/config-agent                1/1       1          1       16h
deployment.apps/cluster-metadata-operator   1/1       1          1       16h
deployment.apps/controller-manager          1/1       1          1       16h
deployment.apps/flux-logs-agent             1/1       1          1       16h
deployment.apps/metrics-agent               1/1       1          1       16h
deployment.apps/resource-sync-agent         1/1       1          1       16h

NAME                                            READY   STATUS  RESTART  AGE
pod/cluster-metadata-operator-7fb54d9986-g785b  2/2     Running  0       16h
pod/clusteridentityoperator-6d6678ffd4-tx8hr    3/3     Running  0       16h
pod/config-agent-544c4669f9-4th92               3/3     Running  0       16h
pod/controller-manager-fddf5c766-ftd96          3/3     Running  0       16h
pod/flux-logs-agent-7c489f57f4-mwqqv            2/2     Running  0       16h
pod/metrics-agent-58b765c8db-n5l7k              2/2     Running  0       16h
pod/resource-sync-agent-5cf85976c7-522p5        3/3     Running  0       16h
```

All Pods should show `STATUS` as `Running` and `READY` should be either `3/3` or `2/2`. Fetch logs and describe pods that are returning `Error` or `CrashLoopBackOff`. If any of these pods are stuck in `Pending` state it could be because of insufficient resources on cluster nodes. [Scaling up your cluster](https://kubernetes.io/docs/tasks/administer-cluster/) will get these pods to transition to `Running` state.

## Connecting Kubernetes clusters to Azure Arc

Connecting clusters to Azure requires access to both an Azure subscription and `cluster-admin` access to a target cluster. If the cluster cannot be reached or has insufficient permissions onboarding will fail.

### Insufficient cluster permissions

If the provided kubeconfig file does not have sufficient permissions to install the Azure Arc agents, the Azure CLI command will return an error attempting to call the Kubernetes API.

```azurecli
$ az connectedk8s connect --resource-group AzureArc --name AzureArcCluster
Command group 'connectedk8s' is in preview. It may be changed/removed in a future release.
Ensure that you have the latest helm version installed before proceeding to avoid unexpected errors.
This operation might take a while...

Error: list: failed to list: secrets is forbidden: User "myuser" cannot list resource "secrets" in API group "" at the cluster scope
```

Cluster owner should use a Kubernetes user with cluster administrator permissions.

### Installation timeouts

Azure Arc agent installation requires running a set of containers on the target cluster. If the cluster is running over a slow internet connection the container image pull may take longer than the Azure CLI timeouts.

```azurecli
$ az connectedk8s connect --resource-group AzureArc --name AzureArcCluster
Command group 'connectedk8s' is in preview. It may be changed/removed in a future release.
Ensure that you have the latest helm version installed before proceeding to avoid unexpected errors.
This operation might take a while...
```

### Helm issue

Helm `v3.3.0-rc.1` version has an [issue](https://github.com/helm/helm/pull/8527) where helm install/upgrade (used under the hood by connectedk8s CLI extension) results in running of all hooks leading to the following error:

```console
$ az connectedk8s connect -n shasbakstest -g shasbakstest
Command group 'connectedk8s' is in preview. It may be changed/removed in a future release.
Ensure that you have the latest helm version installed before proceeding.
This operation might take a while...

Please check if the azure-arc namespace was deployed and run 'kubectl get pods -n azure-arc' to check if all the pods are in running state. A possible cause for pods stuck in pending state could be insufficientresources on the kubernetes cluster to onboard to arc.
ValidationError: Unable to install helm release: Error: customresourcedefinitions.apiextensions.k8s.io "connectedclusters.arc.azure.com" not found
```

To recover from this issue, follow these steps:

1. Delete the Azure Arc enabled Kubernetes resource of concern in the Azure portal.
2. Run the following commands on your machine:
    
    ```console
    kubectl delete ns azure-arc
    kubectl delete clusterrolebinding azure-arc-operator
    kubectl delete secret sh.helm.release.v1.azure-arc.v1
    ```

3. [Install a stable version](https://helm.sh/docs/intro/install/) of Helm 3 on your machine instead of the release candidate version.
4. Run the `az connectedk8s connect` command with the appropriate values to connect the cluster to Azure Arc.

## Configuration management

### General
To help troubleshoot issues with source control configuration, run az commands with --debug switch.

```console
az provider show -n Microsoft.KubernetesConfiguration --debug
az k8sconfiguration create <parameters> --debug
```

### Create source control configuration
Contributor role on the Microsoft.Kubernetes/connectedCluster resource is necessary and sufficient for creating Microsoft.KubernetesConfiguration/sourceControlConfiguration resource.

### Configuration remains `Pending`

```console
kubectl -n azure-arc logs -l app.kubernetes.io/component=config-agent -c config-agent
$ k -n pending get gitconfigs.clusterconfig.azure.com  -o yaml
apiVersion: v1
items:
- apiVersion: clusterconfig.azure.com/v1beta1
  kind: GitConfig
  metadata:
    creationTimestamp: "2020-04-13T20:37:25Z"
    generation: 1
    name: pending
    namespace: pending
    resourceVersion: "10088301"
    selfLink: /apis/clusterconfig.azure.com/v1beta1/namespaces/pending/gitconfigs/pending
    uid: d9452407-ff53-4c02-9b5a-51d55e62f704
  spec:
    correlationId: ""
    deleteOperator: false
    enableHelmOperator: false
    giturl: git@github.com:slack/cluster-config.git
    helmOperatorProperties: null
    operatorClientLocation: azurearcfork8s.azurecr.io/arc-preview/fluxctl:0.1.3
    operatorInstanceName: pending
    operatorParams: '"--disable-registry-scanning"'
    operatorScope: cluster
    operatorType: flux
  status:
    configAppliedTime: "2020-04-13T20:38:43.081Z"
    isSyncedWithAzure: true
    lastPolledStatusTime: ""
    message: 'Error: {exit status 1} occurred while doing the operation : {Installing
      the operator} on the config'
    operatorPropertiesHashed: ""
    publicKey: ""
    retryCountPublicKey: 0
    status: Installing the operator
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
## Monitoring

Azure Monitor for containers requires its DaemonSet to be run in privileged mode. To successfully set up a Canonical Charmed Kubernetes cluster for monitoring, run the following command:

```console
juju config kubernetes-worker allow-privileged=true
```