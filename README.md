# The App Management Patterns in IBM Cloud Pak for Multicloud Management

## Concepts

[IBM Cloud Pak for Multicloud Management](https://www.ibm.com/support/knowledgecenter/SSFC4F_1.3.0/kc_welcome_cloud_pak.html), a.k.a CP4MCM, introduces a lightweight app management model for the hybrid multicloud world.

The overall model can be illustrated as following:

![misc/mcm-app-model.png](misc/mcm-app-model.png)

## Get Started

> Note: these demos have been tested in the env where includes:
> 1. The CP4MCM v1.3 deployed on the Hub Cluster on IBM OpenShift Kubernetes Service, a.k.a ROKS;
> 2. One IBM Kubernetes Service (IKS) cluster v1.15.x on IBM Cloud as the managed cluster

Before we try out these demo patterns, let's assume there is at least one managed cluster with following required tags:
- `cluster=managed-app-cluster`
- `environment=PROD`

Then we need to do two simple things:

1. To create two namespaces: `app-entitlement` and `app-project`;

2. To create necessary `ImagePolicy` for the images.

```sh
$ oc apply -f 0-preparation/1-namespace.yaml
$ oc apply -f 0-preparation/2-imagepolicy.yaml
```

> Note: the `ImagePolicy` here is really to allow almost every image, please review it while trying in production.

> OUTPUT:

```
namespace/app-entitlement created
namespace/app-project created

imagepolicy.securityenforcement.admission.cloud.ibm.com/app-image-policy created
```


## App Management Patterns

### Pattern #1: Namespace

#### Goal

The goal of this demo is to deploy a ModResort web-app to the managed cluster(s).

#### Channel

In Hub Cluster:

```sh
$ oc apply -f 1-namespace/1-channel.yaml
```

> OUTPUT:

```
channel.app.ibm.com/app-modresort-channel created
deployable.app.ibm.com/app-modresort-deployment created
deployable.app.ibm.com/app-modresort-service created
```

#### Subscription

In Hub Cluster:

```sh
$ oc apply -f 1-namespace/2-subscription.yaml
```

> OUTPUT:

```
subscription.app.ibm.com/app-modresort created
application.app.k8s.io/app-modresort created
placementrule.app.ibm.com/app-modresort created
```

#### Outcome

In Hub Cluster:

```sh
$ oc get appsub,application,placementrule -n app-project
NAME                                     STATUS       AGE
subscription.app.ibm.com/app-modresort   Propagated   1m25s

NAME                                   AGE
application.app.k8s.io/app-modresort   1m25s

NAME                                      AGE
placementrule.app.ibm.com/app-modresort   1m25s
```

In Managed Cluster:

```sh
$ kubectl get pod -n default
NAME                                        READY   STATUS    RESTARTS   AGE
app-modresort-deployment-6b9d78cc7c-l5cf2   1/1     Running   0          58s
```

> Note: the app is as expected to be placed under the managed cluster's `default` namespace.


### Pattern #2: Helm Repo

#### Goal

The goal of this demo is to deploy a Helm Chart from a Helm Repo to the managed cluster(s).

I provide two examples here:
1. The Channel points to "offical" [Kubernetes Charts repo](https://kubernetes-charts.storage.googleapis.com/) and Subscription looks for `phpmyadmin` Chart as target to deploy;
2. Another Channel points to a [custom Chart Repo](https://github.com/IBM/helm101) and Subscription looks for `guestbook` Chart as target to deploy.

> Note: 
> 1. The model of these two examples is identical;
> 2. The following process covers the first example but you can try another.

#### Channel

In Hub Cluster:

```sh
$ oc apply -f 2-helm-repo/1-channel.yaml
```

> OUTPUT:

```
channel.app.ibm.com/kubernetes-charts created
```

#### Subscription

In Hub Cluster:

```sh
$ oc apply -f 2-helm-repo/2-subscription.yaml
```

> OUTPUT:

```
subscription.app.ibm.com/app-phpmyadmin created
application.app.k8s.io/app-phpmyadmin created
placementrule.app.ibm.com/app-phpmyadmin-prod created
```

#### Outcome

In Hub Cluster:

```sh
$ oc get appsub,application,placementrule -l app=app-phpmyadmin -n app-project
NAME                                      STATUS       AGE
subscription.app.ibm.com/app-phpmyadmin   Propagated   6m52s

NAME                                    AGE
application.app.k8s.io/app-phpmyadmin   6m52s

NAME                                            AGE
placementrule.app.ibm.com/app-phpmyadmin-prod   6m52s
```

In Managed Cluster:

```sh
$ kubectl get pod -n default
NAME                                        READY   STATUS    RESTARTS   AGE
phpmyadmin-79c9b6f7bd-6nhbf                 1/1     Running   0          5m14s
```

### Pattern #3: Object Bucket

#### Goal

The goal of demo is to use a S3 Bucket as the Channel and deploy Nginx to the Managed Cluster(s).

> Note: This demo uses [IBM Cloud Object Storage](https://www.ibm.com/sg-en/cloud/object-storage) service but any s3-compliant services should work.

#### Channel

In Hub Cluster:

```sh
$ oc create secret generic object-bucket-secret \
    --from-literal=access_key_id="" \
    --from-literal=secret_access_key=""
$ oc apply -f 3-object-bucket/1-channel.yaml
```

> OUTPUT:

```
channel.app.ibm.com/minio-bucket created
```

#### Subscription

In Hub Cluster:

```sh
$ oc apply -f 3-object-bucket/2-subscription.yaml
```

> OUTPUT:

```
subscription.app.ibm.com/app-nginx created
application.app.k8s.io/app-nginx created
placementrule.app.ibm.com/app-nginx-prod created
```

#### Outcome

TODO

### Pattern #4: GitHub Repo

TODO