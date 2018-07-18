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

Delete your NodePort service:
```
$ kubectl delete service nginx-deployment
service "nginx-deployment" deleted

$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   2h
```

Note: In a vanilla K8s deployment your NodePort IP could be an exposed <Public_IP>:<Node_Port> combination that is accessible by the public. But that is not the case in this scenario because I am running Kubernetes within a private subnet.

## Deploy and configure network load balancer

On cloud providers which support external load balancers, setting the type field to LoadBalancer will provision a load balancer for your Service. The actual creation of the load balancer happens asynchronously, and information about the provisioned balancer will be published in the Service’s .status.loadBalancer field.

Below is an example LoadBalancer service definition:
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155
```
Traffic from the external load balancer will be directed at the backend Pods, though exactly how that works depends on the cloud provider. Some cloud providers allow the loadBalancerIP to be specified. In those cases, the load-balancer will be created with the user-specified loadBalancerIP. If the loadBalancerIP field is not specified, an ephemeral IP will be assigned to the loadBalancer. If the loadBalancerIP is specified, but the cloud provider does not support the feature, the field will be ignored.

## Know how to use Ingress rules
Reference from kubernetes.io:
- [Concepts - Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

Typically, services and pods have IPs only routable by the cluster network. All traffic that ends up at an edge router is either dropped or forwarded elsewhere. An Ingress is a collection of rules that allow inbound connections to reach the cluster services. It can be configured to give services externally-reachable URLs, load balance traffic, terminate SSL, offer name based virtual hosting, and more. Users request ingress by POSTing the Ingress resource to the API server. An Ingress controller is responsible for fulfilling the Ingress, usually with a loadbalancer, though it may also configure your edge router or additional frontends to help handle the traffic in an HA manner.

### Ingress Controllers
In order for the Ingress resource to work, the cluster must have an Ingress controller running. Kubernetes does not run an ingress controller by default. This is unlike other types of controllers, which typically run as part of the kube-controller-manager binary, and which are typically started automatically as part of cluster creation. Choose the ingress controller implementation that best fits your cluster, or implement a new ingress controller.
- Traefik
- NGINX
- HAProxy
- Envoy
- Istio

Here is an example Traefik Ingress Controller that is configured to be compatible with Kubernetes on DC/OS that we will name `traefik.yaml` for our example:
```
$ cat traefik.yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        args:
        - --web
        - --kubernetes
# NOTE: What follows are necessary additions to
# https://docs.traefik.io/user-guide/kubernetes
# Please check below for a detailed explanation
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: k8s-app
                operator: In
                values:
                - traefik-ingress-lb
            topologyKey: "kubernetes.io/hostname"
      nodeSelector:
        kubernetes.dcos.io/node-type: public
      tolerations:
      - key: "node-type.kubernetes.dcos.io/public"
        operator: "Exists"
        effect: "NoSchedule"
```

A little detail on what this particular Ingress Controller is set up to work with DC/OS is found here: [Mesosphere Documentation - Using a Traefik Ingress Controller](https://docs.mesosphere.com/services/kubernetes/1.2.0-1.10.5/ingress/#example-using-the-traefik-ingress-controller) but some of the information is also below:
- Bind each pod’s :80 port to the host’s :80 port. This is the easiest way to expose the ingress controller on the public node as DC/OS already opens up the :80 port on public agents. However, when using hostPort you are responsible for making sure that no other application (either in-cluster or even outside DC/OS) is using the :80 port on every public agent. Kubernetes won’t be able to schedule pods on a particular agent if the port is already being used.
- Make use of pod anti-affinity to ensure that pods are spread among available public agents in order to ensure high-availability. This is somewhat redundant when using hostPort as described above, but it may be useful when exposing the controller using a different method (see below).
- Make use of the nodeSelector constraint to force pods to be scheduled on public nodes only.
- Make use of the node-type.kubernetes.dcos.io/public node taint so that the pods can actually run on the public nodes.

Go ahead and deploy your Ingress Controller:
```
$ kubectl create -f traefik.yaml
clusterrole.rbac.authorization.k8s.io "traefik-ingress-controller" created
clusterrolebinding.rbac.authorization.k8s.io "traefik-ingress-controller" created
serviceaccount "traefik-ingress-controller" created
deployment.extensions "traefik-ingress-controller" created

$ kubectl get deployments --namespace=kube-system
NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-dns                     2         2         2            2           3h
kubernetes-dashboard         1         1         1            1           3h
metrics-server               1         1         1            1           3h
traefik-ingress-controller   1         1         1            1           1m

$ kubectl get pods --namespace=kube-system -o wide
NAME                                         READY     STATUS    RESTARTS   AGE       IP        NODE
kube-dns-797d4bd8dd-gqr2z                    3/3       Running   0          3h        9.0.7.4   kube-node-1-kubelet.kubernetes.mesos
kube-dns-797d4bd8dd-v58tt                    3/3       Running   0          3h        9.0.7.3   kube-node-1-kubelet.kubernetes.mesos
kubernetes-dashboard-5c469b58b8-49r5b        1/1       Running   0          3h        9.0.7.2   kube-node-1-kubelet.kubernetes.mesos
metrics-server-77c969f8c-hvf9s               1/1       Running   0          3h        9.0.7.5   kube-node-1-kubelet.kubernetes.mesos
traefik-ingress-controller-bfb78685c-9hbc5   1/1       Running   0          2m        9.0.8.2   kube-node-public-0-kubelet.kubernetes.mesos
```

As you can see above, the Traefik ingress controller was deployed into the `kube-system` namespace and is consuming `kube-node-public-0-kubelet.kubernetes.mesos` the Kubernetes public agent

Now that your Ingress Controller is deployed, we can expose the `nginx-deployment` application to the public internet by following the steps:
- Deploy application (in our case the `nginx-deployment` app is already RUNNING)
- Create a Service to expose your application
- Create an Ingress Service to allow your Service to be accessible to the public

Following our `nginx-deployment` definition, below is an ingress service definition for this nginx-deployment which we will name `nginx-ingress.yaml` for our example:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
  name: nginx-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: nginx-deployment
          servicePort: 80
```

Now we can deploy and expose our application:
```
$ kubectl create -f nginx-service.yaml
service "nginx-deployment" created

$ kubectl get svc -o wide
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes         ClusterIP   10.100.0.1       <none>        443/TCP   3h        <none>
nginx-deployment   ClusterIP   10.100.100.158   <none>        80/TCP    10s       app=nginx

$ kubectl create -f nginx-ingress.yaml
ingress.extensions "nginx-ingress" created

$ kubectl get ingress
NAME            HOSTS     ADDRESS   PORTS     AGE
nginx-ingress   *                   80        12s

$ kubectl describe ingress nginx-ingress
Name:             nginx-ingress
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *
        /   nginx-deployment:80 (<none>)
Annotations:
  kubernetes.io/ingress.class:  traefik
Events:                         <none>
```

Now that you have a running app with a service exposed by the Traefik Ingress Controller, navigate to your Kubernetes Public Node IP. You can reach it through your browser, or by curling the Public IP from anywhere:
```
$ curl 54.190.7.200
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

