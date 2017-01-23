---
image: "/img/about-bg.jpg"
description: ""
draft: false
date: "2016-12-08T21:54:36Z"
title: "'Kubernetes The Hard Way' - Local with Libvirt"
tags: 
  - "Kubernetes"
  - "SaltStack"
  - "devops"
categories: 
  - "Tech"
---

Many of the introductory articles on-line for Kubernetes focus on
getting the simplest environment possible up and running, with a view
to getting the user quickly on to the topic of how to use Kubernetes
to run their software. This is ideal for people wanting to evaluate
Kubernetes features or validate their application for a
container environment. 

It's like there is an inherent assumption that when it comes to that
time down the line when you are going to use it in anger, someone else
in your organisation will have delivered that environment for you to
run your application on.

Now that approach doesn't work for me for a couple of reasons. If my
team is responsible for the design of the system delivery then there
is a reasonable chance we will also be involved in the deployment
design as well. Devops doesn't work if the ops/environments
responsibility is siloed into one person in the team. Secondly if my
goal is to understand cost and benefit of a technology then I need to
understand the cost/complexity of delivering the environment in the
way it will ultimately be required in production. That at least
involves understanding how HA and security will work as well as some
exploration of how the platform will interact with the infrastructure
environment  e.g. networking and storage. A little extra time up
front grappling with these "harder" things will pay off in the long
run. 

### Enter "Kubernetes - The Hard Way"
For this reason I was looking for a way of learning about Kubernetes
that took a more detailed step by step approach to delivering the
environment. Enter "The Hard Way".
[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
is a tutorial by [Kelsey Hightower](https://twitter.com/kelseyhightower)
which is :

> "optimized for learning, which means taking the long route to help
people understand each task required to bootstrap a Kubernetes
cluster." 

This is exactly what I was looking for so I dived in. However I did
have some idiosyncrasies in how I worked it. 
Kelsey's tutorial is based on infrastructure provisioned in either AWS
or GCE, and he provides instructions for both. In my case I wanted to
use libvirt on an old server I had lying around because I wanted to
understand what would be involved in on-premises Kubernetes
deployments. I also had a secondary goal of learning how to use
[SaltStack](https://github.com/saltstack/salt) and wanted to have a
real world use case for my first Salt automation (but more of that in
another post!).

### The Infrastructure

The tutorial delivers a HA Kubernetes Cluster. Kubernetes delivers HA
by storing all state and data in a separate store, etcd. There are a
set of mater processes which run on Kubernetes master nodes and a set
of worker processes that run on Kubernetes worker nodes.

In a HA configuration there are three master nodes. In order to save
on hardware, (my server is 2 x quad core w 32GB RAM),  the etcd cluster
runs on the same nodes as the master processes. For a production
deployment it is recommended to run the etcd cluster on separate nodes.
The tutorial also proposes three worker nodes which which will allow
for applications to be deployed in a HA fashion on top of Kubernetes.

Kubernetes requires that all pods deployed in a cluster can see each
other without the use of NAT. This requires some means of presenting a
flat network for the cluster. This is often done using container
networking fabrics such as [Flannel](https://github.com/coreos/flannel).
However where infrastructure is provided by an IAAS cloud then there
may already be a flat network in place at the IAAS tier, this is the
case for GCE and AWS in the tutorial, so the container network fabric
is not required.

In my libvirt case I created a separate libvirt network for my
Kubernetes hosts so they were all sharing the same flat segment at the
infrastructure layer.

Virtual machines are created based on the [Centos 7 cloud image](http://cloud.centos.org/centos/7/images/).
and have a fixed IP address on the flat network, injected by copying
an ifcfg file via cloud-init user data. I also added the SaltStack
repo and salt-minion rpm install via the user data (but this is for
future Salt learning). I hacked together a python script to generate
the user data and create the virtual machines based on a libvirt XML
template. This is a bit of a 'chicken and egg' situation - [Salt Virt]
(https://docs.saltstack.com/en/latest/topics/virt/index.html) would be
a better solution but I am not there yet!

![infrastructure](/img/k8sflat.svg)

So now that infrastructure is set up the next five stages of the
tutorial continue as is. The next part where there is a change is in
'Managing the Container Network Routes'.

### What's different in the Container Network Routes
As the tutorial uses the [kubenet](http://kubernetes.io/docs/admin/network-plugins/#kubenet)
plug-in for pod networking we do need to take care of routing between
pods on the worker nodes. In the configuration of the control plane
in the tutorial, the _kube-controller-manager_ is configured to allocate
CIDRs from within the cluster CIDR to each node. The _kubelet_ on each
node will allocate addresses to each pod it creates from within the
nodes CIDR.
Now the kubenet plug-in is a rudimentary network plug-in and does not
provide any advanced features related to inter-node routing or
network policy. It relies on routing configuration on the nodes to
deliver traffic between nodes.
So as explained in the tutorial you will need to find the CIDR that
the _kube-controller-manager_ automatically assigned to each node
and configure routing via the host IP address of that node.
The tutorial provides a kubectl query that will give you that mapping:

```
kubectl get nodes \
--output=jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address} {.spec.podCIDR} {"\n"}{end}'
```

Once you have this you can issue appropriate _ip route_ commands on
each node to route to the CIDRs on the other nodes.
e.g. on node-1

```
ip route add <cidr of node2> via <ip addr of node 2>
ip route add <cidr of node3> via <ip addr of node 3>
```

and so on for each of the other nodes.

![routes](/img/k8sflat_routes.svg)

Now you are good to go and play with your cluster.

However if you will run into a problem with this approach if you want
to move to an automated solution - (did I mention I will be doing this
with SaltStack in a future post :-) ).
Each time you add a node it will automatically be allocated a CIDR and
you will not know the routing required until you query the assigned
CIDR as above.


### Enabling Manual (Predictable) Assignment of Node CIDR

A little bit of reading and research shows that there is an
alternative means of assigning pod CIDR to nodes. Automatic allocation
is enabled/disabled via a command line argument to
_kube-controller-manager_ and as such can be set in the systemd unit
file. You can then assign a specific CIDR to each node via a command
line argument for _kubelet_.

First of all modify the _kube-controller-manager_ systemd unit file in
the 'Bootstrapping an H/A Kubernetes Control Plane' section to change
_--allocate-node-cidrs_ to false.

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --leader-elect=true \
  --master=http://HOST_INTERNAL_IP:8080 \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --allocate-node-cidrs=false \
  --service-cluster-ip-range=10.32.0.0/24 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Now we need to assign the CIDR for each node manually, and obviously
avoid clashes. We then configure this value by adding a pod-cidr
argument in the _kubelet_ systemd unit.
This is straightforward when it is being driven by an automation tool.

```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/opt/kubernetes/bin/kubelet \
  --allow-privileged=true \
  --api-servers=https://192.168.170.10:6443,https://192.168.170.11:6443,https://192.168.170.12:6443 \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --network-plugin=kubenet \
  --container-runtime=docker \
  --docker=unix:///var/run/docker.sock \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --pod-cidr=10.200.0.0/24 \
  --reconcile-cidr \
  --serialize-image-pulls=false \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Now your automation tool can co-ordinate assignments in kubelet and
routing on each node.

### So What Have I learned?

The goal of [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
is to learn rather than to generate a production ready deployment.

The situation described here will not arise when CNI plug-in is being
used as the CNI plug-in will allow Kubernetes to provide an update to
the networking fabric as node CIDRs are assigned.

Making it a bit harder still  with a different infrastructure
environment than specified in the tutorial meant getting a good
understanding of pod networking. Progressing to an automation of the
deployment exposed different options for node CIDR assignment.

I'd recommend familiarising yourself with [Kubernetes network plug-ins](http://kubernetes.io/docs/admin/network-plugins/)
and the different options they provide. I'd also recommend looking
into the [CNI specification](https://github.com/containernetworking/cni).

Oh, and do follow Kelsey Hightower on Twitter and Github - he's a mine
of information and innovation!
