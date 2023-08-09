# How to configure CloudGuard Controller for Kubernetes

## Overview

This guide describes configuration steps to connect Check Point Security Management to your Kubernetes cluster to access inventory and use Kubernetes dynamic objects in Check Point policy.

## Prerequisites

- Check Point Security Management Server admin access (see [CloudGuard Controller R81 Administration Guide](https://sc1.checkpoint.com/documents/R81/WebAdminGuides/EN/CP_R81_CloudGuard_Controller_AdminGuide/Topics-CGRDG/Supported-Data-Centers-Kubernetes.htm?tocpath=Supported%20Data%20Centers%7C_____6))
- kubectl admin access to your cluster

## Steps

### 0. Obtaining admin access to my AKS cluster (depends on your environment)

- I prefer to use [Azure Cloud Shell](https://shell.azure.com/)

- I list my AKS clusters

  ```bash
  az aks list -o table
  ```

- I select my cluster

  ```bash
  az aks get-credentials --admin --name aks1 -g tf-azure-training-rg
  ```


### 1. Save CA certificate from Kubernetes cluster

- I save CA certificate from Kubernetes cluster in BASE64 format

  ```bash
  kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | tee ca.crt.b64
  ```

### 2. Create Kubernetes Service Account for CloudGuard Controller access

- I create Kubernetes Service Account for CloudGuard Controller access with recommended permissions

  ```bash
  kubectl create serviceaccount cloudguard-controller
  kubectl create clusterrole endpoint-reader --verb=get,list --resource=endpoints
  kubectl create clusterrolebinding allow-cloudguard-access-endpoints --clusterrole=endpoint-reader --serviceaccount=default:cloudguard-controller
  kubectl create clusterrole pod-reader --verb=get,list --resource=pods
  kubectl create clusterrolebinding allow-cloudguard-access-pods --clusterrole=pod-reader --serviceaccount=default:cloudguard-controller
  kubectl create clusterrole service-reader --verb=get,list --resource=services
  kubectl create clusterrolebinding allow-cloudguard-access-services --clusterrole=service-reader --serviceaccount=default:cloudguard-controller
  kubectl create clusterrole node-reader --verb=get,list --resource=nodes
  kubectl create clusterrolebinding allow-cloudguard-access-nodes --clusterrole=node-reader --serviceaccount=default:cloudguard-controller
  ```

### 3. Create and save Kubernetes Service Account token

- create secret with SA token and obtain it

  ```bash
  # create token
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Secret
  type: kubernetes.io/service-account-token
  metadata:
    name: cloudguard-controller
    annotations:
      kubernetes.io/service-account.name: "cloudguard-controller"
  EOF

  # note base64-decoded auth token:
  echo; kubectl get secret/cloudguard-controller -o json | jq -r .data.token | base64 -d | tee token.txt ; echo; echo
  ```

### 4. Kubernetes API server endpoint

- I obtain Kubernetes API server endpoint

  ```bash
  kubectl cluster-info | grep 'Kubernetes ' | awk '/http/ {print $NF}'
  ```

- example: https://aks1-8tl1rwon.hcp.westeurope.azmk8s.io:443 (will use it as whole URL including protocol prefix https://)

### 5. Create CloudGuard Controller Data Center object in Check Point Security Management

- based on data collected:
    - Kubernetes API server endpoint - URL including protocol prefix https://
    - Kubernetes Service Account token - `token.txt`
    - CA certificate from Kubernetes cluster -`ca.crt.b64`

we are ready to create CloudGuard Controller Data Center object in Check Point Security Management

