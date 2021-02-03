# Argo Rollouts - Kubernetes Progressive Delivery Controller
#### What is Argo Rollouts?
---------------------
Argo Rollouts is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

Argo Rollouts (optionally) integrates with ingress controllers and service meshes, leveraging their traffic shaping abilities to gradually shift traffic to the new version during an update. Additionally, Rollouts can query and interpret metrics from various providers to verify key KPIs and drive automated promotion or rollback during an update.

This guide will demonstrate various concepts and features of Argo Rollouts by going through deployment, upgrade, promotion, and abortion of a Rollout.

#### Why Argo Rollouts ?
---------------------
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


### Requirements to run argocd rollouts
---------------------
   -  Kubernetes cluster with argo-rollouts controller installed (see install guide)
      	*	https://argoproj.github.io/argo-rollouts/installation/#controller-installation
   -  kubectl with argo-rollouts plugin installed (see install guide)
      	*	https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation

### Install argo-rollouts controller on K8s Cluster
---------------------

```	
[user@kub-app001 ~]$ kubectl create namespace argo-rollouts
namespace/argo-rollouts created

[user@kub-app001 ~]$ wget  https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
--2021-01-08 06:12:51--  https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
Resolving lab-api-proxy.dalab.mycompany.com (lab-api-proxy.dalab.mycompany.com)... 10.164.246.138
Connecting to lab-api-proxy.dalab.mycompany.com (lab-api-proxy.lab.mycompany.com)|10.164.246.138|:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: 780040 (762K) [text/plain]
Saving to: ‘install.yaml’

100%[===============================================================================================================================================>] 780,040     --.-K/s   in 0.02s

2021-01-08 06:12:51 (35.2 MB/s) - ‘install.yaml’ saved [780040/780040]

[user@kub-app001 ~]$ ls -ltr
total 768
-rw-rw-r--. 1 user user 780040 Jan  8 06:12 install.yaml

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
---------------------
```
[user@kub-app001 ~]$ curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (56) Received HTTP code 504 from proxy after CONNECT

[user@kub-app001 ~]$ wget  https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
--2021-01-08 06:18:43--  https://github-production-release-asset-2e65be.s3.amazonaws.com/158012967/8363d100-406a-11eb-98fd-e4d86ac3b339?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20210108%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210108T061843Z&X-Amz-Expires=300&X-Amz-Signature=e641acc1d453fa34e8411f608c12ddc4bc042f49e294aa5a6f0af19a8545cb65&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=158012967&response-content-disposition=attachment%3B%20filename%3Dkubectl-argo-rollouts-linux-amd64&response-content-type=application%2Foctet-stream
Connecting to lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)|10.164.246.138|:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: 48541136 (46M) [application/octet-stream]
Saving to: ‘kubectl-argo-rollouts-linux-amd64’

100%[===============================================================================================================================================>] 48,541,136  7.51MB/s   in 11s

2021-01-08 06:18:54 (4.15 MB/s) - ‘kubectl-argo-rollouts-linux-amd64’ saved [48541136/48541136]

[user@kub-app001 ~]$ chmod +x ./kubectl-argo-rollouts-linux-amd64
[user@kub-app001 ~]$ sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

[user@kub-app001 ~]$ kubectl argo rollouts version
kubectl-argo-rollouts: v0.10.2+54343d8
  BuildDate: 2020-12-17T20:46:09Z
  GitCommit: 54343d8c9eb1e4b9f7e87b3da533530199916733
  GitTreeState: clean
  GoVersion: go1.13.1
  Compiler: gc
  Platform: linux/amd64
```
## Getting started with argocd rollouts
---------------------
- Document Link:  https://argoproj.github.io/argo-rollouts/getting-started/

```
[user@kub-app001 ~]$ mkdir -p kubernetes/argocd-rollout-demo
[user@kub-app001 ~]$ cd kubernetes/argocd-rollout-demo/
```
Download rollout.yaml & service.yaml files 
```
[user@kub-app001 kubernetes]$ wget  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
--2021-01-08 06:24:38--  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
Resolving lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)... 10.164.246.138
Connecting to lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)|10.164.246.138|:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: 752 [text/plain]
Saving to: ‘rollout.yaml’

100%[===============================================================================================================================================>] 752         --.-K/s   in 0s

2021-01-08 06:24:38 (36.0 MB/s) - ‘rollout.yaml’ saved [752/752]

[user@kub-app001 kubernetes]$ wget https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
--2021-01-08 06:24:56--  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
Resolving lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)... 10.164.246.138
Connecting to lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)|10.164.246.138|:8080... connected.
Proxy request sent, awaiting response... 200 OK
Length: 178 [text/plain]
Saving to: ‘service.yaml’

100%[===============================================================================================================================================>] 178         --.-K/s   in 0s

2021-01-08 06:24:56 (8.84 MB/s) - ‘service.yaml’ saved [178/178]
```

### 1. Deploying a Rollout
---------------------
First we deploy a Rollout resource and a Kubernetes Service targeting that Rollout. The example Rollout in this guide utilizes a canary update strategy which sends 20% of traffic to the canary, followed by a manual promotion, and finally gradual automated traffic increases for the remainder of the upgrade. This behavior is described in the following portion of the Rollout spec:
```
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {}
      - setWeight: 40
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
```      

Run the following command to deploy the initial Rollout and Service:

```
[user@kub-app001 kubernetes]$ kubectl apply -f rollout.yaml
rollout.argoproj.io/rollouts-demo created
[user@kub-app001 kubernetes]$ kubectl apply -f service.yaml
service/rollouts-demo created
```
Initial creations of any Rollout will immediately scale up the replicas to 100% (skipping any canary upgrade steps, analysis, etc...) since there was no upgrade that occurred.

The Argo Rollouts kubectl plugin allows you to visualize the Rollout, its related resources (ReplicaSets, Pods, AnalysisRuns), and presents live state changes as they occur. To watch the rollout as it deploys, run the get rollout --watch command from plugin:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch

Name:            rollouts-demo
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:blue (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE  INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy  22s
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  ✔ Healthy  22s  stable
      ├──□ rollouts-demo-7bf84f9696-87tzh  Pod         ✔ Running  22s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-jcm5z  Pod         ✔ Running  22s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-lzcqv  Pod         ✔ Running  22s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-nvxvm  Pod         ✔ Running  22s  ready:1/1
      └──□ rollouts-demo-7bf84f9696-q4ngv  Pod         ✔ Running  22s  ready:1/1


[user@kub-app001 kubernetes]$ kubectl get all -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                  NOMINATED NODE   READINESS GATES

pod/rollouts-demo-7bf84f9696-87tzh   1/1     Running   0          5m28s   10.39.245.175   kub-app003   <none>           <none>
pod/rollouts-demo-7bf84f9696-jcm5z   1/1     Running   0          5m28s   10.47.88.40     kub-app004   <none>           <none>
pod/rollouts-demo-7bf84f9696-lzcqv   1/1     Running   0          5m28s   10.38.46.60     kub-app005   <none>           <none>
pod/rollouts-demo-7bf84f9696-nvxvm   1/1     Running   0          5m28s   10.39.245.177   kub-app003   <none>           <none>
pod/rollouts-demo-7bf84f9696-q4ngv   1/1     Running   0          5m28s   10.47.88.43     kub-app004   <none>           <none>

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE     SELECTOR
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        104d    <none>
service/rollouts-demo   ClusterIP   10.99.64.62    <none>        80/TCP         5m22s   app=rollouts-demo

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx   3/3     3            3           25h   nginx        nginx:1.16.1   app=nginx

NAME                                       DESIRED   CURRENT   READY   AGE     CONTAINERS      IMAGES                        SELECTOR
replicaset.apps/nginx-599859995f           3         3         3       24h     nginx           nginx:1.16.1                  app=nginx,pod-template-hash=599859995f
replicaset.apps/nginx-845c85d7f8           0         0         0       25h     nginx           nginx                         app=nginx,pod-template-hash=845c85d7f8
replicaset.apps/rollouts-demo-7bf84f9696   5         5         5       5m28s   rollouts-demo   argoproj/rollouts-demo:blue   app=rollouts-demo,rollouts-pod-template-hash=7bf84f9696
```

### 2. Updating a Rollout
---------------------
Next it is time to perform an update. Just as with Deployments, any change to the Pod template field (spec.template) results in a new version (i.e. ReplicaSet) to be deployed. Updating a Rollout involves modifying the rollout spec, typically changing the container image field with a new version, and then running kubectl apply against the new manifest. As a convenience, the rollouts plugin provides a set image command, which performs these steps against the live rollout object in-place. Run the following command to update the rollouts-demo Rollout with the "yellow" version of the container:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
rollout "rollouts-demo" image updated
[user@kub-app001 kubernetes]$

[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch

Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/8		*
  SetWeight:     20
  ActualWeight:  20
Images:          argoproj/rollouts-demo:blue (stable)
                 argoproj/rollouts-demo:yellow (canary)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   8m33s
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy  2m4s   canary
│     └──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running  2m4s   ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  ✔ Healthy  8m33s  stable
      ├──□ rollouts-demo-7bf84f9696-87tzh  Pod         ✔ Running  8m33s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-jcm5z  Pod         ✔ Running  8m33s  ready:1/1
      ├──□ rollouts-demo-7bf84f9696-lzcqv  Pod         ✔ Running  8m33s  ready:1/1
      └──□ rollouts-demo-7bf84f9696-q4ngv  Pod         ✔ Running  8m33s  ready:1/1

[user@kub-app001 kubernetes]$ kubectl get all -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                  NOMINATED NODE   READINESS GATES
pod/rollouts-demo-789746c88d-xjxwh   1/1     Running   0          5m27s   10.39.245.178   kub-app003   <none>           <none>
pod/rollouts-demo-7bf84f9696-87tzh   1/1     Running   0          11m     10.39.245.175   kub-app003   <none>           <none>
pod/rollouts-demo-7bf84f9696-jcm5z   1/1     Running   0          11m     10.47.88.40     kub-app004   <none>           <none>
pod/rollouts-demo-7bf84f9696-lzcqv   1/1     Running   0          11m     10.38.46.60     kub-app005   <none>           <none>
pod/rollouts-demo-7bf84f9696-q4ngv   1/1     Running   0          11m     10.47.88.43     kub-app004   <none>           <none>

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        104d   <none>
service/rollouts-demo   ClusterIP   10.99.64.62    <none>        80/TCP         11m    app=rollouts-demo

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx   3/3     3            3           25h   nginx        nginx:1.16.1   app=nginx

NAME                                       DESIRED   CURRENT   READY   AGE     CONTAINERS      IMAGES                          SELECTOR
replicaset.apps/rollouts-demo-789746c88d   1         1         1       5m27s   rollouts-demo   argoproj/rollouts-demo:yellow   app=rollouts-demo,rollouts-pod-template-hash=789746c88d
replicaset.apps/rollouts-demo-7bf84f9696   4         4         4       11m     rollouts-demo   argoproj/rollouts-demo:blue     app=rollouts-demo,rollouts-pod-template-hash=7bf84f9696
```
When the demo rollout reaches the second step, we can see from the plugin that the Rollout is in a paused state, and now has 1 of 5 replicas running the new version of the pod template, and 4 of 5 replicas running the old version. This equates to the 20% canary weight as defined by the setWeight: 20 step

### 3. Promoting a Rollout
---------------------
The rollout is now in a paused state. When a Rollout reaches a pause step with no duration, it will remain in a paused state indefinitely until it is resumed/promoted. To manually promote a rollout to the next step, run the promote command of the plugin:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts promote rollouts-demo
rollout 'rollouts-demo' promoted
```

After promotion, Rollout will proceed to execute the remaining steps. The remaining rollout steps in our example are fully automated, so the Rollout will eventually complete steps until it has has fully transitioned to the new version. Watch the rollout again until it has completed all steps:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch

Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          7/8            *
  SetWeight:     80
  ActualWeight:  80
Images:          argoproj/rollouts-demo:blue (stable)		*
                 argoproj/rollouts-demo:yellow (canary)     *
Replicas:
  Desired:       5
  Current:       5
  Updated:       4
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS         AGE  INFO
⟳ rollouts-demo                            Rollout     ॥ Paused       17m
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy      10m  canary
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running      10m  ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running      47s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running      22s  ready:1/1
│     └──□ rollouts-demo-789746c88d-dn4ql  Pod         ✔ Running      10s  ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  ✔ Healthy      17m  stable
      ├──□ rollouts-demo-7bf84f9696-lzcqv  Pod         ◌ Terminating  17m  ready:1/1
      └──□ rollouts-demo-7bf84f9696-q4ngv  Pod         ✔ Running      17m  ready:1/1
	  
[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch
 
Name:            rollouts-demo
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8           *
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:yellow (stable)   *
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE   INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy     18m
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy     11m   stable
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running     11m   ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running     114s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running     89s   ready:1/1
│     ├──□ rollouts-demo-789746c88d-dn4ql  Pod         ✔ Running     77s   ready:1/1
│     └──□ rollouts-demo-789746c88d-5wmm5  Pod         ✔ Running     66s   ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown  18m
	  

[user@kub-app001 kubernetes]$ kubectl get all -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE                  NOMINATED NODE   READINESS GATES

pod/rollouts-demo-789746c88d-5wmm5   1/1     Running   0          111s    10.47.88.53     kub-app004   <none>           <none>
pod/rollouts-demo-789746c88d-dn4ql   1/1     Running   0          2m2s    10.39.245.191   kub-app003   <none>           <none>
pod/rollouts-demo-789746c88d-r6xh9   1/1     Running   0          2m14s   10.38.46.53     kub-app005   <none>           <none>
pod/rollouts-demo-789746c88d-xjxwh   1/1     Running   0          12m     10.39.245.178   kub-app003   <none>           <none>
pod/rollouts-demo-789746c88d-zg9dn   1/1     Running   0          2m39s   10.47.88.49     kub-app004   <none>           <none>

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        104d   <none>
service/rollouts-demo   ClusterIP   10.99.64.62    <none>        80/TCP         18m    app=rollouts-demo

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/nginx   3/3     3            3           25h   nginx        nginx:1.16.1   app=nginx

NAME                                       DESIRED   CURRENT   READY   AGE   CONTAINERS      IMAGES                          SELECTOR
replicaset.apps/rollouts-demo-789746c88d   5         5         5       12m   rollouts-demo   argoproj/rollouts-demo:yellow   app=rollouts-demo,rollouts-pod-template-hash=789746c88d
replicaset.apps/rollouts-demo-7bf84f9696   0         0         0       19m   rollouts-demo   argoproj/rollouts-demo:blue     app=rollouts-demo,rollouts-pod-template-hash=7bf84f9696

```

-	Note : The promote command also supports the ability to skip all remaining steps and analysis with the --full flag.

Once all steps complete successfully, the new ReplicaSet is marked as the "stable" ReplicaSet. Whenever a rollout is aborted during an update, either automatically via a failed canary analysis, or manually by a user, the Rollout will fall back to the "stable" version.


### 4. Aborting a Rollout
---------------------

Next we will learn how to manually abort a rollout during an update. First, deploy a new "red" version of the container using the set image command, and wait for the rollout to reach the paused step again:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red
rollout "rollouts-demo" image updated

[user@kub-app001 kubernetes]$ kubectl argo rollouts abort rollouts-demo

[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch


Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/8
  SetWeight:     20
  ActualWeight:  20
Images:          argoproj/rollouts-demo:red (canary)
                 argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS         AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused       26m
├──# revision:3
│  └──⧉ rollouts-demo-6f75f48b7            ReplicaSet  ✔ Healthy      23s    canary
│     └──□ rollouts-demo-6f75f48b7-qjsqs   Pod         ✔ Running      23s    ready:1/1
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy      19m    stable
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running      19m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running      9m47s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running      9m22s  ready:1/1
│     ├──□ rollouts-demo-789746c88d-dn4ql  Pod         ◌ Terminating  9m10s  ready:1/1
│     └──□ rollouts-demo-789746c88d-5wmm5  Pod         ✔ Running      8m59s  ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown   26m

```
   
This time, instead of promoting the rollout to the next step, we will abort the update, so that it falls back to the "stable" version. The plugin provides an abort command as a way to manually abort a rollout at any time during an update:

```
[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch


Name:            rollouts-demo
Namespace:       default
Status:          ✖ Degraded   *
Message:         RolloutAborted: Rollout is aborted   *
Strategy:        Canary
  Step:          0/8
  SetWeight:     0
  ActualWeight:  0
Images:          argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       0
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS         AGE   INFO
⟳ rollouts-demo                            Rollout     ✖ Degraded     27m
├──# revision:3
│  └──⧉ rollouts-demo-6f75f48b7            ReplicaSet  • ScaledDown   111s  canary
│     └──□ rollouts-demo-6f75f48b7-qjsqs   Pod         ◌ Terminating  111s  ready:1/1                   *
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy      21m   stable
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running      21m   ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running      11m   ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running      10m   ready:1/1
│     ├──□ rollouts-demo-789746c88d-5wmm5  Pod         ✔ Running      10m   ready:1/1
│     └──□ rollouts-demo-789746c88d-w568c  Pod         ✔ Running      13s   ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown   27m

[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch

Name:            rollouts-demo
Namespace:       default
Status:          ✖ Degraded   *
Message:         RolloutAborted: Rollout is aborted
Strategy:        Canary
  Step:          0/8
  SetWeight:     0
  ActualWeight:  0
Images:          argoproj/rollouts-demo:yellow (stable)  *
Replicas:
  Desired:       5
  Current:       5
  Updated:       0
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout     ✖ Degraded    28m
├──# revision:3
│  └──⧉ rollouts-demo-6f75f48b7            ReplicaSet  • ScaledDown  2m12s  canary
├──# revision:2
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy     21m    stable
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running     21m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running     11m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running     11m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-5wmm5  Pod         ✔ Running     10m    ready:1/1
│     └──□ rollouts-demo-789746c88d-w568c  Pod         ✔ Running     34s    ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown  28m

```
In order to make Rollout considered Healthy again and not Degraded, it is necessary to change the desired state back to the previous, stable version. This typically involves running kubectl apply against the previous Rollout spec. In our case, we can simply re-run the set image command using the previous, "yellow" image

```
[user@kub-app001 kubernetes]$  kubectl argo rollouts set image rollouts-demo  rollouts-demo=argoproj/rollouts-demo:yellow

[user@kub-app001 kubernetes]$ kubectl argo rollouts get rollout rollouts-demo --watch

Name:            rollouts-demo
Namespace:       default
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy     31m
├──# revision:4
│  └──⧉ rollouts-demo-789746c88d           ReplicaSet  ✔ Healthy     24m    stable
│     ├──□ rollouts-demo-789746c88d-xjxwh  Pod         ✔ Running     24m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-zg9dn  Pod         ✔ Running     14m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-r6xh9  Pod         ✔ Running     14m    ready:1/1
│     ├──□ rollouts-demo-789746c88d-5wmm5  Pod         ✔ Running     13m    ready:1/1
│     └──□ rollouts-demo-789746c88d-w568c  Pod         ✔ Running     3m32s  ready:1/1
├──# revision:3
│  └──⧉ rollouts-demo-6f75f48b7            ReplicaSet  • ScaledDown  5m10s
└──# revision:1
   └──⧉ rollouts-demo-7bf84f9696           ReplicaSet  • ScaledDown  31m


[user@kub-app001 kubernetes]$ kubectl argo rollouts list rollouts
NAME           STRATEGY   STATUS        STEP  SET-WEIGHT  READY  DESIRED  UP-TO-DATE  AVAILABLE
rollouts-demo  Canary     Healthy       8/8   0           5/5    5        5           5

```
When a Rollout has not yet reached its desired state (e.g. it was aborted, or in the middle of an update), and the stable manifest were re-applied, the Rollout detects this as a rollback and not a update, and will fast-track the deployment of the stable ReplicaSet by skipping analysis, and the steps.


### Summary

In this guide, we have learned basic capabilities of Argo Rollouts, including:
* Deploying a rollout
* Performing a canary update
* Manual promotion
* Manual abortion

The Rollout in this basic example did not utilize a ingress controller or service mesh provider to route traffic. Instead, it used normal Kubernetes Service networking (i.e. kube-proxy) to achieve an approximate canary weight, based on the closest ratio of new to old replica counts. As a result, this Rollout had a limitation in that it could only achieve a minimum canary weight of 20%, by scaling 1 of 5 pods to run the new version. In order to achieve much finer grained canaries, an ingress controller or service mesh is necessary.

#### Source Link : https://argoproj.github.io/argo-rollouts/installation/
