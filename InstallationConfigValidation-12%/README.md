This Repo is based off of the [CKA Curriculum v1.9.0](https://github.com/cncf/curriculum/blob/master/certified_kubernetes_administrator_exam_v1.9.0.pdf)

# Installation, Configuration and Validation - 12%
Reference from kubernetes.io:

## Design a Kubernetes Cluster
Reference from kubernetes.io:
- [Designing and Preparing](https://kubernetes.io/docs/setup/scratch/#designing-and-preparing)

### Cloud Provider:
Kubernetes has a concept of a Cloud Provider which is a module that provides an interface for managing TCP Load Balancers, Nodes (Instances) and Networking Routes. This is optional, but could be helpful extra tooling in your Kubernetes deployment

### Nodes:
- You can use virtual or physical machines.
- While you can build a cluster with 1 machine, in order to run all the examples and tests you need at least 4 nodes.
- Many Getting-started-guides make a distinction between the master node and regular nodes. This is not strictly necessary.
- Nodes will need to run some version of Linux with the x86_64 architecture. It may be possible to run on other OSes and Architectures, but this guide does not try to assist with that.
- Apiserver and etcd together are fine on a machine with 1 core and 1GB RAM for clusters with 10s of nodes. Larger or more active clusters may benefit from more cores.
- Other nodes can have any reasonable amount of memory and any number of cores. They need not have identical configurations.

### Network:
Kubernetes allocates an IP address to each pod. A process in one pod should be able to communicate with another pod using the IP of the second pod. This is more extensively discussed in the Networking section of this repo. At a high level, this connectivity can be accomplished in two ways:
- Using an overlay network
	- An overlay network obscures the underlying network architecture from the pod network through traffic encapsulation (for example vxlan).
	- Encapsulation reduces performance, though exactly how much depends on your solution.
-Without an overlay network
	- Configure the underlying network fabric (switches, routers, etc.) to be aware of pod IP addresses.
	- This does not require the encapsulation provided by an overlay, and so can achieve better performance.

You will need to select an address range for the Pod IPs, below are a couple approaches:
- GCE: each project has its own 10.0.0.0/8. Carve off a /16 for each Kubernetes cluster from that space, which leaves room for several clusters. Each node gets a further subdivision of this space.
- AWS: use one VPC for whole organization, carve off a chunk for each cluster, or use different VPC for different clusters.

Allocate one CIDR subnet for each node’s PodIPs, or a single large CIDR from which smaller CIDRs are automatically allocated to each node. You should plan this to the number of pods you expect to run on each node. Changing the range (i.e. from /16 to /24) is possible, but note that you may risk disrupting the services and pods that already use it. An example of sizing is below:
- You need max-pods-per-node * max-number-of-nodes IPs in total. A /24 per node supports 254 pods per machine and is a common choice. If IPs are scarce, a /26 (62 pods per machine) or even a /27 (30 pods) may be sufficient.
- For example, use 10.10.0.0/16 as the range for the cluster, with up to 256 nodes using 10.10.0.0/24 through 10.10.255.0/24, respectively.

Lastly for the Master Node:
- Needs a static IP
- Call this MASTER_IP.
- Open any firewalls to allow access to the apiserver ports 80 and/or 443.
- Enable ipv4 forwarding sysctl, net.ipv4.ip_forward = 1

### Software Binaries - see [kubernetes.io](kubernetes.io) for the latest binaries and instructions
- etcd
- A container runner, one of:
	- docker
	- rkt
- Kubernetes
- kubelet
- kube-proxy
- kube-apiserver
- kube-controller-manager
- kube-scheduler

### Security - There are two main options:
- Access the apiserver using HTTP.
	- Use a firewall for security.
	- This is easier to setup.
- Access the apiserver using HTTPS
	- Use https with certs, and credentials for user.
	- This is the recommended approach.
	- Configuring certs can be tricky.

#### HTTPS
For the HTTPS approach, you will need to prepare certs and credentials
- The master needs a cert to act as an HTTPS server.
- The kubelets optionally need certs to identify themselves as clients of the master, and when serving its own API over HTTPS.

You will have to generate the following files:
- CA_CERT - put in on node where apiserver runs, for example in `/srv/kubernetes/ca.crt`
- MASTER_CERT - signed by CA_CERT and put in on node where apiserver runs, for example in `/srv/kubernetes/server.crt`
- MASTER_KEY - put in on node where apiserver runs, for example in `/srv/kubernetes/server.key`
- KUBELET_CERT - optional
- KUBELET_KEY - optional

#### Credentials
The admin user as well as any other users will need a token or a password to identify them. tokens and passwords need to be stored in a file for the apiserver to read. This kubernetes.io documentation uses `/var/lib/kube-apiserver/known_tokens.csv`

For distributing credentials to clients, the convention in Kubernetes is to put the credentials into a kubeconfig file which is typically at `$HOME/.kube/config`. You need to add certs, keys, and the master IP to the kubeconfig file.

## Install Kubernetes masters and nodes, including the use of TLS bootstrapping
Reference from kubernetes.io:
- [Creating a Custom Cluster from Scratch](https://kubernetes.io/docs/setup/scratch/)

Other References:
- [Github: Kelsey Hightower's - Kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

Here we will install Kubernetes the hard way from scratch following Kelsey Hightower's guide.

### Prerequisites
There are many guides out there on how to create instances in the cloud so we will skip that for this guide. For this example I will be using the base Ubuntu 16.04 image on AWS with these specs:
- t2.large (2 CPU, 8GB RAM, 20GB Storage)
	- 1x Masters
	- 3x Kubernetes Worker Agents
- Using a /20 CIDR block
- Security Group Rules - SSH (Port 22), Allow All Traffic (tcp, udp, ICMP)
* See Security section of this Repo for more details on locking down your cluster

#### Client Tools
On your Local Machine, install the `cfssl`, `cfssljson`, and `kubectl` client tools. These will be used to provision a PKI Insfrastructure and generate TLS certificates

#### CFSSL Installation:
OSX:
```
$ curl -o cfssl https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
$ curl -o cfssljson https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
$ chmod +x cfssl cfssljson
$ sudo mv cfssl cfssljson /usr/local/bin/

OR if Homebrew is installed:

$ brew install cfssl
```

Linux:
```
$ wget -q --show-progress --https-only --timestamping \
    https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
    https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
 
$ chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

$ sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

$ sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

Verify that `cfssl` version 1.2.0 or higher is installed:
```
$ cfssl version
Version: 1.3.2
Revision: dev
Runtime: go1.10.2
```

#### Install kubectl:

To install kubectl, the Kubernetes Command Line tool, follow the instructions below:
- [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Configure secure cluster communications
Reference from kubernetes.io:
- [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

### CA and TLS Certificates
Following Kelsey Hightower's guide on Provisioning a CA and Generating TLS Certificates will allow us to generate TLS certificates for the etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy
- [Provisioning a CA and Generating TLS Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md)

#### Certificate Authority
Provision a Certificate Authority that can be used to generate additional TLS certificates. Generate the CA configuration file, certificate, and private key:
```
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

#### Client and Server Certificates
Generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user.

Generate the Admin Client Certificate and private key:
```
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

Generate the Kubelet Client Certificate. Generate a certificate and private key for each Kubernetes worker node:

Worker node 0 - Be sure to fill in the `<WORKER_X_PUBLIC_IP>` and `<WORKER_X_PRIVATE_IP>` values before running:
```
cat > worker-0-csr.json <<EOF
{
  "CN": "system:node:worker-0",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=<WORKER_0_PUBLIC_IP>

INTERNAL_IP=<WORKER_0_PRIVATE_IP>

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-0,${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  worker-0-csr.json | cfssljson -bare worker-0
```

Worker node 1 - - Be sure to fill in the `<WORKER_X_PUBLIC_IP>` and `<WORKER_X_PRIVATE_IP>` values before running:
```
cat > worker-1-csr.json <<EOF
{
  "CN": "system:node:worker-1",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=<WORKER_1_PUBLIC_IP>

INTERNAL_IP=<WORKER_1_PRIVATE_IP>

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-1,${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  worker-1-csr.json | cfssljson -bare worker-1
```

Worker Node 2 - - Be sure to fill in the `<WORKER_X_PUBLIC_IP>` and `<WORKER_X_PRIVATE_IP>` values before running:
```
cat > worker-2-csr.json <<EOF
{
  "CN": "system:node:worker-2",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=<WORKER_2_PUBLIC_IP>

INTERNAL_IP=<WORKER_2_PRIVATE_IP>

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=worker-2,${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  worker-2-csr.json | cfssljson -bare worker-2
```

Generate the `kube-controller-manager` client certificate and private key:
```
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

Generate the `kube-proxy` client certificate and private key:
```
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

Generate the `kube-scheduler` client certificate and private key:
```
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

Generate the Kubernetes API Server certificate and private key - remember to replace the `<MASTER_PUBLIC_IP>`:
```
{

KUBERNETES_PUBLIC_ADDRESS=<MASTER_PUBLIC_IP>

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

Generate the service-account certificate and private key:
```
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

#### Distribute the Client and Server Certificates:

Copy the appropriate certificates and private keys to the 3 Kubernetes Worker instances:
```
$ scp -i <SSH_Key_PATH> ca.pem  worker-0-key.pem worker-0.pem ubuntu@<WORKER_0_PUBLIC_IP>:~
$ scp -i <SSH_Key_PATH> ca.pem  worker-1-key.pem worker-1.pem ubuntu@<WORKER_1_PUBLIC_IP>:~
$ scp -i <SSH_Key_PATH> ca.pem  worker-2-key.pem worker-2.pem ubuntu@<WORKER_2_PUBLIC_IP>:~
```
Copy the appropriate certificates and private keys to each Kubernetes master node (Controller instance):
```
scp -i <SSH_Key_PATH> ca.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem ca-key.pem ubuntu@<MASTER_0_PUBLIC_IP>:~
```

### Generate Kubernetes Files for Authentication
Following Kelsey Hightower's guide on Provisioning a CA and Generating TLS Certificates will allow us to generate Kubernetes Configuration Files, also known as kubeconfigs, which enable K8s clients to locate and authenticate to the K8s API servers
- [Generating Kubernetes Configuration Files for Authentication](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/05-kubernetes-configuration-files.md)

In this section you will generate kubeconfig files for the controller manager, kubelet, kube-proxy, and scheduler clients and the admin user.

#### The kubelet Kubernetes Configuration File:
Since I am only using 1 Kubernetes Master Node in this installation, remember to replace the `<MASTER_0_PUBLIC_IP>` parameter on the Worker nodes:

Worker 0 - remember to replace the `<MASTER_0_PUBLIC_IP>`:
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://<MASTER_0_PUBLIC_IP>:6443 \
    --kubeconfig=worker-0.kubeconfig

    kubectl config set-credentials system:node:worker-0 \
    --client-certificate=worker-0.pem \
    --client-key=worker-0-key.pem \
    --embed-certs=true \
    --kubeconfig=worker-0.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:worker-0 \
    --kubeconfig=worker-0.kubeconfig

kubectl config use-context default --kubeconfig=worker-0.kubeconfig
```

Worker 1 - remember to replace the `<MASTER_0_PUBLIC_IP>`:
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://<MASTER_0_PUBLIC_IP>:6443 \
    --kubeconfig=worker-1.kubeconfig

    kubectl config set-credentials system:node:worker-1 \
    --client-certificate=worker-1.pem \
    --client-key=worker-1-key.pem \
    --embed-certs=true \
    --kubeconfig=worker-1.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:worker-1 \
    --kubeconfig=worker-1.kubeconfig

kubectl config use-context default --kubeconfig=worker-1.kubeconfig
```

Worker 2 - remember to replace the `<MASTER_0_PUBLIC_IP>`:
```
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://<MASTER_0_PUBLIC_IP>:6443 \
    --kubeconfig=worker-2.kubeconfig

    kubectl config set-credentials system:node:worker-2 \
    --client-certificate=worker-2.pem \
    --client-key=worker-2-key.pem \
    --embed-certs=true \
    --kubeconfig=worker-2.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:worker-2 \
    --kubeconfig=worker-2.kubeconfig

kubectl config use-context default --kubeconfig=worker-2.kubeconfig
```

#### The kube-proxy Kubernetes Configuration File.

Swap out the `${KUBERNETES_PUBLIC_ADDRESS}` before executing the command:
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

#### The kube-controller-manager Kubernetes configuration file
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

#### The kube-scheduler Kubernetes Configuration File
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

#### The admin Kubernetes Configuration File
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

Results:
```
$ ls | grep kubeconfig
admin.kubeconfig
kube-controller-manager.kubeconfig
kube-proxy.kubeconfig
kube-scheduler.kubeconfig
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### Distribute the Kubernetes Configuration Files
Workers:
```
$ scp -i <SSH_KEY_PATH> worker-0.kubeconfig kube-proxy.kubeconfig ubuntu@<WORKER_0_IP>:~
$ scp -i <SSH_KEY_PATH> worker-1.kubeconfig kube-proxy.kubeconfig ubuntu@<WORKER_1_IP>:~
$ scp -i <SSH_KEY_PATH> worker-2.kubeconfig kube-proxy.kubeconfig ubuntu@<WORKER_2_IP>:~
```

Master:
```
$ scp -i <SSH_KEY_PATH> admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@<MASTER_PUBLIC_IP>:~
```

### Generating the Data Encryption Config and Key
Generate an encryption key:
```
$ ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Create the `encryption-config.yaml` file:
```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the encryption config file to the Master:
```
$ scp -i <SSH_KEY_PATH> encryption-config.yaml ubuntu@<MASTER_PUBLIC_IP>:~
```

### etcd

Reference: 
- [Bootstrapping the etcd cluster](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/07-bootstrapping-etcd.md)

The guide above bootstraps a three node etcd cluster, but for our exercise we will only be doing one since we only have one Kubernetes Master node.

SSH into the cluster to download etcd:
```
$ ssh -i <SSH_KEY_PATH> ubuntu@<MASTER_PUBLIC_IP>

ubuntu@ip-172-31-20-96:~$
$ wget -q --show-progress --https-only --timestamping \
>   "https://github.com/coreos/etcd/releases/download/v3.3.5/etcd-v3.3.5-linux-amd64.tar.gz"
etcd-v3.3.5-linux-amd64.tar.gz                  100%[======================================================================================================>]  10.75M   924KB/s    in 32s

ubuntu@ip-172-31-20-96:~$ ls
admin.kubeconfig  ca.pem                  etcd-v3.3.5-linux-amd64.tar.gz      kubernetes-key.pem  kube-scheduler.kubeconfig  service-account.pem
ca-key.pem        encryption-config.yaml  kube-controller-manager.kubeconfig  kubernetes.pem      service-account-key.pem
```

Extract and install the `etcd` server and `etcdctl` command line utility:
```
{
  tar -xvf etcd-v3.3.5-linux-amd64.tar.gz
  sudo mv etcd-v3.3.5-linux-amd64/etcd* /usr/local/bin/
}
```

Configure the etcd server:
```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

Set the Internal IP of etcd:
```
INTERNAL_IP=<MASTER_PRIVATE_IP>
```

Set the etcd name to match the hostname of the current compute instance:
```
ETCD_NAME=$(hostname -s)
```

Create the etcd.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Now start the etcd server:
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

#### Verification

List the etcd cluster members:
```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

Output should resemble below:
```
1418fc9607ab9822, started, ip-172-31-20-96, https://172.31.20.96:2380, https://172.31.20.96:2379
```

### Provision the Kubernetes Control Plane
Still in the Kubernetes Master Node, create the Kubernetes configuration directory:
```
$ sudo mkdir -p /etc/kubernetes/config
```

Download and install the official Kubernetes release binaries:
```
$ wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.10.2/bin/linux/amd64/kubectl"

$ {
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
  }
```

Configure the Kubernetes API Server:
```
$ {
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

Create the `kube-apiserver.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://${INTERNAL_IP}:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Configure the Kubernetes Controller Manager
Move the kube-controller-manager kubeconfig into place:
```
$ sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Configure the Kubernetes Scheduler
Move the kube-scheduler kubeconfig into place:
```
$ sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the kube-scheduler.yaml configuration file:
```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the kube-scheduler.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### Start the Controller Services
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

#### Validation
```
$ sudo systemctl status kube-apiserver
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-07-19 21:39:06 UTC; 21s ago
<...>

$ sudo systemctl status kube-controller-manager
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-07-19 21:39:06 UTC; 39s ago
<...>

$ sudo systemctl status kube-scheduler
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-07-19 21:39:06 UTC; 46s ago
<...>
```

kubectl verification:
```
kubectl get componentstatuses --kubeconfig admin.kubeconfig


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
