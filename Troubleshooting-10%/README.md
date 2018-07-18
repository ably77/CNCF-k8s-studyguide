This document is a self-made Lab guide based off of the CKA Curriculum v1.8.0

# Troubleshooting - 10%
Reference from kubernetes.io:
- [Determine the Reason for Pod Failure](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/)
- [Application Introspection and Debugging](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application-introspection/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)

## Troubleshoot application failure
Reference from kubernetes.io:
- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)

There are many application failure scenarios, but first we can focus on a few tools that are useful to troubleshooting and then we can move into some examples

First we will deploy the nginx example below. For our exercise we will use the filename `nginx-deployment.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

Deploy the `nginx-deployment` application:
```
$ kubectl create -f nginx-deployment.yaml
deployment.apps "nginx-deployment" created
```

### Useful troubleshooting tools

The first step in debugging a Pod is taking a look at it. Check the current state of the Pod and recent events using `kubectl get pods` and  `kubectl describe pods <pod_name>`:
```
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-66996bc984-bngc5   1/1       Running   0          4m
nginx-deployment-66996bc984-qkgnk   1/1       Running   0          4m

$ kubectl describe pods nginx-deployment
Name:           nginx-deployment-66996bc984-bngc5
Namespace:      default
Node:           kube-node-1-kubelet.kubernetes.mesos/10.0.6.184
Start Time:     Tue, 17 Jul 2018 12:26:39 -0700
Labels:         app=nginx
                pod-template-hash=2255267540
Annotations:    <none>
Status:         Running
IP:             9.0.3.5
Controlled By:  ReplicaSet/nginx-deployment-66996bc984
Containers:
  nginx:
    Container ID:   docker://5b1210651ad63baa7c8c2e71ea77bc1a2d130c2498ae2a26df2d963ef1b4d85e
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:4a5573037f358b6cdfa2f3e8a9c33a5cf11bcd1675ca72ca76fbe5bd77d0d682
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 17 Jul 2018 12:26:44 -0700
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        500m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-242lk (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
Volumes:
  default-token-242lk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-242lk
    Optional:    false
QoS Class:       Guaranteed
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason                 Age   From                                           Message
  ----    ------                 ----  ----                                           -------
  Normal  Scheduled              3m    default-scheduler                              Successfully assigned nginx-deployment-66996bc984-bngc5 to kube-node-1-kubelet.kubernetes.mesos
  Normal  SuccessfulMountVolume  3m    kubelet, kube-node-1-kubelet.kubernetes.mesos  MountVolume.SetUp succeeded for volume "default-token-242lk"
  Normal  Pulling                3m    kubelet, kube-node-1-kubelet.kubernetes.mesos  pulling image "nginx"
  Normal  Pulled                 2m    kubelet, kube-node-1-kubelet.kubernetes.mesos  Successfully pulled image "nginx"
  Normal  Created                2m    kubelet, kube-node-1-kubelet.kubernetes.mesos  Created container
  Normal  Started                2m    kubelet, kube-node-1-kubelet.kubernetes.mesos  Started container
```

As you can see from the above outputs, the NGINX app was succesfully deployed and is in a `READY` state

### My pod stays PENDING
If a Pod is stuck in Pending it means that it can not be scheduled onto a node. Generally this is because there are insufficient resources of one type or another that prevent scheduling. The output of `kubectl describe` as shown above should outline messages from the scheduler about why it can not schedule the Pod

Common reasons:
- Not enough resources - You may have exhausted the supply of CPU or Memory in your cluster, in this case you need to delete Pods, adjust resource requests, or add new nodes to your cluster. See Compute Resources document for more information.
- Networking - if using hostPort: When you bind a Pod to a hostPort there are a limited number of places that pod can be scheduled. In most cases, hostPort is unnecessary, try using a Service object to expose your Pod. If you do require hostPort then you can only schedule as many Pods as there are nodes in your Kubernetes cluster.

Lets induce a PENDING error:
```
$ kubectl edit deployment/nginx-deployment

Modify the CPU requirements to be > than the resources available to your kubelet:
        resources:
          limits:
            cpu: 2500m
            memory: 128Mi

deployment.extensions "nginx-deployment" edited
```

Because we edited the deployment to have CPU > the amount of CPU resource available to the kubelet, we should see that the deployment is now in a PENDING state:
```
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-54d9549b74-s6md5   0/1       Pending   0          8s
nginx-deployment-66996bc984-bngc5   1/1       Running   0          17m
nginx-deployment-66996bc984-qkgnk   1/1       Running   0          17m

$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         3         1            2           43m
```

Checking the PENDING pod we can see in the Events that we have Insufficient CPU:
```
$ kubectl describe pod nginx-deployment-54d9549b74-s6md5
<...>
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  3m (x26 over 10m)  default-scheduler  0/2 nodes are available: 2 Insufficient cpu.
```

Editing the state back:
```
$ kubectl edit deployment/nginx-deployment
 
Modify the CPU requirements to be < than the resources available to your kubelet:
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
deployment.extensions "nginx-deployment" edited
```

We can see that the Deployment is now successfully scheduled:
```
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-66996bc984-bngc5   1/1       Running   0          41m
nginx-deployment-66996bc984-qkgnk   1/1       Running   0          41m

$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            2           41m
```

### My pod stays waiting
If a Pod is stuck in the Waiting state, then it has been scheduled to a worker node, but it can’t run on that machine. Again, the information from kubectl describe ... should be informative. The most common cause of Waiting pods is a failure to pull the image. There are three things to check:
- Make sure that you have the name of the image correct.
- Have you pushed the image to the repository?
- Run a manual docker pull <image> on your machine to see if the image can be pulled.

### My pod is having an error pulling image

Lets induce an image pull error by fat-fingering the image name:
```
$ kubectl edit deployments/nginx-deployment
<...>
containers:
      - image: nginxx
        imagePullPolicy: Always
deployment.extensions "nginx-deployment" edited

$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         3         1            2           2m

$ kubectl get pods
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-587564b686-msm9n   0/1       ImagePullBackOff   0          1m
nginx-deployment-66996bc984-55xj7   1/1       Running            0          2m
nginx-deployment-66996bc984-rjpdj   1/1       Running            0          2m
```

Exploring the pod with the `ImagePullBackOff` error:
```
$ kubectl describe pod nginx-deployment-587564b686-msm9n
<...>
Conditions:
  Type           Status
  Initialized    True
  Ready          False
  PodScheduled   True
Volumes:
  default-token-242lk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-242lk
    Optional:    false
QoS Class:       Guaranteed
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                 Age               From                                           Message
  ----     ------                 ----              ----                                           -------
  Normal   Scheduled              2m                default-scheduler                              Successfully assigned nginx-deployment-587564b686-msm9n to kube-node-1-kubelet.kubernetes.mesos
  Normal   SuccessfulMountVolume  2m                kubelet, kube-node-1-kubelet.kubernetes.mesos  MountVolume.SetUp succeeded for volume "default-token-242lk"
  Normal   Pulling                26s (x4 over 2m)  kubelet, kube-node-1-kubelet.kubernetes.mesos  pulling image "nginxx"
  Warning  Failed                 25s (x4 over 2m)  kubelet, kube-node-1-kubelet.kubernetes.mesos  Failed to pull image "nginxx": rpc error: code = Unknown desc = Error response from daemon: pull access denied for nginxx, repository does not exist or may require 'docker login'
  Warning  Failed                 25s (x4 over 2m)  kubelet, kube-node-1-kubelet.kubernetes.mesos  Error: ErrImagePull
  Normal   BackOff                14s (x6 over 1m)  kubelet, kube-node-1-kubelet.kubernetes.mesos  Back-off pulling image "nginxx"
  Warning  Failed                 14s (x6 over 1m)  kubelet, kube-node-1-kubelet.kubernetes.mesos  Error: ImagePullBackOff
```

Fixing this image parameter will fix the deployment:
```
$ kubectl edit deployments/nginx-deployment
<...>
containers:
      - image: nginx
        imagePullPolicy: Always
deployment.extensions "nginx-deployment" edited

$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            2           4m

$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-66996bc984-55xj7   1/1       Running   0          4m
nginx-deployment-66996bc984-rjpdj   1/1       Running   0          4m
```

Delete the deployment:
```
$ kubectl delete deployment nginx-deployment
deployment.extensions "nginx-deployment" deleted
```

### My Pod is crashing or otherwise unhealthy:

First take a look at the logs of the current container
```
kubectl logs <pod_name> <container_name>
```

Or if your container has previously crashed, you can access the previous container's crash log:
```
kubectl logs --previous <pod_name> <container_name>
```

Or you can exec into the running container:
```
$ kubectl exec -it nginx-deployment-66996bc984-cl457 bash
root@nginx-deployment-66996bc984-cl457:/#
```

## Troubleshoot control plane/worker node failure
Reference from kubernetes.io:
- [Troubleshoot Clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)

The first thing to debug in your cluster is if your nodes are all registered correctly. Verify that all nodes you expect to see present are in the `Ready` state:
```
$ kubectl get nodes
NAME                                   STATUS    ROLES     AGE       VERSION
kube-node-0-kubelet.kubernetes.mesos   Ready     <none>    3h        v1.10.3
kube-node-1-kubelet.kubernetes.mesos   Ready     <none>    3h        v1.10.3
```

### Looking at Logs
For now, digging deeper into the cluster requires logging into the relevant machines. Here are the locations of the relevant log files. (note that on systemd-based systems, you may need to use journalctl instead)

Master:
- /var/log/kube-apiserver.log - API Server, responsible for serving the API
- /var/log/kube-scheduler.log - Scheduler, responsible for making scheduling decisions
- /var/log/kube-controller-manager.log - Controller that manages replication controllers

Worker Nodes:
- /var/log/kubelet.log - Kubelet, responsible for running containers on the node
- /var/log/kube-proxy.log - Kube Proxy, responsible for service load balancing

Common cluster failure root causes:
- VM(s) shutdown
- Network partition within cluster, or between cluster and users
- Crashes in Kubernetes software
- Data loss or unavailability of persistent storage (e.g. GCE PD or AWS EBS volume)
- Operator error, e.g. misconfigured Kubernetes software or application software

## Troubleshoot Networking:
Reference from kubernetes.io:
- [Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

Confirm that the service exists:
```
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   6h
```

From the output above, you can see that there is no service that exists. This is correct, because we never deployed a service. Lets go ahead and do that.

Here is a Service definition for our `nginx-deployment`. For this example we will name it `nginx-service.yaml`:
```
kind: Service
apiVersion: v1
metadata:
  name: nginx-deployment
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
```

Deploy the service:
```
$ kubectl create -f nginx-service.yaml
service "nginx-deployment" created

$ kubectl get services
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes         ClusterIP   10.100.0.1      <none>        443/TCP   6h
nginx-deployment   ClusterIP   10.100.177.47   <none>        80/TCP    12s
```

### Does the Service work by DNS?

We can check if the Service works by DNS by launching a Pod in the same namespace and using the `nslookup` utility:
```
### Run an interactive Curl Container in Kubernetes
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
If you don't see a command prompt, try pressing enter.
[ root@curl-775f9567b5-nf6tf:/ ]$

[ root@curl-775f9567b5-nf6tf:/ ]$ nslookup nginx-deployment
Server:    198.51.100.4
Address 1: 198.51.100.4

Name:      nginx-deployment
Address 1: 10.100.177.47 nginx-deployment.default.svc.cluster.local
```

If this fails, perhaps your Pod and Service are in different Namespaces, try a namespace-qualified name. Your `nslookup` would look something like below:
```
[ root@curl-775f9567b5-nf6tf:/ ]$ nslookup nginx-deployment
Server:    198.51.100.4
Address 1: 198.51.100.4

nslookup: can't resolve 'nginx-deployment'
```

To show this we will deploy our `nginx-deployment.yaml`, `nginx-service.yaml`, and the `curl` busybox image into a namespace called `foo`:
```
$ kubectl create namespace foo
namespace "foo" created

$ kubectl create -f nginx-deployment.yaml --namespace=foo
deployment.apps "nginx-deployment" created

$ kubectl get pods --namespace=foo
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-66996bc984-7wt85   1/1       Running   0          7s
nginx-deployment-66996bc984-bdwbm   1/1       Running   0          7s

$ kubectl create -f nginx-service.yaml --namespace=foo
service "nginx-deployment" created

$ kubectl get svc --namespace=foo
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-deployment   ClusterIP   10.100.253.135   <none>        80/TCP    11s
```

Now from your BusyBox container:
```
[ root@curl-775f9567b5-zgnvm:/ ]$ nslookup nginx-deployment.foo
Server:    198.51.100.4
Address 1: 198.51.100.4

Name:      nginx-deployment.foo
Address 1: 10.100.253.135 nginx-deployment.foo.svc.cluster.local
```

As you can see above, because we deployed the pod and service in the same namespace we were able to access the DNS hostname

To remove:
```
$ kubectl delete deployments nginx-deployment --namespace=foo
deployment.extensions "nginx-deployment" deleted

$ kubectl get deployments --namespace=foo
No resources found.

$ kubectl delete service nginx-deployment --namespace=foo
service "nginx-deployment" deleted

$ kubectl get service --namespace=foo
No resources found.

$ kubectl delete namespace foo
namespace "foo" deleted

$ kubectl get namespaces
NAME              STATUS    AGE
dcos-kubernetes   Active    6h
default           Active    6h
heptio-ark        Active    6h
kube-public       Active    6h
kube-system       Active    6h
```

### Does any Service exist in the DNS 

If none of the above is working, we might have to explore outside of the Pod. Maybe the DNS service itself isnt working. The Kubernetes master service should always work, so we can check there

From the BusyBox:
```
[ root@curl-775f9567b5-nf6tf:/ ]$ nslookup kubernetes.default
Server:    198.51.100.4
Address 1: 198.51.100.4

Name:      kubernetes.default
Address 1: 10.100.0.1 kubernetes.default.svc.cluster.local
```

If this fails, you should look ingo debugging DNS through kube-proxy before debugging your pod service.

### Does the Service work by IP?

Lets induce an error and work through how to fix it. What we're going to do is edit the Service deployment to a different port. In our example we will use port 8080:
```
$ kubectl edit svc nginx-deployment
<...>
ports:
  - port: 8080
    protocol: TCP
    targetPort: 80
:wq
service "nginx-deployment" edited
```

You can see now that the Service Port changed to 8080. Now try to curl the Service IP
```
$ kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP    6h
nginx-deployment   ClusterIP   10.100.183.153   <none>        8080/TCP   34s

[ root@curl-775f9567b5-nf6tf:/ ]$ curl 10.100.193.118
<TIMEOUT>
```

If your Service is working correctly, you should be able to `curl` the Cluster-IP and get correct responses. In the situation above, this is not the case because the service and pod ports are not matching

Editing this back will return us to a working state:
```
$ kubectl edit svc nginx-deployment
<...>
ports:
  - port: 80
    protocol: TCP
    targetPort: 80
:wq
service "nginx-deployment" edited

$ kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP   6h
nginx-deployment   ClusterIP   10.100.183.153   <none>        80/TCP    5m

### In Busybox:
[ root@curl-775f9567b5-nf6tf:/ ]$ curl 10.100.183.153
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Does the Service have any Endpoints?
If you got this far, we assume that you have confirmed that your Service exists and is resolved by DNS. Now let’s check that the Pods you ran are actually being selected by the Service. We can now check if the Service has any Endpoints:
```
$ kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP    6h
nginx-deployment   ClusterIP   10.100.183.153   <none>        8080/TCP   34s

$ kubectl get endpoints nginx-deployment
NAME               ENDPOINTS                 AGE
nginx-deployment   9.0.7.17:80,9.0.9.31:80   7m
```

In this example above, because my service is working correclty I am able to get an output from `kubectl get endpoints`, however here is an example where this doesnt work if the service doesnt exist:
```
$ kubectl delete service nginx-deployment
service "nginx-deployment" deleted

$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   6h

$ kubectl get endpoints nginx-deployment
Error from server (NotFound): endpoints "nginx-deployment" not found
```




