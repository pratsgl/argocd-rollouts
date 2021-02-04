# Argo Rollouts - Kubernetes Progressive Delivery Controller
## What is Argo Rollouts ?
Argo Rollouts is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as Blue-Green, Canary, Canary analysis, Experimentation, and Progressive delivery features to Kubernetes.

Argo Rollouts (optionally) integrates with ingress controllers and service meshes, leveraging their traffic shaping abilities to gradually shift traffic to the new version during an update. Additionally, Rollouts can query and interpret metrics from various providers to verify key KPIs and drive automated promotion or rollback during an update.

This guide will demonstrate various concepts and features of Argo Rollouts by going through deployment, upgrade, promotion, and abortion of a Rollout.

## Why Argo Rollouts ?
Kubernetes Deployments provides the RollingUpdate strategy which provide a basic set of safety guarantees (readiness probes) during an update. However the rolling update strategy faces many limitations:

   *  	Few controls over the speed of the rollout
   *  	Inability to control traffic flow to the new version
   *  	Readiness probes are unsuitable for deeper, stress, or one-time checks
   *	No ability to query external metrics to verify an update
   *	Can halt the progression, but unable to automatically abort and rollback the update

For these reasons, in large scale high-volume production environments, a rolling update is often considered too risky of an update procedure since it provides no control over the blast radius, may rollout too aggressively, and provides no automated rollback upon failures.
Features

   *  	Blue-Green update strategy
   *  	Canary update strategy
   *  	Fine-grained, weighted traffic shifting
   *  	Automated rollbacks and promotions
   *  	Manual judgement
   *  	Customizable metric queries and analysis of business KPIs
   *  	Ingress controller integration: NGINX, ALB
   *  	Service Mesh integration: Istio, Linkerd, SMI
   *  	Metric provider integration: Prometheus, Wavefront, Kayenta, Web, Kubernetes Jobs

## Concepts
### Rollout
A Rollout is Kubernetes workload resource which is equivalent to a Kubernetes Deployment object. It is intended to replace a Deployment object in scenarios when more advanced deployment or progressive delivery functionality is needed. A Rollout provides the following features which a Kubernetes Deployment cannot:

* blue-green deployments
* canary deployments
* integration with ingress controllers and service meshes for advanced traffic routing
* integration with metric providers for blue-green & canary analysis
* automated promotion or rollback based on successful or failed metrics

#### Progressive Delivery

Progressive delivery is the process of releasing updates of a product in a controlled and gradual manner, thereby reducing the risk of the release, typically coupling automation and metric analysis to drive the automated promotion or rollback of the update.

Progressive delivery is often described as an evolution of continuous delivery, extending the speed benefits made in CI/CD to the deployment process. This is accomplished by limiting the exposure of the new version to a subset of users, observing and analyzing for correct behavior, then progressively increasing the exposure to a broader and wider audience while continuously verifying correctness.

#### Deployment Strategies

While the industry has used a consistent terminology to describe various deployment strategies, the implementations of these strategies tend to differ across tooling. To make it clear how the Argo Rollouts will behave, here are the descriptions of the various deployment strategies implementations offered by the Argo Rollouts.
Rolling Update

A RollingUpdate slowly replaces the old version with the new version. As the new version comes up, the old version is scaled down in order to maintain the overall count of the application. This is the default strategy of the Deployment object.
Recreate

A Recreate deployment deletes the old version of the application before bring up the new version. As a result, this ensures that two versions of the application never run at the same time, but there is downtime during the deployment.

#### Blue-Green

A Blue-Green deployment (sometimes referred to as a Red-Black) has both the new and old version of the application deployed at the same time. During this time, only the old version of the application will receive production traffic. This allows the developers to run tests against the new version before switching the live traffic to the new version.

#### Canary

A Canary deployment exposes a subset of users to the new version of the application while serving the rest of the traffic to the old version. Once the new version is verified to be correct, the new version can gradually replace the old version. Ingress controllers and service meshes such as NGINX and Istio, enable more sophisticated traffic shaping patterns for canarying than what is natively available (e.g. achieving very fine-grained traffic splitting, or splitting based on HTTP headers).

### Requirements to run argocd rollouts
---------------------
   -  Kubernetes cluster with argo-rollouts controller installed (see install guide)
      	*	https://argoproj.github.io/argo-rollouts/installation/#controller-installation
   -  kubectl with argo-rollouts plugin installed (see install guide)
      	*	https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation

### Install argo-rollouts

#### Create argo-rollouts namespace
```	
[user@kub-app001 ~]$ kubectl create namespace argo-rollouts
namespace/argo-rollouts created
```
### Download manifest install.yaml 
```
[user@kub-app001 ~]$ wget  https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
--2021-01-08 06:12:51--  https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
Resolving lab-api-proxy.dalab.mycompany.com (lab-api-proxy.dalab.mycompany.com)... 10.164.246.138
Connecting to lab-api-proxy.dalab.mycompany.com (lab-api-proxy.lab.mycompany.com)|10.164.246.138|:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: 780040 (762K) [text/plain]
Saving to: ‘install.yaml’

100%[===============================================================================================================================================>] 780,040     --.-K/s   in 0.02s

2021-01-08 06:12:51 (35.2 MB/s) - ‘install.yaml’ saved [780040/780040]
```
```
[user@kub-app001 ~]$ ls -ltr
total 768
-rw-rw-r--. 1 user user 780040 Jan  8 06:12 install.yaml
```
## Install argo-rollouts
```
[user@kub-app001 ~]$ kubectl apply -n argo-rollouts -f install.yaml
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/analysisruns.argoproj.io unchanged
customresourcedefinition.apiextensions.k8s.io/analysistemplates.argoproj.io unchanged
customresourcedefinition.apiextensions.k8s.io/clusteranalysistemplates.argoproj.io unchanged
customresourcedefinition.apiextensions.k8s.io/experiments.argoproj.io unchanged
customresourcedefinition.apiextensions.k8s.io/rollouts.argoproj.io unchanged
serviceaccount/argo-rollouts unchanged
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-admin unchanged
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-edit unchanged
clusterrole.rbac.authorization.k8s.io/argo-rollouts-aggregate-to-view unchanged
clusterrole.rbac.authorization.k8s.io/argo-rollouts-clusterrole unchanged
clusterrolebinding.rbac.authorization.k8s.io/argo-rollouts-clusterrolebinding unchanged
service/argo-rollouts-metrics unchanged
deployment.apps/argo-rollouts unchanged
```
#### Following are the resources created in argo-rollouts name space 
```
[user@kub-app001 ~]$ kubectl get all -n argo-rollouts
NAME                                 READY   STATUS    RESTARTS   AGE
pod/argo-rollouts-6f6b9bd669-nzr44   1/1     Running   0          18m

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/argo-rollouts-metrics   ClusterIP   10.106.243.232   <none>        8090/TCP   18m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argo-rollouts   1/1     1            1           18m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/argo-rollouts-6f6b9bd669   1         1         1       18m
```
### Now install argo-rollouts plugin 

```
[user@kub-app001 ~]$ wget  https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
--2021-01-08 06:18:43--  https://github-production-release-asset-2e65be.s3.amazonaws.com/158012967/8363d100-406a-11eb-98fd-e4d86ac3b339?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20210108%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210108T061843Z&X-Amz-Expires=300&X-Amz-Signature=e641acc1d453fa34e8411f608c12ddc4bc042f49e294aa5a6f0af19a8545cb65&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=158012967&response-content-disposition=attachment%3B%20filename%3Dkubectl-argo-rollouts-linux-amd64&response-content-type=application%2Foctet-stream
Proxy request sent, awaiting response... 200 OK
Length: 48541136 (46M) [application/octet-stream]
Saving to: ‘kubectl-argo-rollouts-linux-amd64’

100%[===============================================================================================================================================>] 48,541,136  7.51MB/s   in 11s

2021-01-08 06:18:54 (4.15 MB/s) - ‘kubectl-argo-rollouts-linux-amd64’ saved [48541136/48541136]
```
```
[user@kub-app001 ~]$ chmod +x ./kubectl-argo-rollouts-linux-amd64
```
```
[user@kub-app001 ~]$ sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```
#### Check if plugin "argo rollouts" works  
```
[user@kub-app001 ~]$ kubectl argo rollouts version
kubectl-argo-rollouts: v0.10.2+54343d8
  BuildDate: 2020-12-17T20:46:09Z
  GitCommit: 54343d8c9eb1e4b9f7e87b3da533530199916733
  GitTreeState: clean
  GoVersion: go1.13.1
  Compiler: gc
  Platform: linux/amd64
```
#### Source Link : https://argoproj.github.io/argo-rollouts/installation/
