# Study Notes 1: A bird's-eye view of a Kubernetes cluster


I've been meaning to do this for a while now. I've studied quite a bit of Kubernetes in the past year and have a ton of handwritten notes that I want to throw away, so I'm transfering all of those notes into my blog. This is also a great review opportunity (and also has the added advantage that I will stop needing to comb through endless sheets of paper when I want to remember something). This is the first of such posts and it covers some basics around the architecture of a Kubernetes cluster. 

## Overview

Kubernetes is basically a system for automating deployment, management, and scaling of containerized applications. In other words, it essentially allows you to automate a lot of systems management tasks in a relatively pain free way. Usually, if you want to use Kubernetes for something, you need access to a Kubernetes clusters, which consists on a set of nodes, called worker nodes, which can be physical or virtual, that will host you applications as containers. To manage these worker nodes, we need a centralized entity, which is a separate node known as the master node. This node contains an ample set of components that together form what is known as the Kubernetes control plane.

{{< figure src="/images/kubernetes_overview/master_and_worker_nodes.png" title="Overview of master and worker nodes in Kubernetes" >}}

Let us cover some of the components that exist in the master node. Note that the purpose of this post is not to provide an extensive list of these components, but rather a good enough list to understand the inner workings of a Kubernetes cluster. With that in mind, we have the follow cogs in a master node:
* `kube-apiserver`
* `etcd` server
* `kube-scheduler`
* Controller manager

Likewise, each worker node also has its own components that are require for its normal functioning:
* `kubelet`
* `kube-proxy`
* Container Runtime Interface (CRI)

Let us go through these one by one and then well look at how the Kubernetes cluster handles the simple task of creating a pod.

## `kube-apiserver`

The `kube-apiserver` is possibly the most important component of Kubernetes, since it is responsible for orchestrating all operations within the cluster. Additionally, as the name suggests, it also exposes the Kubernetes API, which is used by external users to perform operations on the clusters, as well as by the various controllers to monitor the state of the cluster and make necessary changes as required. It is also the primary way that worker nodes communicate with the master node. If you have ever used the `kubectl` command, congratulations, you have performed a request to the `kube-apiserver`.

## `etcd` server

Information about the nodes and the various Kubernetes objects must be stored somewhere, and that's where `etcd` comes in. `etcd` is a distributed reliable key-value store that is simple, secure and fast (all the good qualities we want from a piece of software). Any and all changes performed in a Kubernetes cluster are recorded in the `etcd` server and are only considered complete precisely when they are stored. 

## `kube-scheduler`

Fortunately, Kubernetes employs a lot of meaningful naming, so we can correctly infer that the `kube-scheduler` is the control plane component that is in charge of scheduling. More specifically, it is in charge of scheduling pods on the available nodes taking into account existing constraints, such as the pod resource requirements, worker nodes capacity, etc. One very important detail is that it *only* takes care of scheduling, it doesn't actually place any pod on any node. Since this component has to take into account several different constraints, it's process to place a pod on a node generally has two phases:
* Firstly, it filters out any nodes that can immediately be deemed unfit for the pod, such as nodes with insufficient resources;
* Then, it ranks the remaining nodes via a scoring function. There are several scoring functions that we can pick from, and we can even make our own custom scoring method.

## Controller manager

The controller manager is the component that takes care of managing the different controllers on our Kubernetes cluster. A controller is simply a process that continuously monitors the state of various objects withing the system and ensures that these are in the desired functioning state. Generally speaking, there is a controller for each type of Kubernetes objects, but we'll look at two examples: the replication controller and the node controller.

There is a type of Kubernetes object called ReplicaSet, which essentially is just a way of telling Kubernetes that we want our pod to have a given number of replicas. What actually ensures that we have the desired number of replicas is, you've guessed it, the replication controller. Super simple.

A perhaps more interesting example of a controller is the node controller. This is the controller that is responsible for onboarding new nodes to the cluster and handling situations where these nodes become unavailable or are completely destroyed. Of course, to achieve this, this controller has to constantly monitor the nodes. By default, it does so every 5 seconds, a period known as the *node monitor period*. In case the node controller stops receiving a given node's heartbeat, the node controller waits 40 seconds for another heartbeat, a period known as the node monitor grace perior, and, if it does not get a heartbeat in that period, the node is marked as unreachable. After a node is marked as unreachable, the node controller waits for 5 minutes for it to come back up (pod eviction timeout). if it doesn't, then the pods assigned to that node are removed and provisioned on healthy nodes.

## Container runtime interface

The whole point of Kubernetes is to run containerized applications so we need something to actually run the containers, which is where the container runtime comes in. This is basically a dedicated piece of software used to run the containers and, naturally, it must exists on every worker node. Additionally, we can also have it running on the master node if we wish to host the control plane components as containers.

The Container Runtime Interface (CRI) was introduced in Kubernetes to allow for supporting multiple container runtimes. Basically, for a container runtime to work with Kubernetes, it must satisfy the following criteria:
* CRI - it must adhere to the defined interface;
* `imagespec` - its container images must be built according to the Open Container Initiative (OCI) specifications;
* `runtimespec` - the container runtime development must also adhere to the OCI specifications.

Some time ago, Kubernetes used mostly Docker for its container runtime. However, as things evolved, Kubernetes no longer supports Docker as a container runtime and most of its deployments employ `containerd` instead.

## `kubelet`

A `kubelet` is an agent running on each worker node in a Kubernetes cluster that actively listens for instructions from the `kube-apiserver` and deploys/destroys containers on the node as required. It also periodically publishes status reports that are fetched by the `kube-apiserver` that allow for monitoring the status of the nodes and of the containers on them. Note that the `kubelet` doesn't create/destroy the containers itself, but it does relay those orders to the container runtime.

## `kube-proxy`

Last but not least, we have the `kube-proxy`. This is another component that is running on every worker node and its main job is to ensure that the necessary rules are in place on the worker nodes to allow the containers on them to communicate with each other. It is essentially a pod networking solution, *i.e.*, an internal virtual network spanning all nodes in the cluster. I'll dive further into this topic in a future post, because there is a lot to unpack here. 

## Workflow for creating a pod

Now we are ready to look at what happens in a Kubernetes cluster when we perform a request to create a pod! So let's dive in. It begins, obviously, with a user performing a request to the `kube-apiserver` to create a pod. This component will first authenticate and then validate the request and will then create the pod without assigning it to any node. Naturally, the information on the `etcd` cluster is also updated. At this point, the user receives a message back from the cluster stating that the pod has been created. Afterwards, the `kube-scheduler` monitoring loop picks up the existence of an unassigned pod, identifies the right node to place it on, and relays this information back to the `kube-apiserver`, which promptly writes it to the `etcd` data store. Now, the `kube-apiserver` passes that information along to the `kubelet` that is on the appropriate worker node for it to create the pod on the node and instruct the container runtime engine to pull and deploy the application image. Finally, the `kubelet` updates the status back to the `kube-apiserver` which once again updates the data on the `etcd` server. 

While this might seem like a complicated workflow, it actually becomes fairly intuitive when we realize that Kubernetes is very modular in its construction, *i.e.*, every component on has one clearly defined function.
