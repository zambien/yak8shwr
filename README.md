# yak8shwr
Yet Another Kubernetes the Hard Way Repo

## Description

Kubernetes the Hard Way  using Vagrant and VirtualBox for **MAXIMUM LEARNING!11!!!!!!one!!?1!**

This is an as-code implementation of Drew Viles' [medium article](https://medium.com/\@DrewViles/kubernetes-the-hard-way-on-bare-metal-vms-fdb32bc4fed0") which is heavily based upon Kelsey Hightower's legendary repo, [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).  The format of this repo mimics and in some cases directly copies Kelsey Hightower's [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

After going through `Kubernetes the Hard Way` I felt a good appreciation for the aspects of configuring and running a kubernetes cluster.  But I wanted to leave it up so that I could play more and I'm cheap so I thought a local environment would be great to continue the learning while also learning how one might set this up in a local datacenter instead of a cloud provider.  This repo will be laid out in a similar fashion to Kelsey's with instructions on how to set this up step-by-step.

## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

## Cluster Details

yak8shwr guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

* [kubernetes](https://github.com/kubernetes/kubernetes) 1.15.3
* [containerd](https://github.com/containerd/containerd) 1.2.9
* [coredns](https://github.com/coredns/coredns) v1.6.3
* [cni](https://github.com/containernetworking/cni) v0.7.1
* [etcd](https://github.com/coreos/etcd) v3.4.0

## Labs

This tutorial assumes you have a computer with at least 4 cores, 100GB disk space, and 8GB RAM and that you are working on a system with a good native shell.  I used linux so if this doesn't work on a Mac I'm sorry.

Note that all of the 100GB disk will not be used, however theoretically it could.

* [Prerequisites](docs/01-create-VMs.md)
* [Install Tools](docs/02-install-tools.md)
* [Certs and Stuff](docs/03-certs-and-stuff.md)
* [Kubeconfig and Encryption Keys](docs/04-kubeconfigs.md)
* [Setup Controllers](docs/05-setup-controllers.md)
* [Configure the load balancer](docs/06-config-lb.md)
* [Setup Workers](docs/07-setup-workers.md)
