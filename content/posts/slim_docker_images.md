---
title: "Slim Docker Images via Build-step Containers"
date: 2023-07-14T17:48:07+01:00
categories: ["Study Notes"]
description: "How to get slim Docker images using build-step containers."
draft: false
---

Docker images are supposed to be as small as possible, containing only what is absolutely required for the application inside them to run. In this post, I'll go over build-step containers and how to use them with Docker. For that matter let us consider an example Go application, nothing fancy, like the one given by the code snippet below:

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

This application simply starts a web server on port 8090 that prints a simple Monty Python quote.

## Containerizing the application

The first step is to write a Dockerfile in our root directory. I am going to be using the image for Golang 1.19, to match the version I am running locally, and have the image simply build and run the application.

```Dockerfile
FROM golang:1.19

ADD . /askeladden
WORKDIR /askeladden

RUN go build

ENTRYPOINT ./askeladden
```

Firstly, we have to build our image, which is as simple as being in the same directory of the Dockerfile and running

```bash
docker build . -f Dockerfile -t 'not-so-slim-shady'
```
{{< admonition type=note title="docker build . -f Dockerfile -t 'not-so-slim-shady'" open=true >}}
When we give the Docker CLI the `build` command, we are telling it to build an image. Images are later used to create containers, which are running instances of a given image. The `.` refers to the path of the context, which, in the context (ha-ha, funny guy) of Docker, refers to the set of files that can be used by the build process. The `-f` is shorthand notation for `--file` and is the path to the `Dockerfile`, while `-t` is short for `--tag` and allows us to name/tag our image. In this case, I only named it.
{{< /admonition >}}

We can check our image by listing all images built by Docker using `docker images`:

```
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
not-so-slim-shady   latest    3673b4f2f4c8   14 seconds ago   1.07GB
```

Great, the image is there and taking up 1.07GB! So now we have to create and run a container based on this image to check if our web server is working. This can be achieved by running:

```plaintext
docker run -d -p 88:8090 not-so-slim-shady
```

{{< admonition type=note title="docker run -d -p 88:8090 not-so-slim-shady" open=true >}}
The `run` command creates and runs a container. The `-d` flag, which stands for `--detach`, will run the container in the background of the terminal, meaning it will not block the terminal window. Since the container has its own network, we have to expose the port `8090` where our web server can be accessed inside the container to outside the container. This is achieved by publishing the port and mapping it into a port on the host machine, *i.e.*, my computer, with the `-p` flag followed by the specification of the host port and container port in the `<host_port>:<container_port>` format. Finally, we specify the name of the image we want to build our container from which, in this case, is just `not-so-slim-shady`. Docker will first look for this image locally and, if it doesn't find it, it will attempt to retrieve it from a public image repository.
{{< /admonition >}}

We can see the list of running containers by typing `docker ps`, and we can check that it is properly running by navigating to `http://localhost:88/`. So, now we're fairly certain that our web server is containerized as desired, so we can stop the container using the `docker stop <container_ID>` command (you can retrieve the container ID from the output of `docker ps`). 

Now the output of `docker ps` doesn't show anything but the container wasn't actually deleted, it was merely stopped. If we type `docker ps -a`, where `-a` stands for `--all`, we can see that our container is in an `Exited` status. To avoid taking up resources, let's just delete it with `docker rm <container_ID>`.

## Using a build-step container

The procedure above resulted in a Docker image based on the official Golang 1.19 image with our application shipped alongside it and, as we've seen above, our Docker image is about 1Gb in size, which is a tad bit too much. The reason the image is so large is because it has Golang installed, which we don't actually require for any purpose after the web server is built. The idea is simple: we create a container with Golang 1.19 and build our binary, just as before. But after building it, we create another container, based on a Debian image, and simply inject our binary into it. We can then discard the container we used for building the web server and we are left with a slimmer Docker image. 

So the first step is to pick the Debian image that we'll use. We have no special preference for the Debian version, so we can use the latest version, which is called `bookworm`. However, we will pick the `bookworm-slim` variant, which is a Debian `bookworm` version, but without extra files that aren't usually required by containers, such as manual pages and documentation.

```Dockerfile
FROM golang:1.19 AS builder

ADD . /repo
WORKDIR /repo

RUN go build -o bin/example

FROM debian:bookworm-slim
COPY --from=builder /repo/bin/example /usr/bin/example

ENTRYPOINT ./usr/bin/example
```

Note that we gave the alias `builder` to our first container by specifying `AS builder` and then injected the binary built inside it into the slimmed docker image with `COPY --from=builder`. Running `docker images` now gives us:

```
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
slim-shady          latest    7c20d0afa6c3   4 seconds ago    81.3MB
not-so-slim-shady   latest    74f9c0e9c60d   15 minutes ago   1.07GB
```

Which means that our new image is only 81.3Mb in size, 10 times less than what the previous image was. To check if this image is working as excepted, let us create a container based on it and publish it's port so we can see our web server working by running:

```
docker run -d -p 88:8090 slim-shady
```

Navigating to `http://localhost:8080/` in a browser shows that this is working as expected. 
