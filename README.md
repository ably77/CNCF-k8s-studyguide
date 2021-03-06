This Repo is based off of the [CKA Curriculum v1.9.0](https://github.com/cncf/curriculum/blob/master/certified_kubernetes_administrator_exam_v1.9.0.pdf)

Much of the information gathered in this repo is pulled from [Kubernetes.io](kubernetes.io), but is just consolidated in a way that is easier to digest. It was done this way because the test only allows us to use the [Kubernetes.io](kubernetes.io) webpage and [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) anyways. Some examples I purposefully changed as much as possible to use a consistent NGINX example across all of the hands-on labs.

The general breakdown is below and is what this repo aims to cover:

## Completed
- **Application Lifecycle Management - 8%**
  - Understand Deployments and how to perform rolling updates and rollbacks
  - Know various ways to configure applications
  - Know how to scale applications
  - Understand the primitives necessary to create a self-healing application
- **Core Concepts - 19%**
  - Understand the Kubernetes API primitives
  - Understand the Kubernetes cluster architecture
  - Understand Services and other network primitives
- **Scheduling - 5%**
  - Use label selectors to schedule Pods
  - Understand the role of DaemonSets
  - Understand how resource limits can affect Pod scheduling
  - Understand how to run multiple schedulers and how to configure Pods to use them
  - Manually schedule a pod without a scheduler
  - Display scheduler events
  - Know how to configure the Kubernetes scheduler
  - Optional: Using Taints
- **Cluster Maintenance - 11%**
  - Understand Kubernetes cluster upgrade process
  - Facilitate operating system upgrades
  - Implement backup and restore methodologies
- **Logging / Monitoring - 5%**
  - Understand how to monitor all cluster components
  - Understand how to monitor applications
  - Manage cluster component logs
  - Manage application logs
- **Storage - 7%**
  - Understand persistent volumes and know how to create them
  - Understand access modes for volumes
  - Understand persistent volume claims primitive
  - Understand Kubernetes storage objects
  - Know how to configure applications with persistent storage
- **Troubleshooting - 10%**
  - Troubleshoot application failure
  - Troubleshoot control plane failure
  - Troubleshoot worker node failure
  - Troubleshoot networking
- **Networking - 11%**
  - Understand the networking configuration on the cluster nodes
  - Understand Pod networking concepts
  - Understand service networking
  - Deploy and configure network load balancer
  - Know how to use Ingress rules
  - Know how to configure and use the cluster DNS
  - Understand CNI

## WIP
- **Security - 12%** 
  - Know how to configure authentication and authorization
  - Understand Kubernetes security primitives
  - Know to configure network policies
  - Create and manage TLS certificates for cluster components
  - Work with images securely
  - Define security contexts
  - Secure persistent key value store
  - Work with role-based access control
  
- **Installation, Configuration & Validation - 12%**
  - Design a Kubernetes Cluster
  - Install Kubernetes masters and nodes, including the use of TLS bootstrapping
  - Configure secure cluster communications
  - Configure a HA Kubernetes cluster
  - Know where to get the Kubernetes release binaries
  - Provision underlying infrastructure to deploy a Kubernetes cluster
  - Choose a network solution
  - Choose your Kubernetes infrastructure configuration
  - Run end-to-end tests on your cluster
  - Analyse end-to-end test results
  - Run Node end-to-end tests  
