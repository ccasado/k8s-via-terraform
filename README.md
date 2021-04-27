# k8s-via-terraform

## Prerequisites

You will need an existing kubernetes cluster. If you don't have a cluster, you can use kind to provision a local kubernetes cluster.

```$ brew install kind````

Donwload and save the kind configuration into a filename kind-config.yaml with port mapping config to access the nginx locally.

```curl https://raw.githubusercontent.com/hashicorp/learn-terraform-deploy-nginx-kubernetes-provider/master/kind-config.yaml --output kind-config.yaml```




