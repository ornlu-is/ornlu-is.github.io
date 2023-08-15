# Kubernetes deploy job failed but the service was deployed


A while back, I was investigating a bug where my deployment job had failed, but the service had been deployed. At first I thought this was weird, afterwards I thought this was extremely concerning: had this happened before and I was just noticing now by accident? Since I did not want to have failed deployment jobs actually deploying my services, I took a closer look at this issue.

## The situation

What happened was pretty simple. I updated the Kubernetes manifest files for my service, opened the PR, got it reviewed and approved, the CI builds that ran on my PR were all successful, life was good. So I merged the PR and, after a few minutes, I checked the version of the running service in the staging environment and saw that it was the most recent version. Everything seemed fine (*little did he know that everything was not fine*).

I went about my life and opened a release PR, got it approved, CI builds looked good so I proceeded with the merge and consequent deployment. After a few minutes of celebrating my victory over yet-another-JIRA-ticket, I get an automated message on the chat: "Production deployment job failed". I had just snatched defeat from the jaws of victory.

The first thing I did after getting that message, was to check what was the version of the service running in production, and, to my surprise, it was the newer version.

## What actually happened

I had to dig deeper into this, meaning I had to understand how the deployment job worked. The actual deployment job pretty much only did one thing: it ran `kubectl apply` on the manifests from my repository. And this was the command that caused the job to fail. So the underlying action of the deployment job failed, the deployment job itself failed, but the deployment happened. 

Just like in the *Inception* movie, "we need to go deeper". There was only one suspect: `kubectl apply`. For my Kubernetes resources, I had several files in my repository that were combined into a single manifest file and `kubectl apply` was called to apply them all. However, one of the resources specified in the Kubernetes resource failed to be created. And I assumed that `kubectl` rolled back on the other resource it created since it failed to create all of them. This was my mistake: `kubectl apply` does not undo its changes if one of the resources failed to be created.

## How could this have been prevented

Ideally, the deployment job would begin by performing `kubectl apply`, but with the `--dry-run` flag set. This flag can take two main values: `client` or `server`. If used with the former, the `kubectl` tool will attempt to validate the manifest without actually performing any request to the Kubernetes cluster. On the other hand, if used with the `server` value, then an actual request is performed to the Kubernetes cluster, which will attempt to validate it without creating any of the specified resources. 

However, operating under the constraints of different parts of software/infrastructure being owned by different teams, means that not all can be changed. Nonetheless, we can have the next best thing. Notice that I only got a chat alert message when the production deployment job failed. What about the staging deployment job? Had it failed too? Turns out, it had, but I did not get any chat message for a failed staging deployment because we hadn't configured such alerts for our staging environment.

It is a sub-optimal solution, but I can't force other teams to change their stuff, I can only shrug and walk it off.

