# Adventures in Overengineering 4: using Terraform for Kubernetes namespace creation


This post has one very simple and short goal: to use Terraform to create a Kubernetes namespace to which we will migrate the application that we've been working on. Now, you might ask yourself: "Is it overkill to use Terraform just to create Kubernetes namespaces?" To which the answer is: yes, yes it is. But will that stop me? Short answer "no", long answer "no, it will not". The name of the series is "*Adventures in Overengineering*" and I plan to remain faithful to that name until I run out of ideas.

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/askeladden/tree/v0.4
{{< /admonition >}}

## What is Terraform? 

A concept that quickly rose to popularity in recent times is the concept of infrastructure as code, IaC, where, as the name suggests, infrastructure is managed and provisioned via code instead of manual processes. This has a number of advantages, such as being able to fully leverage a version control system to track, and possibly rollback, infrastructure. Naturally, it also makes distributing infrastructure configuration across developers and teams much easier. One popular tool for this paradigm, is Terraform, which takes a declarative approach to IaC, by allowing developers to write files which declare the currently desired state of the infrastructure. There are many other technologies that can be used for this purpose, but Terraform is the one I am used to using, so that is what I'm going to go with.

## Setting up Terraform with a local Kubernetes cluster

First things first, we have to install Terraform. For that matter, we can simply follow the [official documentation](https://developer.hashicorp.com/terraform/docs), which happens to be very straightforward. Now we just need one last step before we are good to go. Since I am using `microk8s`, its configuration isn't saved under `~/.kube/config`. We can take care of this with one simple command:

```plaintext
microk8s config > ~/.kube/config
```

And we are done, installation-wise. That was quick.

## Creating the necessary files to create a Kubernetes namespace with Terraform

We will be using Terraform for managing infrastructure, which means that the files we create for this purpose already have one natural destination: our `infrastructure/` directory. We create a new directory inside `infrastructure`, called `terraform`, and populate it with the following files:

```plaintext
terraform/
├── backend.tf
├── main.tf
├── provider.tf
└── versions.tf
```

Let us go over these one by one, starting with `backend.tf`, which, as the name suggests, specifies what backend terraform should use for remote state management and state locking. In this case, we will be using the `kubernetes` backend, which stores the Terraform state as a Kubernetes secret:

```tf
terraform {
  backend "kubernetes" {
    secret_suffix   = "state"
    config_path     = "~/.kube/config"
    namespace       = "terraform"
  }
}
```

If we look at the Terraform documentation for this backend type, we see that this backend is limited by Kubernetes maximum secret size, which is 1MB. This might be troublesome in the future but for now, our mindset is going to be "if it breaks, it breaks" and then we'll fix it when necessary. Since we are using the Kubernetes backend and we want to use Terraform to create a Kubernetes namespace, we also need to specify a provider, in `provider.tf`, which, in the Terraform nomenclature, is a set of plugins to interact with resources:

```tf
provider "kubernetes" {
  config_path       = "~/.kube/config"
  config_context    = "microk8s"
}
```

Providers are plugins and plugins have different versions depending on the underlying API they interact with, which means that we should specify the provider version we want in `versions.tf`:

```tf
terraform {
  required_providers {
    kubernetes = {
      source = "hashicorp/kubernetes"
    }
  }
}
```

I didn't really specify the version I want, so we will just end up using the latest version, if it breaks, it breaks, but I still enjoy having everything organized into different files. Finally, we will create the resources we want under `main.tf` which, in this case, is a Kubernetes namespace:

```tf
resource "kubernetes_namespace" "askeladden-staging" {
  metadata {
    name = "askeladden-staging"
  }
}
```

Note that we are using the Kubernetes backend, which will create a Kubernetes secret storing the Terraform state files. The secret has to go somewhere and, in `backend.tf`, I specified that it is goes into the `terraform` namespace, because I do not want things going into the default namespace. The namespace needs to exist prior to applying the Terraform changes so I'm also going to add `infrastructure/k8s/tf-namespace.yaml` file with the following contents:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: terraform
```

Yes, I know, this kind of defeats the purpose of having terraform generate the namespaces for me, but I do not want to use the default Kubernetes namespace and I cannot use a Kubernetes Terraform backend in a namespace that does not exist.

## Using Terraform

{{< admonition type=warning title="mk" open=true >}}
If you haven't read my previous posts, you will probably wonder what the `mk` command is. It is simply an alias for `microk8s kubectl`. Go read my previous posts, this is an adventure, you cannot start halfway.
{{< /admonition >}}

We are ready to begin! Let's first have a look at what namespaces we have configured in our cluster using `mk get namespaces`:
```plaintext
NAME                 STATUS   AGE
kube-system          Active   15d
kube-public          Active   15d
kube-node-lease      Active   15d
default              Active   15d
askeladden-dev       Active   14d
container-registry   Active   14d
```

The `askeladden-dev` namespace is where our web server is deployed. Unfortunately for this namespace, it will not live for much longer. First, let's create the new `terraform` namespace, which will host the Terraform state files, using:

```plaintext
mk apply -f infrastructure/k8s/tf-namespace.yaml
```

Running `mk get namespaces` once again now shows our newly born namespace:
```
NAME                 STATUS   AGE
kube-system          Active   15d
kube-public          Active   15d
kube-node-lease      Active   15d
default              Active   15d
askeladden-dev       Active   14d
container-registry   Active   14d
terraform            Active   9s
```

With the `terraform` namespace created, we can now navigate to `/infrastructure/terraform/` running:

```plaintext
terraform init
```

{{< admonition type=note title="terraform init" open=true >}}
`terraform init` initializes a working directory containing Terraform configuration files and should always be ran prior to writing a new Terraform configuration. This will create a `.terraform/` directory, and a `.terraform.lock.hcl` file, along with `.tfstate` files in case the specified backend is local (in our case, we are using the Kubernetes backend, so the state files are kept in the Kubernetes cluster). The `.terraform.lock.hcl` file should be included in the version control repository, to be used as a discussion medium abour potential changes to external dependencies via code review. On the other hand, the `.terraform/` directory is basically a local cache, and should not be commited. 
{{< /admonition >}}

This will output the following:

```plaintext
Initializing the backend...

Successfully configured the backend "kubernetes"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding latest version of hashicorp/kubernetes...
- Installing hashicorp/kubernetes v2.19.0...
- Installed hashicorp/kubernetes v2.19.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

The aforementioned command also created a directory `.terraform` and a `.terraform.lock.hcl` file. To avoid forgetting about the existence of the `.terraform` and accidentally commiting this to my repository, I'll create a `.gitignore` file with the contents presented below:

```plaintext
**/.terraform/
```

{{< admonition type=note title=".gitignore init" open=true >}}
If you have any intentionally untracked files and you want `git` to ignore them, you can create a `.gitignore` file at the root of the project's repository. You can then specify patterns inside this file, one patter per line, which `git` will use to match files the in project and ignore these matched files. This effectively means that these files will not exist in the remote repository.
{{< /admonition >}}

If we want to see what resources Terraform will create/modify/destroy, we can use the `terraform` plan, which provides a pretty output of stating what will happen if the Terraform changes are applied:

```plaintext
Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # kubernetes_namespace.askeladden-staging will be created
  + resource "kubernetes_namespace" "askeladden-staging" {
      + id = (known after apply)

      + metadata {
          + generation       = (known after apply)
          + name             = "askeladden-staging"
          + resource_version = (known after apply)
          + uid              = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly
these actions if you run "terraform apply" now.
```

We can see that Terraform is going to create a Kubernetes namespace with the given parameters since it is marked with a `+` sign, neat! There's also a note at the end stating that Terraform cannot guarantee that these actions are the ones that will be performed when I run `terraform apply`, and that is completely true. In the time between me running `terraform plan` and `terraform apply`, some other developer in my imaginary team might have applied other Terraform changes that clash with mine. It is a risk I am willing to take. Funnily enough, running `terraform apply` produces the output of `terraform plan`:

```plaintext
Terraform used the selected providers to generate the following execution plan. Resource actions are
indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # kubernetes_namespace.askeladden-staging will be created
  + resource "kubernetes_namespace" "askeladden-staging" {
      + id = (known after apply)

      + metadata {
          + generation       = (known after apply)
          + name             = "askeladden-staging"
          + resource_version = (known after apply)
          + uid              = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

And prompts me to write `yes`, which is precisely what I'll do, after thoroughly verifying that the Terraform output matches what i want to create.

```plaintext
kubernetes_namespace.askeladden-staging: Creating...
kubernetes_namespace.askeladden-staging: Creation complete after 0s [id=askeladden-staging]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Great, it seems Terraform performed exactly what we expected it to. Just to be completely sure that the namespace was created, let's run `mk get namespaces` once again:

```plaintext
NAME                 STATUS   AGE
kube-system          Active   15d
kube-public          Active   15d
kube-node-lease      Active   15d
default              Active   15d
askeladden-dev       Active   14d
container-registry   Active   14d
terraform            Active   14m
askeladden-staging   Active   60s
```

And the future home for our web server is created!

## Re-deploying the web server in the newly created namespace

Note that I've created a namespace called `askeladden-staging` and we have a namespace called `askeladden-dev`. For reasons that will become clear in future posts, I am now deprecating the `askeladden-dev` namespace, meaning that I have to deploy the web server into `askeladden-staging`, check that it is working, and then delete the `askeladden-dev` namespace along with the Kubernetes resources in it.

For consistency, we need to change two things in our Helm chart. The first is the `deploymentLabels`, in `services/askeladden/chart/values.yaml`. We currently have them set as: 

```yaml
deploymentLabels:
  env: dev
```

And we want to change them to:

```yaml
deploymentLabels:
  env: staging
```

So that it matches the suffix of our namespace. Secondly, we still have our app deployed and exposed on port `30008`, which means that, if we attempt to deploy our new app, we will get an error message stating that this port is already in use. As such, we have to change the `nodePort` field, under `service`, to be something else. In this case, we will change it to `30009`.

After these changes, we can deploy our app to the `askeladden-staging` namespace using:

```plaintext
microk8s helm upgrade askeladden services/askeladden/chart/askeladden --namespace="askeladden-staging"
```

It's impressive how much easier this is to perform with Helm than with plain Kubernetes. Now, if we navigate to `http://localhost:30009`, we should the see the Monty Python quote we've been using:

```plaintext
Strange women lying in ponds distributing swords is no basis for a system of government.
```

Indeed it is, Monty Python, indeed it is. We can list our Helm releases for the `askeladden-staging` namespace using:
```plaintext
microk8s helm list --namespace="askeladden-staging"
```

```plaintext
NAME      	NAMESPACE         	REVISION	UPDATED                                 	STATUS  	CHART           	APP VERSION
askeladden	askeladden-staging	2       	2023-03-29 23:17:48.168348961 +0100 WEST	deployed	askeladden-0.0.1	0.0.3
```

Which just confirms what we already knew: our web server has been deployed successfully. However, if we run the above command for the `askeladden-dev` namespace:

```plaintext
NAME      	NAMESPACE     	REVISION	UPDATED                                 	STATUS    CHART           	APP VERSION
askeladden	askeladden-dev	1       	2023-03-27 11:03:41.930152552 +0100 WEST	deployed  askeladden-0.0.1	0.0.3 
```

We see that our web server still exists in the other namespace. Helm is also useful in case we want to remove delete old deployments, we simply have to run:

```plaintext
microk8s helm uninstall askeladden --namespace="askeladden-dev"
```

Which outputs the message: `release "askeladden" uninstalled`. We can verify this operation with the `mk get all` command, which tells us exactly what we wanted to hear:

```plaintext
No resources found in askeladden-dev namespace.
```

I almost forgot, we have our context pointing to the `askeladden-dev` namespace. Let us change that before we delete the namespace so that it points to `askeladden-staging`. First, we create a new context with

```plaintext
mk config set-context askeladden-staging --namespace=askeladden-staging --cluster=microk8s-cluster --user=admin
```

Just like we have done in the first post of these adventures. We have created the new context, so now we have to use it, which is easily achieved with:

```plaintext
mk config use-context askeladden-staging
```

If we now type `mk config current-context`, we get `askeladden-staging` as our output. We are going to delete the `askeladden-dev` so there is no point in keeping the `askeladden-dev` context around. We delete it through:

```plaintext
mk config delete-context askeladden-dev
```

Finally, we will delete our namespace using:

```plaintext
mk delete namespace askeladden-dev
```

We can delete the `infrastructure/k8s/namespace.yaml` file that we had since it is no longer required.

And we are done! We have successfully used Terraform to create a new Kubernetes namespace in our `microk8s` cluster and migrated our web server to it using Helm!
