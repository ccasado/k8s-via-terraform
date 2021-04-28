# k8s-via-terraform

## Objective

Use [Terraform Kubernetes provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs) to interact with kubernetes, by scheduling and exposing nginx deployment on a k8s cluster.

## Prerequisites

* Terraform
* Docker
* Kind


## Setup

You will need an existing kubernetes cluster. We will use kind to provision a local kubernetes cluster.

```
$ brew install kind
```


Donwload and save the kind configuration into a filename kind-config.yaml with port mapping config to access the nginx locally.

```
$ curl https://raw.githubusercontent.com/hashicorp/learn-terraform-deploy-nginx-kubernetes-provider/master/kind-config.yaml --output kind-config.yaml
```

Creating a cluster with kind

```
$ kind create cluster --name terraform-learn --config kind-config.yaml
Creating cluster "terraform-learn" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-terraform-learn"
You can now use your cluster with:

kubectl cluster-info --context kind-terraform-learn

Thanks for using kind! 😊

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

You can check if your cluster exists by listing your kind clusters

```
$ kind get clusters
terraform-learn
```

And use kubectl to interact with your cluster

```
$ kubectl cluster-info --context kind-terraform-learn
Kubernetes control plane is running at https://127.0.0.1:55552
KubeDNS is running at https://127.0.0.1:55552/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Configure the provider

Create a directory k8s-via-terraform and change for it

```
$ mkdir k8s-via-terraform; cd k8s-via-terraform
```

Create a file named kubernetes.tf and add the following content:

```
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
    }
  }
}

provider "kubernetes" {
  config_path = "~/.kube/config"
  config_context = "kind-terraform-learn"

}
```

And now, run terraform init to download the latest version and initialize terraform workspace

```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/kubernetes...
- Installing hashicorp/kubernetes v2.1.0...
- Installed hashicorp/kubernetes v2.1.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```


Schedule a deployment

Add the following to your kubernetes.tf file. This configuration will schedule a nginx deployment with two replicas on your cluster, internally exposing port 80 (http)

```
resource "kubernetes_deployment" "nginx" {
  metadata {
    name = "scalable-nginx-example"
    labels = {
      App = "ScalableNginxExample"
    }
  }

  spec {
    replicas = 2
    selector {
      match_labels = {
        App = "ScalableNginxExample"
      }
    }
    template {
      metadata {
        labels = {
          App = "ScalableNginxExample"
        }
      }
      spec {
        container {
          image = "nginx:1.7.8"
          name  = "example"

          port {
            container_port = 80
          }

          resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }
        }
      }
    }
  }
}
```

Apply the configuration to schedule nginx

```
$ terraform apply
```

After apply, verify the nginx deployment is running

```
$ kubectl get deployment
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
scalable-nginx-example   2/2     2            2           19h
```

Schedule a service

Now you will expose your nginx instance via NodePort to access you instance from outside the cluster

Add the following configuration to your kubernetes.tf file

```
resource "kubernetes_service" "nginx" {
  metadata {
    name = "nginx-example"
  }
  spec {
    selector = {
      App = kubernetes_deployment.nginx.spec.0.template.0.metadata[0].labels.App
    }
    port {
      node_port   = 30201
      port        = 80
      target_port = 80
    }

    type = "NodePort"
  }
}
```

Apply the configuration to schedule the NodePort service

```
$ terraform apply
```

And verify if nginx service is running

```
$ kubectl get services
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        21h
nginx-example   NodePort    10.96.100.214   <none>        80:30201/TCP   20h
```
