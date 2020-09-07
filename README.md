# Some Typical App Deploy Patterns in IBM Cloud Pak for Multicloud Management

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

```sh
$ oc apply -f 0-preparation/1-namespace.yaml
```

2. To create necessary `ImagePolicy` for the images.

```sh
$ oc apply -f 0-preparation/2-imagepolicy.yaml
```

> Note: the `ImagePolicy` here is really to allow almost every image, please review it while trying in production. If there is no `ImagePolicy` CRD in your env, it's fine to ignore.

> OUTPUT:

```
namespace/app-entitlement created
namespace/app-project created

imagepolicy.securityenforcement.admission.cloud.ibm.com/app-image-policy created
```

## The Typical App Deploy Patterns

There are 4 major app deploy patterns by using 4 different `Channel`s:

| Pattern # | Channel Type | Description | Refer To  |
| --- | --- | --- | --- |
| 1 | Namespace | By using `Namespace` type, we can simply wrap the native Kubernetes objects as `Deployable`s to be deployed and managed by MCM | [Pattern #1](#pattern-1-namespace)  |
| 2 | HelmRepo | `HelmRepo` might be more common so we can point the `Channel` to a particular Helm Repo and then let the `Subscrition` to specify one or more Helm Charts to be deployed and managed by MCM | [Pattern #2](#pattern-2-helm-repo)  |
| 3 | ObjectBucket | `ObjectBucket` might be another common way to embrace GitOps experience: we can point the `Channel` to a S3-compliant bucket, the manifest files in this bucket will be deployed and managed by MCM | [Pattern #3](#pattern-3-object-bucket)  |
| 4 | GitHub | GitHub repo is common to be used to store Kubernetes resources YAML files and unpackaged Helm charts. Once the `Channel` points to such a `GitHub` repo, the objects will be deployed and managed by MCM | [Pattern #4](#pattern-4-github-repo)  |


### Pattern #1: Namespace

#### Goal

The goal of this demo is to deploy a `ModResort` web-app to the managed cluster(s) by using explicit `Deployable`s in namespace.

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

There are 3 examples available under `2-helm-repo` folder:

**1. The `phpmyadmin` Chart**

- 2-helm-repo/1-channel.yaml
- 2-helm-repo/2-subscription.yaml

The `Channel` points to the **offical** [Kubernetes Charts repo](https://kubernetes-charts.storage.googleapis.com/) and `Subscription` looks for `phpmyadmin` Chart as target to deploy.


**2. The `guestbook` Chart**

- 2-helm-repo/a-channel.yaml
- 2-helm-repo/b-subscription.yaml

The `Channel` points to a [custom Chart repo](https://github.com/IBM/helm101) and `Subscription` looks for `guestbook` Chart as target to deploy.

**3. The `bookinfo` Chart**

- 2-helm-repo/x-channel-bookinfo.yaml
- 2-helm-repo/y-subscription-bookinfo.yaml

The `Channel` points to another [custom Chart repo](https://raw.githubusercontent.com/dymaczew/charts/master/repo/incubator/) and `Subscription` looks for the famous `bookinfo` example originated from Istio as target to deploy.
Kudos to @dymaczew who has made them instumented with ICAM's [runtime data collectors](https://www.ibm.com/support/knowledgecenter/en/SSFC4F_1.3.0/icam/dc_runtime_intro.html) so you can get the 4 golden sigals staightaway!
Do remember to follow the doc [here](https://www.ibm.com/support/knowledgecenter/en/SSFC4F_1.3.0/icam/dc_config_server_info.html) to download the configuration file named `ibm-cloud-apm-dc-configpack.tar` from ICAM and then prepare the secret in the managed cluster(s).

```sh
# The file `ibm-cloud-apm-dc-configpack.tar` is downloaded from ICAM
$ tar xvf ibm-cloud-apm-dc-configpack.tar
$ cd ibm-cloud-apm-dc-configpack

# Create the secret in the namespace where the app will be deployed
# so that the runtime data collectors can talk to ICAM for 4 golden signals
# here we use the default namespace, but feel free to change to your desired one
$ kubectl create secret generic icam-server-secret -n default \
--from-file=keyfiles/keyfile.jks \
--from-file=keyfiles/keyfile.p12 \
--from-file=keyfiles/keyfile.kdb \
--from-file=keyfiles/ca.pem \
--from-file=keyfiles/cert.pem \
--from-file=keyfiles/key.pem \
--from-file=global.environment
``` 

#### Channel

> The following process covers the first example but you can try anyone you want.

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

The goal of this demo is to use a S3 Bucket as the Channel and deploy `Nginx` app to the Managed Cluster(s).

#### Prerequisites

1. S3-compliant server is accessible;
2. A bucket is created with deployable Kubernetes objects copied inside.

> Note: Please refer to [Annex -> Setup MinIO](#setup-minio) for a quick guide for MinIO.

#### Channel

In Hub Cluster:

> Note: change the 

```sh
$ oc create secret generic object-bucket-secret \
    --from-literal=AccessKeyID='<THE ACCESSKEY>' \              # CHANGE ME!!
    --from-literal=SecretAccessKey='<THE SECRETKEY>'            # CHANGE ME!!

$ OBJECT_BUCKET_URL='http:\/\/your.minio.url\/bucket-name'      # CHANGE ME!!
$ cat 3-object-bucket/1-channel.yaml | \
    sed "s/<EXPOSED-OBJECT-BUCKET-URL>/$OBJECT_BUCKET_URL/g" | \
    oc apply -f -
```

> Note: 
> 1. The OBJECT_BUCKET_URL pattern must be `<HTTP/HTTPS>://<DOMAIN>/<BUCKET>`. For example: http://my-minio.example.com/my-bucket;
> 2. As here we're using `sed` to replace the content, we have to escape the special char `/`.
> 3. For those who are using IBM Cloud Object Storage service, you may enable the [HMAC Credential](https://cloud.ibm.com/docs/cloud-object-storage?topic=cloud-object-storage-uhc-hmac-credentials-main) to create such a key pair and create the secret accordingly.


> OUTPUT:

```
channel.app.ibm.com/object-bucket created
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

In Hub Cluster:

```sh
$ oc get appsub,application,placementrule -l app=app-nginx -n app-project
NAME                                 STATUS       AGE
subscription.app.ibm.com/app-nginx   Propagated   4m42s

NAME                               AGE
application.app.k8s.io/app-nginx   4m42s

NAME                                       AGE
placementrule.app.ibm.com/app-nginx-prod   4m41s
```

In Managed Cluster:

```sh
$ kubectl get pod -n default
kubectl get deploy,pod -n default
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx   1/1     1            1           2m3s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-78646cb555-cb6m8   1/1     Running   0          2m3s

$ kubectl get deploy,pod -n default
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx   1/1     1            1           2m57s

NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-78646cb555-cb6m8   1/1     Running   0          2m57s
```

### Pattern #4: GitHub Repo

TODO


## Annex

### Setup MinIO

This is a very simple guide about how to setup MinIO server in OpenShift by using Helm Chart to support Pattern #3.

You may refer to the official docs [here](https://github.com/helm/charts/tree/master/stable/minio) for more details.

```sh
$ MINIO_NANESPACE=app-entitlement           # CHANGE ME!!
$ MINIO_HELM_CHART_NAME=my-minio            # CHANGE ME!!

$ oc create sa $MINIO_HELM_CHART_NAME

# To grant permission for the sa MinIO uses
$ oc adm policy add-scc-to-user anyuid system:serviceaccount:$MINIO_NANESPACE:my-minio
# This shouldn't need any more as my PR (https://github.com/helm/charts/pull/21907) has been merged
$ oc adm policy add-scc-to-user anyuid system:serviceaccount:$MINIO_NANESPACE:default

$ helm install --name $MINIO_HELM_CHART_NAME stable/minio --tls \
    --set persistence.enabled=false \
    --set serviceAccount.create=false \
    --set serviceAccount.name=$MINIO_HELM_CHART_NAME \
    --set "buckets[0].name=my-bucket" \
    --set "buckets[0].policy=none"

$ oc expose service/my-minio
$ oc get routes
NAME       HOST/PORT                                                                                                      PATH   SERVICES   PORT   TERMINATION   WILDCARD
my-minio   <EXPOSED MINIO URL>          my-minio   http                 None
```

> Notes:

1. Take note of the `accessKey` and `secretKey` generated. If you didn't set that, there has already one pair by default -- assuming we're going to use the default key pair for simplicity:

```
accessKey: "AKIAIOSFODNN7EXAMPLE"
secretKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

2. Take note of the `<EXPOSED MINIO URL>` for further configuration in `Channel`.


Lastly, we need to copy over the manifest files into the `my-bucket`.

You may do it through UI, or CLI as blow:

```sh
$ mc config host add my-minio http://<EXPOSED MINIO URL> AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

$ mc cp 3-object-bucket/objects-in-bucket/deploy.yaml my-minio/my-bucket
$ mc cp 3-object-bucket/objects-in-bucket/service.yaml my-minio/my-bucket

$ mc ls my-minio/my-bucket
[2020-04-16 11:35:01 +08]    339B deploy.yaml
[2020-04-16 11:35:46 +08]    169B service.yaml
```
