---
title: "Adventures in Overengineering 1: deploying a Golang web server"
date: 2023-03-15T20:46:20Z
tags: ["go", "kubernetes", "docker"]
categories: ["Adventures in Overengineering"]
draft: false
---

There is a wide variety of blog posts on the internet teaching one how to create a simple web server in Golang. But most of them are incredibly boring and minor variations of each other. So I'm going to take a different approach: I'm going to deploy a simple REST API in Golang. Notice that I said *deploy*, not create, which means that my first step is a tiny bit different.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/askeladden/tree/v0.1
{{< /admonition >}}


## Creating a Kubernetes Cluster

Yes, you read that right, the first step is to create our own Kubernetes cluster locally. Why? Because I'm too cheap to use any of the cloud providers out there and this has the upside of being a cool learning opportunity.

After some research, which is just a fancy way of saying "looking at the first page of Google results", I found that `microk8s` is the solution I am looking for getting a local Kubernetes cluster up and running. The official install guide seems straightforward enough so let's give it a go. First step:

```plaintext
sudo snap install microk8s --classic
```

That was quick enough. The second step in the documentation is to run

```plaintext
microk8s status --wait-ready
```

Let's do that. And this is where we find our first hurdle. The result of the above command is:

```plaintext
Insufficient permissions to access MicroK8s.
You can either try again with sudo or add the user luis to the 'microk8s' group:

    sudo usermod -a -G microk8s luis
    sudo chown -R luis ~/.kube

After this, reload the user groups either via a reboot or by running 'newgrp microk8s'.
```

Running the above as `sudo` does work, and it outputs the following:
```plaintext
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    dns                  # (core) CoreDNS
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    kube-ovn             # (core) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    minio                # (core) MinIO object storage
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    storage              # (core) Alias to hostpath-storage add-on, deprecated
```

I wish all obstacles in life were that easy to overcome. But it is cumbersome to write a few extra characters every time I want to interact with `microk8s`, so I'm going to also perform the other step suggested by `microk8s`, which is to add myself to the `microk8s` group.

{{< admonition type=note title="usermod -a -G microk8s luis" open=true >}}
In case you are not aware what `usermod` is, running `man usermod` on your Linux machine will elucidate you (`man` stands for `manual`, as in, instruction book, not as in manual labour, even though some of these things do sometimes feel like a chore).

The manual gives us the following short description of `usermod`: "modify a user account". Seems intuitive enough. However, the command `microk8s` is instructing us to run is `sudo usermod -a -G microk8s luis`, so we have to look a bit more carefully into the documentation to understand what these flags are.

The `-a` flag is shorthand notation for `--append` which is used to add a user to the group, and `-G` stands for `--groups` which is used to specify to which groups we want to add our user. So the command does exactly what one would expect: it adds the user `luis` (hey, that's me!) to the `microk8s`.
{{< /admonition >}}

{{< admonition type=note title="chown -R luis ~/.kube" open=true >}}
`man chown` tells us that this command is used to change the user and/or group ownership of a given file. We are running it with the `-R` flag which stands for `--recursive` and give it the user `luis` (hey, that's me again!) and the `~/.kube` directory. Basically, this sets us are the owners of everything inside the given directory.

I hope that I'm not the only one that always mistakenly reads `clown` instead of `chown`.
{{< /admonition >}}

{{< admonition type=note title="newgrp microk8s" open=true >}}
While the name might suggest that this command is used to create a new group, surprise, it is not what it does. What it does is change the current group ID during a login session. In other words, it logs me into the group.
{{< /admonition >}}

In the name of saving a few keystrokes and winning back a few seconds of precious lifetime, I have a set an alias for `microk8s kubectl` via

```plaintext
alias mk="microk8s kubectl"
```

So whenever you read `mk`, keep it mind that it stands for `microk8s kubectl`.

## Enabling the Kubernetes Dashboard

We are running our own local cluster, so we need some easy way to get visibility on what is inside it and what its state is. For that matter, we can use the Kubernetes dashboard add-on.

Apparently, it is supposed to be super easy to enable a new add-on with `microk8s`, so let's try getting the Kubernetes dashboard up and running by typing `microk8s enable dashboard` into our terminal. This outputs the following:
```plaintext
Infer repository core for addon dashboard
Enabling Kubernetes Dashboard
Infer repository core for addon metrics-server
Enabling Metrics-Server
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-admin created
Metrics-Server is enabled
Applying manifest
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
secret/microk8s-dashboard-token created

If RBAC is not enabled access the dashboard using the token retrieved with:

microk8s kubectl describe secret -n kube-system microk8s-dashboard-token

Use this token in the https login UI of the kubernetes-dashboard service.

In an RBAC enabled setup (microk8s enable RBAC) you need to create a user with restricted
permissions as shown in:
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
```

So a bunch of stuff happened. At least `microk8s` told us what happened instead of making all of these operations under the hood which means that now we have the pleasure of figuring out what the hell just happened! On line 1, `microk8s` seems to just be searching for the repository with the desired add-on and line 2 begins the process of enabling it. This is where the fun really begins. The first step in enabling the Kubernestes dashboard is, on line 3, is to search for a repository for the `metrics-server` and begin enabling it on line 4.

Each Kubernetes node has a kubelet which is responsible for managing pod deployments and instructing the container runtime to start or stop containers are required, and the `metrics-server` scrapes resource metrics from each existing kubelet and exposes them in the Kubernetes API server via the metrics API to be used by the Horizontal and Vertical Pod Autoscalers (HPA and VPA). It should be noted that its purpose is not to serve as a monitoring solution, its metrics are to be used by HPA and VPA, but the metrics it collects do show up in the Kubernetes dashboard, so that is why it is has to be enabled. It is only after the `metrics-server` is enabled that `microk8s` will begin creating the objects required to enable the dashboard. 

To access the dashboard, run `microk8s dashboard-proxy`, which outputs:
```plaintext
Checking if Dashboard is running.
Infer repository core for addon dashboard
Infer repository core for addon metrics-server
Waiting for Dashboard to come up.
Trying to get token from microk8s-dashboard-token
Waiting for secret token (attempt 0)
Dashboard will be available at https://127.0.0.1:10443
Use the following token to login:
[TOKEN APPEARS HERE]
```

Just navigate to the specified page, input the token, and the dashboard is ready to use.

## Creating the web server

We are doing this in Go, so the first step is to create a directory and a Go module by running `go mod init`. In my case, since my GitHub name is `ornlu-is` and I decided to name the package `askeladden`, this equates to running:

```plaintext
go mod init github.com/ornlu-is/askeladden
```

This will create a `go.mod` file that is used by Go to track the code's dependencies. Now we can create our `main.go` file and get to coding! The code we need to add to this file is the following:

```go
package main

import (
	"fmt"
	"net/http"
)

func rootPathHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "Strange women lying in ponds distributing swords is no basis for a system of government.\n")
}

func main() {
	http.HandleFunc("/", rootPathHandler)

	err := http.ListenAndServe(":8090", nil)
	if err != nil {
		panic(err)
	}
}
```

You might be thinking: "This web server does nothing except print out a Monty Python quote". Yes, that is exactly what it does. I did say this was a simple web server, and it doesn't get much easier than this. You can test this out by running `go run main.go` and navigating to `http://localhost:8090/`.

## Containerizing our web server

{{< admonition type=warning title="Docker" open=true >}}
This section assumes that you have Docker installed and configured on your machine. If not, go check out the official documentation, it's pretty high quality.
{{< /admonition >}}

Kubernetes is defined as being a system for automating deployment, scaling, and management of containerized applications. We have a Kubernetes cluster, we have an application, but we still need to containerize it! The first step is to write a `Dockerfile` in our root directory. I am going to be using the image for Golang 1.19, to match the version I am running locally, and have the image simply build and run the application.


```Dockerfile
FROM golang:1.19

ADD . /askeladden
WORKDIR /askeladden

RUN go build

ENTRYPOINT ./askeladden
```

But before testing this in our cluster, we should do a local sanity check to be sure everything is working as expected. Firstly, we have to build our image, which is as simple as being in the same directory of the `Dockerfile` and running 

```plaintext
docker build . -f Dockerfile -t 'askeladden'
```

{{< admonition type=note title="docker build . -f Dockerfile -t 'askeladden'" open=true >}}
When we give the Docker CLI the `build` command, we are telling it to build an image. Images are later used to create containers, which are instances of a given image. The `.` refers to the path of the context, which, in the context (ha-ha, funny guy) of Docker, refers to the set of files that can be used by the build process. The `-f` is shorthand notation for `--file` and is the path to the `Dockerfile`, while `-t` is short for `--tag` and allows us to name/tag our image. In this case, I only named it.
{{< /admonition >}}

We can check our image by listing all images built by Docker using `docker images`:

```plaintext
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
askeladden   latest    8324675e97c1   57 seconds ago   1e+03MB
```

Great, the image is there! So now we have to create and run a container based on this image to check if our web server is working. This can be achieved by running:

```plaintext
docker run -d -p 88:8090 askeladden
```

{{< admonition type=note title="docker run -d -p 88:8090 askeladden" open=true >}}
The `run` command creates and runs a container. The `-d` flag, which stands for `--detach`, will run the container in the background of the terminal, meaning it will not block the terminal window. Since the container has its own network, we have to expose the port `8090` where our web server can be accessed inside the container to outside the container. This is achieved by publishing the port and mapping it into a port on the host machine, *i.e.*, my computer, with the `-p` flag followed by the specification of the host port and container port in the `<host_port>:<container_port>` format. Finally, we specify the name of the image we want to build our container from which, in this case, is just `askeladden`. Docker will first look for this image locally and, if it doesn't find it, it will attempt to retrieve it from a public image repository.
{{< /admonition >}}

We can see the list of running containers by typing `docker ps`, and we can check that it is properly running by navigating to `http://localhost:88/`. So, now we're fairly certain that our web server is containerized as desired, so we can stop the container using the `docker stop <container_ID>` command (you can retrieve the container ID from the output of `docker ps`). 

Now the output of `docker ps` doesn't show anything but the container wasn't actually deleted, it was merely stopped. If we type `docker ps -a`, where `-a` stands for `--all`, we can see that our container is in an `Exited` status. To avoid taking up resources, let's just delete it with `docker rm <container_ID>`.

## Creating a Kubernetes Namespace

{{< admonition type=warning title="Kubernetes dashboard" open=true >}}
In this section, most, if not all, of the commands that I will run that `get` or `describe` a given Kubernetes objectives can be replaced by using the Kubernetes dashboard that was previously enabled. But it will not make you look as cool as if you did everything in the terminal.
{{< /admonition >}}

We want to deploy the web server into our cluster in a way that isolates it from the remaining pods that are running in it, which means that we have to deploy it into a separate namespace. To get a list of what namespaces are already configured on the cluster, we can simply run `mk get namespaces`:

```plaintext
NAME              STATUS   AGE
kube-system       Active   23h
kube-public       Active   23h
kube-node-lease   Active   23h
default           Active   23h
```

We can see that there are several namespaces whose name starts with `kube`, meaning that these are Kubernetes system namespaces and we do not want to tinker with those, at least for now. We could deploy our web server to the `default` namespace (which would be where it would go if we did not specify any namespace), but it will be helpful in the future to have this running in its own namespace.

So, let's start creating Kubernetes definition files! Create a directory called `k8s` and a file inside it named `namespace.yaml` with the following content:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: askeladden-dev
```

To create a Kubernetes object from a `.yaml` file, all we have to do is run 

```plaintext
mk create -f k8s/namespaces.yaml
```

We can easily check if it has been created by running `mk get namespaces` again:
```plaintext
NAME              STATUS   AGE
kube-system       Active   23h
kube-public       Active   23h
kube-node-lease   Active   23h
default           Active   23h
askeladden-dev    Active   5h51m
```

However, Kubernetes objects will still be created into the `default` namespace if the target namespace is not specified. Again, in typical software engineering fashion, we will go to great lengths to save a few keystrokes.

We can check our Kubernetes config with `mk config view` to look for what contexts are configured. Contexts are merely aliases for cluster parameters to make our lives easier.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:16443
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
current-context: microk8s
kind: Config
preferences: {}
users:
- name: admin
  user:
    token: REDACTED
```

There is only one context, `microk8s`, so it is obvious which context we are using. But if it weren't as obvious, we could get that information by running

```plaintext
mk config current-context
```

The context that we are using by default does not have any namespace associated with which means, you've guessed it, the namespace that we default to is the `default` namespace. This also means we have to create our own context, which can be readily achieve through:

```plaintext
mk config set-context askeladden-dev --namespace=askeladden-dev --cluster=microk8s-cluster --user=admin
```

It is simply a context that, when used, will automatically fill unspecified cluster parameters with the ones in the context. We can now see that our `config` has a new context:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:16443
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    namespace: askeladden-dev
    user: admin
  name: askeladden-dev
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
current-context: askeladden-dev
kind: Config
preferences: {}
users:
- name: admin
  user:
    token: REDACTED
```

All that is needed is to switch to it using:

```plaintext
mk config use-context askeladden-dev
```

The context? `askeladden-dev`. The keystrokes? 20% less. Great success.


## Deploying our containerized web server into our Kubernetes cluster

So what is running in the namespace we just created? We can figure this out by running:

```plaintext
mk get all
```

Which readily lets us know that

```plaintext
No resources found in askeladden-dev namespace.
```

There is one issue that I haven't addressed yet. We built the container image locally, but the cluster does not have access to the Docker daemon that is running on my machine and thus it cannot use the image we have built to create the container. To first step to taking care of this is to enable the registry in `microk8s` via:

```plaintext
microk8s enable registry
```

Which outputs the following:

```
Infer repository core for addon registry
Infer repository core for addon hostpath-storage
Enabling default storage class.
WARNING: Hostpath storage is not suitable for production environments.
deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon.
The registry will be created with the size of 20Gi.
Default storage class will be used.
namespace/container-registry created
persistentvolumeclaim/registry-claim created
deployment.apps/registry created
service/registry created
configmap/local-registry-hosting configured
```

Again, a bunch of stuff happened. `microk8s` created all the objects required to have its own registry. So now we have to push our local image into the registry that is in the cluster, so that `microk8s` can use that image to create our container.

For that matter, we first get our image ID from the output of `docker images`, and then we tag it with:

```plaintext
docker tag <image_ID> localhost:32000/askeladden:registry
```

Note that `localhost:32000` is a port on our host machine that is connected to the cluster's registry. Now all that is left to do is to push this image into the registry with:

```
docker push localhost:32000/askeladden:registry
```

Now we are ready to deploy our container to the cluster! All that we need now is to create a Kubernetes definition file `k8s/deployment.yaml`, with the following contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-askeladden-dev
  labels:
    env: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: askeladden
  template:
    metadata:
      labels:
        app: askeladden
    spec:
      containers:
      - name: askeladden
        image: localhost:32000/askeladden:registry
```

This will create three different Kubernetes objects: a `Pod` with our container running inside it, a `ReplicaSet` which ensures that we always have a given number of copies of the `Pod` running (in this case, it's just one), and a `Deployment` object, which handles how `Pod`s are updated, rolled out, scaled, etc. We can see all of these objects by running:

```plaintext
mk get all
```

Which outputs this OCD triggering table formatted result:

```plaintext
NAME                                             READY   STATUS    RESTARTS   AGE
pod/deployment-askeladden-dev-75c45dc7d9-r9hcf   1/1     Running   0          78s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-askeladden-dev   1/1     1            1           78s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-askeladden-dev-75c45dc7d9   1         1         1       78s

```

But (there's always a "but"), our web server is running inside the container, which is inside the pod, which is inside the cluster. We cannot access it as we have before because we have not exposed a port in the cluster with which to communicate with the server. To solve this, we have to create a Kubernetes `Service` of the `NodePort` kind (this is not the only option, it is just the option I felt like doing right now). As before, we create a new Kubernetes definition file, `k8s/service.yaml`, with the following contents:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: askeladden-dev-service
  labels:
    app: askeladden
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8090
    nodePort: 30008
    protocol: TCP
  selector:
    app: askeladden
```

There are a few things to note here. The `targetPort` is the port on the `Pod` that we want to expose. In our cause, this is where the web server is running, so it is `8090`. The `port` is the port on the `Service` that connects to the `targetPort`. Finally, the `nodePort` is the port of the node that is exposed to the outside world. The `Service` searches for `Pod`s with labels matching those given under `selector` and exposes their `targetPort` via the `nodePort`.

After creating the service the usual way, we can new see a new Kubernetes object in our namespace:

```plaintext
NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/askeladden-dev-service   NodePort   10.152.183.55   <none>        80:30008/TCP   3s
```

## Future Overengineering Steps

It's all about saving those precious keystrokes, and there are two major keystroke consumers as of now:
* Having to push the Docker images into the cluster's registry manually;
* Having to create the Kubernetes objects manually.

So the next objective might be to reduce one of those. Maybe. I don't know.