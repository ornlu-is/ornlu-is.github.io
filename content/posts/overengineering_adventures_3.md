---
title: "Adventures in Overengineering 3: manning the Helm"
date: 2023-03-26T23:35:24+01:00
categories: ["Adventures in Overengineering"]
description: "Using helm on a local Kubernetes cluster"
draft: false
---

Ahoy! It's time to include Helm in our project! Why? Three reasons: making our life easier when it comes to managing Kubernetes resources, staying true to the overengineering game by introducing yet another tool, and will also allow me to write more pirate related puns than what should be legally allowed. I apologize in advance for my terrible sense of humour. It is recommended to read this post while listening to "Under Jolly Roger" by Running Wild or any other pirate song.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/askeladden/tree/v0.3
{{< /admonition >}}

## What is this "Helm"? Do I look like a sailor to you?

First, a fun fact. The name Kubernetes stands for "helmsman" or "pilot" in Greek, and its logo is actually a helm (I cannot believe I missed this before), which justifies the naming choice for the tool we are about to introduce into the project. Helm is basically a package manager for Kubernetes, and works through specification of charts (this is not a pun, this their the actual name). In this context, a chart is simply a collection of files that describe a related set of Kubernetes resources. Using this tool yields the following advantages:
* Reduces the complexity of deployments since Kubernetes requires resources to be created in a given order and Helm takes care of that for us;
* Reproducible deployments, since a Helm guarantees that what is in the chart matches the configuration in the Kubernetes cluster;
* Easier to rollback to previous deployments since Helm maintains a comprehensive revision history;
* Allows us to sail under the Jolly Roger.

With all this in mind, let's begin adapting our project to support Helm!

## Setting sails with Helm

First, let us look at our current directory tree. We can do this by running the `tree` command in our root directory, which outputs the following:
```plaintext
.
├── Dockerfile
├── go.mod
├── k8s
│   ├── deployment.yaml
│   ├── namespace.yaml
│   └── service.yaml
├── main.go
└── README.md
```

{{< admonition type=warning title="tree" open=true >}}
You have to install `tree` since it is not shipped with most operating systems. If, like me, you are running Ubuntu, you can install it using `sudo apt install tree`.
{{< /admonition >}}

I want to use Helm uniquely to control the deployment of my web server (in case you haven't read my previous posts, this is an incredible web server that simply prints a Monty Python quote). The first step is to create our Helm chart. For that matter, we have to create the following directories and files in our project's root directory:
```plaintext
chart
└── askeladden
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   └── service.yaml
    └── values.yaml
```

{{< admonition type=note title="helm create chart" open=true >}}
Alternatively, we could've used `helm create chart` which would create the Helm chart for Nginx, but then we'd have to delete several of the files and most of their content, so it is easier to create the files manually, since this implies less keystrokes and, in turn, a longer finger joint lifetime.
{{< /admonition >}}

These are almost as many files as what we already had in our project, so it is worth going over their purpose one by one:
* `Chart.yaml` - metadata on the chart itself;
* `values.yaml` - contains the default values for our chart that will be injected by Helm into the templates;
* `templates/_helpers.tpl` - contains utilities to be used across templates;
* `templates/deployment.yaml` - this will be the template for our Kubernetes deployment;
* `templates/service.yaml` - and this will be the template for our Kubernetes service.

We can already fill out the contents of `Chart.yaml` with the following:
```yaml
apiVersion: v2
name: askeladden
description: A web server that barely does anything
version: 0.0.1
appVersion: "0.0.3"
```

Where each field has the following function:
* `apiVersion` specifies the Helm version we are using;
* `name` is the name of the chart. Here I named it the same as our application because naming things is hard;
* `description` is self-explanatory;
* `version` is the chart version;
* `appVersion` is the application version, which is "0.0.3" since this is the third post in the series.

However, for the remaining files, it is required that we use [yo-ho-Golang's template syntax](https://pkg.go.dev/text/template), since is that is what confers Helm its flexibility. We have a bunch of repeated fields in our Kubernetes files such as the labels used for the `matchSelector` or the `name`, and we have some values that can be seen as overall configuration values and, as such, should be in the `values.yaml` file to be injected into the Kubernetes definition files by Helm. The objective is to have helm output a Kubernetes manifest that matches exactly what we previously had. Let us begin by writing the following into our `values.yaml` file:
```yaml
replicaCount: 1

image: localhost:32000/askeladden:slim

labels:
  app: askeladden

selectorLabels:
  app: askeladden

deploymentLabels:
  env: dev

service:
  type: NodePort
  port: 80
  targetPort: 8090
  nodePort: 30008
```

Then, let us define our helpers. These are simple reusable Go template shortcodes that can be used to improve readability of the templates. We have defined three different label dictionaries in our `values.yaml`, so let us create helpers that will fill the templates with the values of these dictionaries. Thus, our `_helper.tpl` has the following content:

```tpl
{{/*
Get the selector labels dictionary
*/}}
{{- define "chart.selectorLabels" }}
{{- range $key,$value := .Values.selectorLabels }}
{{- $key }}: {{ $value }}
{{- end}}
{{- end }}

{{/*
Get the labels dictionary
*/}}
{{- define "chart.labels" }}
{{- range $key,$value := .Values.labels }}
{{- $key }}: {{ $value }}
{{- end}}
{{- end }}

{{/*
Get the deployment labels dictionary
*/}}
{{- define "chart.deploymentLabels" }}
{{- range $key,$value := .Values.labels }}
{{- $key }}: {{ $value }}
{{- end}}
{{- end }}
```

Now, if we want to call any of the helpers we defined, we would use, for example `{{- include "chart.labels" . }}`, where `.` represents the object that contains all the information in the `values.yaml` file. Using the helpers, we get the following service template:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}
  labels:
    {{- include "chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    nodePort: {{ .Values.service.nodePort }}
    protocol: TCP
  selector:
    {{- include "chart.selectorLabels" . | nindent 6 }}
```

Note that there is a subtle difference between specifying `{{ .... }}` and `{{- ... }}`: the latter will remove all trailing whitespace, while the former will not. That is why we pipe the results of the helpers' calls into `nindent`, so that we can specify the indentation in a more taylored fashion. Following the same paradigm, we get the deployment template below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    {{- include "chart.deploymentLabels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image }}
```

To check if we have done everything properly and that we have a valid YAML file, we can run `helm template chart/askeladden`, which outputs:
```yaml
---
# Source: askeladden/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: askeladden
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
---
# Source: askeladden/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: askeladden
  labels:
    app: askeladden
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
        image: localhost:32000/askeladden:slim
```

Since we had no error messages, we have a YAML file, which is great! Note that if did have any errors, we would only see the error message in the terminal, without the rendered Kubernetes manifest. In that case, to see what is wrong with the manifest, we would add the `--debug` flag to the aforementioned command, which will print the rendered manifest along with the detected errors.

## Swab the project's structure

Notice that we had three Kubernetes resource definition files but we left one of them out of our Helm chart. This is because the namespace is not a part of our web server, but a part of our cluster, and, as such, we want to keep it out of our chart. For that matter, we will create an `infrastructure` directory which will in turn have a `k8s` directory and we will place our `namespace.yaml` file there. Then, we create a `services/askeladden` directory and place the remaining contents there, except the `go.mod` file. I am a big fan of documenting stuff properly (as this blog highlights), so I also created a `README.md` for the `askeladden` service. Our directory tree ends up as follows: 

```plaintext
askeladden/
├── go.mod
├── infrastructure
│   └── k8s
│       └── namespace.yaml
├── README.md
└── services
    └── askeladden
        ├── chart
        │   └── askeladden
        │       ├── Chart.yaml
        │       ├── templates
        │       │   ├── deployment.yaml
        │       │   ├── _helpers.tpl
        │       │   └── service.yaml
        │       └── values.yaml
        ├── Dockerfile
        ├── main.go
        └── README.md
```

## Land ho! Deploy app!

Now we need to use Helm to deploy our service! We can see that our service is still running if we run `mk get all`, we get the following output:

```plaintext
NAME                                             READY   STATUS    RESTARTS       AGE
pod/deployment-askeladden-dev-6b766588c5-hkn2x   1/1     Running   3 (139m ago)   5d11h

NAME                             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/askeladden-dev-service   NodePort   10.152.183.55   <none>        80:30008/TCP   11d

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-askeladden-dev   1/1     1            1           11d

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-askeladden-dev-6b766588c5   1         1         1       5d11h
replicaset.apps/deployment-askeladden-dev-75c45dc7d9   0         0         0       11d
```

There is something odd, we have our first ever replica set still hanging around. Kubernetes keeps old replica sets around because that is what it uses in case we need to rollback our deployment. However, we are now using Helm, so we do not need that anymore, since Helm has its own revision history. Let us delete our deployment with `mk delete deployment.apps/deployment-askeladden-dev`.

{{< admonition type=note title="mk delete deployment.apps/deployment-askeladden-dev" open=true >}}
We would usually have to write this command in the `mk delete <resource_type> <resource_name>`. However, since we are passing the `delete` command the name of the deployment in the `<resource_type>/<resource_name>` format, we only require one argument for the command.
{{< /admonition >}}

Similarly, we will delete our service with `mk delete service/askeladden-dev-service`. Running `mk get all` now shows that we have no resources in our namespace:

```plaintext
No resources found in askeladden-dev namespace.
```

We are good to go to use Helm now! However, just to be on the safe side of things, we want to test our chart a bit more. We already know that it is a valid YAML file, so now we want to know if it generates any possible errors in our Kubernetes cluster. For that matter, we can use the following command:

```plaintext
microk8s helm install askeladden chart/askeladden --dry-run
```

{{< admonition type=note title="microk8s helm install askeladden chart/askeladden --dry-run" open=true >}}
The `helm install <name> <chart_path>` command install a given chart on the cluster with the given name. The trick here is the `--dry-run` flag, which validates the manifests by communicating with the Kubernetes cluster. Pretty neat!
{{< /admonition >}}

This command outputs the following:

```plaintext
NAME: askeladden
LAST DEPLOYED: Mon Mar 26 11:02:41 2023
NAMESPACE: askeladden-dev
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: askeladden/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: askeladden
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
---
# Source: askeladden/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: askeladden
  labels:
    app: askeladden
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
        image: localhost:32000/askeladden:slim
```

Everything seems in order, meaning that we can board the ship and deploy by running the same command without the `--dry-run` flag.

```plaintext
NAME: askeladden
LAST DEPLOYED: Mon Mar 26 11:03:41 2023
NAMESPACE: askeladden-dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

No error messages seems like a good sign. Let us check if we have any Kubernetes resources on our namespace with `mk get all`:

```plaintext
NAME                              READY   STATUS    RESTARTS   AGE
pod/askeladden-6b766588c5-xjm7t   1/1     Running   0          6s

NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/askeladden   NodePort   10.152.183.151   <none>        80:30008/TCP   6s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/askeladden   1/1     1            1           6s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/askeladden-6b766588c5   1         1         1       6s
```

All the necessary resources have been create, meaning that we have only one validation step left: navigating to `http://localhost:30008/`. Opening the browser with that link shows the following message:

```plaintext
Strange women lying in ponds distributing swords is no basis for a system of government.
```

We have successfully converted our Kubernetes configurations files into a Helm chart and deployed it without any hassle! We can now peacefully set sails to the next overengineering adventure!
