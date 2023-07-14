---
title: "Adventures in Overengineering 2: a better Docker image"
date: 2023-03-21T20:25:26Z
categories: ["Adventures in Overengineering"]
description: "Deploying a slim Docker image into a microk8s kubernetes local cluster."
draft: true
---

Last post, we managed to do the following:
* created a Kubernetes clusters using `microk8s`;
* created a very simple Golang web server and its Docker image;
* created a Kubernetes deployment for the web server and a service to expose it.

In this post, we will improve on the Docker image we previously built because we can always do better and that is the point of this whole series: each post must represent an improvement on the previous post, either by enhancing capabilities or employing better practices. And, if I didn't do this post now, I would very quickly run out of space in my hard drive.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/askeladden/tree/v0.2
{{< /admonition >}}

## Better Docker image?

It isn't immediately clear what is meant by a "better Docker image", so let's recap, it will help us make some sense out of this and understand why it is a good change to make. Last time, the `Dockerfile` that we created was the following:

```Dockerfile
FROM golang:1.19

ADD . /askeladden
WORKDIR /askeladden

RUN go build

ENTRYPOINT ./askeladden
```

This resulted in a Docker image based on the official Golang 1.19 image with out application shipped alongside it. We still have this image saved on our local registry so let's take a look at it using `docker images`:

```plaintext
REPOSITORY                   TAG        IMAGE ID       CREATED      SIZE
askeladden                   latest     8324675e97c1   6 days ago   1e+03MB
```

It seems that our Docker image is about 1Gb in size, which is a tad bit too much. The reason the image is so large is because it has Golang installed, which we don't actually require for any purpose after the web server is built. The idea is simple: we create a container with Golang 1.19 and build our binary, just as before. But after building it, we create another container, based on a Debian image, and simply inject our binary into it. We can then discard the container we used for building the web server and we are left with a slimmer Docker image. 

So the first step is to pick the Debian image that we'll use. We have no special preference for the Debian version, so we can use the latest version, which is called `bookworm`. However, we will pick the `bookworm-slim` variant, which is a Debian `bookworm` version, but without extra files that aren't usually required by containers, such as manual pages and documentation.

```Dockerfile
FROM golang:1.19 AS builder

ADD . /askeladden
WORKDIR /askeladden

RUN go build

FROM debian:bookworm-slim
COPY --from=builder /askeladden/askeladden /usr/bin/askeladden

ENTRYPOINT ./usr/bin/askeladden
```

Note that we gave the alias `builder` to our first container by specifying `AS builder` and then injected the binary built inside it into the slimmed docker image with `COPY --from=builder`. Running `docker images` now gives us:

```plaintext
REPOSITORY                   TAG        IMAGE ID       CREATED         SIZE
askeladden                   latest     c2c2327d336a   5 minutes ago   81.1MB
```

Which means that our new image is only 81.1Mb in size, 10 times less than what the previous image was. To check if this image is working as excepted, let us create a container based on it and publish it's port so we can see our web server working by running:

```plaintext
docker run -d -p 8080:8090 askeladden
```

Navigating to `http://localhost:8080/` in a browser shows that this is working as expected. 

## Deploying a container with the slim Docker image

This is great, now we have to get our image into the cluster's registry. So first we tag it:

```plaintext
docker tag <image_ID> localhost:32000/askeladden:slim
```

Then we push it into the cluster's registry:

```plaintext
docker push localhost:32000/askeladden:slim
```

We can check if this image was successfully pushed into the cluster's registry by running:

```plaintext
curl http://localhost:32000/v2/askeladden/tags/list | jq '.'
```

{{< admonition type=warning title="curl and jq" open=true >}}
If you do not have `curl` nor `jq` installed you will obviously not be able to run the command above. But you should install them, they come in handy.
{{< /admonition >}}

{{< admonition type=note title="curl http://localhost:32000/v2/askeladden/tags/list | jq '.'" open=true >}}
`curl`, which stands for "Client for URL", is simply a command-line tool that can be used to transfer data to or from a web server. In this case, if we had simply written `curl http://localhost:32000/v2/askeladden/tags/list`, we would've obtained the desired result in JSON format but not "prettified". This is a fairly short output, so it isn't really required to "prettify" it, but it's a good practice nonetheless. For that matter, we pipe the result, using the pipe `|` operator, into `jq` which is a simple JSON processor. The `'.'` part tells `jq` that the way we want our JSON to be processed, is that we do not want it processed at all, we are merely here for the cosmetic enhancements.

Alternatively, we could've just navigated to `http://localhost:32000/v2/askeladden/tags/list` using a web browser, but it wouldn't look as cool as using the terminal.
{{< /admonition >}}

The above command outputs the following:

```json
{
  "name": "askeladden",
  "tags": [
    "registry",
    "slim"
  ]
}
```

Which means that our image is in the cluster's registry! However, our pod still isn't running the latest image. We can see that our pod is running with `mk get pods`:

```plaintext
NAME                                         READY   STATUS    RESTARTS       AGE
deployment-askeladden-dev-75c45dc7d9-r9hcf   1/1     Running   1 (112m ago)   6d
```

And we can inspect this pod using:

```plaintext
mk describe pod deployment-askeladden-dev-75c45dc7d9-r9hcf
```

{{< admonition type=note title="kubectl describe pod [PodName]" open=true >}}
The `describe pod [PodName]` command prints out a bunch of information on the existing pod. In fact, its usage is more general than this, since we can use it to describe any type of Kubernetes object. Its syntax is fairly simple and intuitive, it's always `describe [ObjectType] [ObjectName]`.
{{< /admonition >}}

This produces the following output:

```plaintext
Name:             deployment-askeladden-dev-75c45dc7d9-r9hcf
Namespace:        askeladden-dev
Priority:         0
Service Account:  default
Node:             luis-thinkpad-t490/192.168.1.83
Start Time:       Wed, 15 Mar 2023 21:56:14 +0000
Labels:           app=askeladden
                  pod-template-hash=75c45dc7d9
Annotations:      cni.projectcalico.org/containerID: a1ce6f36e0f31e62f1d8574b0e02579f441490c66c2c22f60f8440c69aabfe84
                  cni.projectcalico.org/podIP: 10.1.68.19/32
                  cni.projectcalico.org/podIPs: 10.1.68.19/32
Status:           Running
IP:               10.1.68.19
IPs:
  IP:           10.1.68.19
Controlled By:  ReplicaSet/deployment-askeladden-dev-75c45dc7d9
Containers:
  askeladden:
    Container ID:   containerd://3806e5e9dffe4fd9e9519e1dea1149af619d2d6bda25b9b39155699b383f65b1
    Image:          localhost:32000/askeladden:registry
    Image ID:       localhost:32000/askeladden@sha256:319994c8bbc2f0b69b5bdd26bfe3b0967e7858bd6426697b97c9c35b4da0b76a
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 21 Mar 2023 20:23:02 +0000
    Last State:     Terminated
      Reason:       Unknown
      Exit Code:    255
      Started:      Wed, 15 Mar 2023 21:56:15 +0000
      Finished:     Tue, 21 Mar 2023 20:22:42 +0000
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4cv2j (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-4cv2j:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason             Age                    From     Message
  ----     ------             ----                   ----     -------
  Warning  MissingClusterDNS  3m19s (x63 over 119m)  kubelet  pod: "deployment-askeladden-dev-75c45dc7d9-r9hcf_askeladden-dev(ebd02f7f-cf55-45ba-be93-a03eb5e270e4)". kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to "Default" policy.
```

That's a lot of text but we really only care about what is under `Containers/askeladden/Image`, which is `localhost:32000/askeladden:registry`. This is our old image, not the slim version of it. Which means we have to edit our deployment and apply it. In our `k8s/deployment.yaml` file, all we have to change is the last line so that it reads `image: localhost:32000/askeladden:slim`. Afterwards, running `mk apply -f k8s/deployment.yaml` will trigger the deployment with the new container image.

There is one cool detail about Kubernetes deployments. By default, they are configured to perform what is known as a rolling update. This means that, when a deployment is triggered, it will first create a new container, wait for it to run successfully, and only then terminate the old one. We can clearly see that if we type `mk get pods` a few seconds apart after triggering the deployment:

```plaintext
NAME                                         READY   STATUS        RESTARTS       AGE
deployment-askeladden-dev-6b766588c5-hkn2x   1/1     Running       0              19s
deployment-askeladden-dev-75c45dc7d9-r9hcf   1/1     Terminating   1 (127m ago)   6d
```

```plaintext
NAME                                         READY   STATUS    RESTARTS   AGE
deployment-askeladden-dev-6b766588c5-hkn2x   1/1     Running   0          34s
```

Navigating to `http://localhost:30008/` we can again check that our new container is, in fact, running as expected.