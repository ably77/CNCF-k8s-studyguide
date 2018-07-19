This Repo is based off of the [CKA Curriculum v1.9.0](https://github.com/cncf/curriculum/blob/master/certified_kubernetes_administrator_exam_v1.9.0.pdf)

# Installation, Configuration and Validation - 12%
Reference from kubernetes.io:

## Design a Kubernetes Cluster
Reference from kubernetes.io:
- [Designing and Preparing](https://kubernetes.io/docs/setup/scratch/#designing-and-preparing)

## Install Kubernetes masters and nodes, including the use of TLS bootstrapping
Reference from kubernetes.io:
- [Creating a Custom Cluster from Scratch](https://kubernetes.io/docs/setup/scratch/)

## Configure secure cluster communications
Reference from kubernetes.io:
- [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)


## Configure a HA Kubernetes Cluster
Reference from kubernetes.io:
- [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/independent/high-availability/)

## Know where to get the Kubernetes release binaries
Reference from kubernetes.io:
- [Kubernetes Releases](https://kubernetes.io/docs/imported/release/notes/)

You can access the Binaries at the link above, or straight from the kubernetes.io homepage

## Provision underlying infrastructure to deploy a Kubernetes cluster
Reference:
- [Github - Kelsey Hightower's Kubernetes the Hard Way - Cloud Infrastructure Provisioning on GCP](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/f9486b081f8f54dd63a891463f0b0e783d084307/docs/01-infrastructure-gcp.md)

## Choose a Network solution
Reference from kubernetes.io:
- [Network](https://kubernetes.io/docs/setup/scratch/#network)

## Choose your Kubernetes infrastructure configuration
Reference from kubernetes.io:

## Run end-to-end tests on your cluster
Reference from kubernetes.io:
- [Validation - End-to-End Testing](https://kubernetes.io/docs/getting-started-guides/ubuntu/validation/)

References: 
- [ADVANCED: Github - End-to-End Testing in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/e2e-tests.md)

Here are a few simple commands to test your cluster:
```
$ kubectl cluster-info
Kubernetes master is running at http://localhost:9000
KubeDNS is running at http://localhost:9000/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
kube-node-0-kubelet.kubernetes.mesos          Ready     <none>    7h        v1.10.3
kube-node-1-kubelet.kubernetes.mesos          Ready     <none>    7h        v1.10.3
kube-node-public-0-kubelet.kubernetes.mesos   Ready     <none>    7h        v1.10.3

$ kubectl get componentstatuses
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: getsockopt: connection refused
controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: getsockopt: connection refused
etcd-2               Healthy     {"health":"true"}
etcd-0               Healthy     {"health":"true"}
etcd-1               Healthy     {"health":"true"}

$ kubectl get pods -o wide --show-labels --all-namespaces
NAMESPACE     NAME                                         READY     STATUS    RESTARTS   AGE       IP        NODE                                          LABELS
kube-system   kube-dns-797d4bd8dd-gqr2z                    3/3       Running   0          7h        9.0.7.4   kube-node-1-kubelet.kubernetes.mesos          k8s-app=kube-dns,pod-template-hash=3538068488
kube-system   kube-dns-797d4bd8dd-v58tt                    3/3       Running   0          7h        9.0.7.3   kube-node-1-kubelet.kubernetes.mesos          k8s-app=kube-dns,pod-template-hash=3538068488
kube-system   kubernetes-dashboard-5c469b58b8-49r5b        1/1       Running   0          7h        9.0.7.2   kube-node-1-kubelet.kubernetes.mesos          k8s-app=kubernetes-dashboard,pod-template-hash=1702561464
kube-system   metrics-server-77c969f8c-hvf9s               1/1       Running   0          7h        9.0.7.5   kube-node-1-kubelet.kubernetes.mesos          k8s-app=metrics-server,pod-template-hash=337525947
kube-system   traefik-ingress-controller-bfb78685c-9hbc5   1/1       Running   0          4h        9.0.8.2   kube-node-public-0-kubelet.kubernetes.mesos   k8s-app=traefik-ingress-lb,name=traefik-ingress-lb,pod-template-hash=696342417

$ kubectl get svc  -o wide --show-labels --all-namespaces
NAMESPACE     NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE       SELECTOR                       LABELS
default       kubernetes             ClusterIP   10.100.0.1       <none>        443/TCP         7h        <none>                         component=apiserver,provider=kubernetes
kube-system   kube-dns               ClusterIP   10.100.0.2       <none>        53/UDP,53/TCP   7h        k8s-app=kube-dns               addonmanager.kubernetes.io/mode=Reconcile,k8s-app=kube-dns,kubernetes.io/cluster-service=true,kubernetes.io/name=KubeDNS
kube-system   kubernetes-dashboard   ClusterIP   10.100.133.106   <none>        80/TCP          7h        k8s-app=kubernetes-dashboard   k8s-app=kubernetes-dashboard
kube-system   metrics-server         ClusterIP   10.100.211.252   <none>        443/TCP         7h        k8s-app=metrics-server         kubernetes.io/name=Metrics-server
```

### Running the actual e2e tests

## Analyze end-to-end test results
References from kubernetes.io:
- [Evaluating end-to-end results](https://kubernetes.io/docs/getting-started-guides/ubuntu/validation/#evaluating-end-to-end-results)

It is not enough to just simply run the test. Result output is stored in two places. The raw output of the e2e run is available in the juju show-action-output command, as well as a flat file on disk on the kubernetes-e2e unit that executed the test.
Note: The results will only be available once the action has completed the test run. End-to-end testing can be quite time consuming, often taking more than 1 hour, depending on configuration.

### Accessing the results in a flat file

Show the output inline:
```
juju run-action kubernetes-e2e/0 test
```

Output:
```
Action queued with id: 4ceed33a-d96d-465a-8f31-20d63442e51b
```

Show the results in your terminal:
```
juju show-action-output 4ceed33a-d96d-465a-8f31-20d63442e51b
```

## Run node end-to-end tests
References:
- [Github - Node End-to-End tests](https://github.com/kubernetes/community/blob/master/contributors/devel/e2e-node-tests.md)
