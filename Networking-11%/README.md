This document is a self-made Lab guide based off of the CKA Curriculum v1.8.0

# Networking - 11%

## Understand the network configuration on the cluster nodes
Reference from kubernetes.io:
- [Concepts - Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

Four fundamental networking problems Kubernetes aims to solve:
- Highly-coupled container-to-container communications: this is solved by pods and localhost communications.
- Pod-to-Pod communications: this is the primary focus of this document.
- Pod-to-Service communications: this is covered by services.
- External-to-Service communications: this is covered by services.

Kubernetes assumes that pods can communicate with other pods, regardless of which host they land on. Every pod gets its own IP address so you do not need to explicitly create links between pods and you almost never need to deal with mapping container ports to host ports.

Compared to the Docker model which uses host-private networking, the Kubernetes networking model does not use the virtual bridge `docker0` which assigns a subnet from one of the private address blocks on the bridge. This allows for containers running in Kubernetes to be able to communicate across nodes without the need of port-mapping

## Understand pod networking concepts
Kubernetes imposes the following fundamental requirements on any networking implementation (barring any intentional network segmentation policies):
- all containers can communicate with all other containers without NAT
- all nodes can communicate with all containers (and vice-versa) without NAT
- the IP that a container sees itself as is the same IP that others see it as

Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. Containers inside a Pod can communicate with one another using localhost. When containers in a Pod communicate with entities outside the Pod, they must coordinate how they use the shared network resources (such as ports).

## Understand Service Networking
Reference from kubernetes.io:
- [Concepts - Services](https://kubernetes.io/docs/concepts/services-networking/service/)

A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service. The set of Pods targeted by a Service is (usually) determined by a Label Selector (see below for why you might want a Service without a selector).

For Kubernetes-native applications, Kubernetes offers a simple Endpoints API that is updated whenever the set of Pods in a Service changes. For non-native applications, Kubernetes offers a virtual-IP-based bridge to Services which redirects to the backend Pods.

### ClusterIP Service

`ClusterIP` is the default ServiceType in Kubernetes. `ClusterIP` exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster.

Lets run through a basic service example using our familiar `nginx-deployment.yaml` below:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Deploy the `nginx-deployment.yaml`:
```
$ kubectl create -f nginx-deployment.yaml
deployment.apps "nginx-deployment" created

$ kubectl get deployments -o wide
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                    SELECTOR
nginx-deployment   3         3         3            3           1m        nginx        nginx                     app=nginx

$ kubectl get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP         NODE
nginx-deployment-666865b5dd-4k755   1/1       Running   0          59s       9.0.9.16   kube-node-0-kubelet.kubernetes.mesos
nginx-deployment-666865b5dd-n68wp   1/1       Running   0          59s       9.0.9.17   kube-node-0-kubelet.kubernetes.mesos
nginx-deployment-666865b5dd-nvwtl   1/1       Running   0          59s       9.0.7.10   kube-node-1-kubelet.kubernetes.mesos
```

At this point you should be able to SSH into any node in your cluster and curl any of the IPs above and we should get the NGINX landing page:
```
root@ip-10-0-4-243 2d9d4a8d-d992-4d72-908f-1e8f34353ab3 # curl 9.0.9.16
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

Having an accessible pod that we can talk to directly is great, but what happens if a node dies? The pods on the node die with it and the Deployment will create new pods with new IPs. This is the fundamental problem that a Service solves

A Service is a Kubernetes abstraction which defines a logical set of Pods running somewhere in the cluster. A Service is assigned a unique IP address that is tied to the lifespan of the Service itself and will not change as long as the Service is alive. Pods can then be configured to talk to the service to let the service expose the Pod rather thane exposing the Pod directly. This helps us avoid the situation above.

### Creating a Service

Lets start by creating a service we will define as `nginx-service.yaml`:
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

The simple service definition above creates a Service called `nginx-service` and the selector will bind to any deployment with the label: app=nginx on port 80

The same simple Service can also be created by the command below:
```
$ kubectl expose deployment/nginx-deployment
service "nginx-deployment" exposed

$ kubectl get svc -o wide
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP   1h        <none>
nginx-deployment   ClusterIP   10.100.242.223   <none>        80/TCP    13s       app=nginx
```

Now we can describe the Service that was just created:
```
$ kubectl describe service nginx-deployment
Name:              nginx-deployment
Namespace:         default
Labels:            app=nginx
Annotations:       <none>
Selector:          app=nginx
Type:              ClusterIP
IP:                10.100.242.223
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         9.0.7.10:80,9.0.9.16:80,9.0.9.17:80
Session Affinity:  None
Events:            <none>
```

Now that the Service is created, you should be able to curl the NGINX service on `<CLUSTER_IP>:<PORT>` from any node in your cluster. In order to do so, lets deploy a curl docker container into our cluster.

The command below will deploy a busybox curl container in an interactive shell that we can use to curl the ClusterIP:
```
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty --rm
If you don't see a command prompt, try pressing enter.
[ root@curl-775f9567b5-wtdkr:/ ]$

[ root@curl-775f9567b5-wtdkr:/ ]$ curl 10.100.242.223
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


As you can see, I can still continue to curl the Pod directly as well:
[ root@curl-775f9567b5-wtdkr:/ ]$ curl 9.0.7.10:80
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

You can also use the `nslookup` utility tool to get the DNS hostname for the NGINX service as well:
```
$ kubectl get svc -o wide
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP   1h        <none>
nginx-deployment   ClusterIP   10.100.200.213   <none>        80/TCP    42s       app=nginx

Back in busybox:

[ root@curl-775f9567b5-wtdkr:/ ]$ nslookup nginx-deployment
Server:    198.51.100.4
Address 1: 198.51.100.4

Name:      nginx-deployment
Address 1: 10.100.200.213 nginx-deployment.default.svc.cluster.local

[ root@curl-775f9567b5-wtdkr:/ ]$ curl nginx-deployment.default.svc.cluster.local
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

For now, we can remove this Service:
```
$ kubectl delete svc nginx-deployment
service "nginx-deployment" deleted

$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1h
```

### NodePort
Reference from kubernetes.io:
- [Publishing Services - Service Types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)

NodePort: Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting <NodeIP>:<NodePort>.

If you set the type field to NodePort, the Kubernetes master will allocate a port from a range specified by --service-node-port-range flag (default: 30000-32767), and each Node will proxy that port (the same port number on every Node) into your Service. That port will be reported in your Service’s .spec.ports[*].nodePort field.

More information can be accessed at the link above if you want to specify a particular IP:Port combination

Lets continue with our `nginx-deployment` to expose this application through `NodePort`:
```
$ kubectl expose deployment/nginx-deployment --type="NodePort" --port 80
service "nginx-deployment" exposed

$ kubectl get svc -o wide
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE       SELECTOR
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP        2h        <none>
nginx-deployment   NodePort    10.100.136.148   <none>        80:30582/TCP   17s       app=nginx

$ kubectl describe svc nginx-deployment
Name:                     nginx-deployment
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP:                       10.100.136.148
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30582/TCP
Endpoints:                9.0.7.10:80,9.0.9.16:80,9.0.9.17:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Back in my BusyBox I can curl the NodePort IP:
```
[ root@curl-775f9567b5-wtdkr:/ ]$ curl 10.100.214.112
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

Note: In a vanilla K8s deployment your NodePort IP could be an exposed <Public_IP>:<Node_Port> combination that is accessible by the public. But that is not the case in this scenario because I am running Kubernetes within a private subnet.



