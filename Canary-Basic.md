#### This guide will demonstrate various concepts and features of Argo Rollouts by going through deployment, upgrade, promotion, and abortion of a Rollout.

## Deployments application with Canary-Basic strategy using argocd rollouts 

```
[user@kub-app001 ~]$ mkdir -p kubernetes/argocd-rollout-demo
[user@kub-app001 ~]$ cd kubernetes/argocd-rollout-demo/
```
Download rollout.yaml & service.yaml files 
```
[user@kub-app001 kubernetes]$ wget  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
--2021-01-08 06:24:38--  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
Resolving lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)... 10.164.246.138
Proxy request sent, awaiting response... 200 OK
Length: 752 [text/plain]
Saving to: ‘rollout.yaml’

100%[===============================================================================================================================================>] 752         --.-K/s   in 0s

2021-01-08 06:24:38 (36.0 MB/s) - ‘rollout.yaml’ saved [752/752]
```
```
[user@kub-app001 kubernetes]$ wget https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
--2021-01-08 06:24:56--  https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
Resolving lab-api-proxy.lab.mycompany.com (lab-api-proxy.lab.mycompany.com)... 10.164.246.138
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
