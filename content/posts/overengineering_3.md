---
title: "Adventures in Overengineering 3: Installing Salt to manage Raspberry Pi machines"
date: 2023-10-10T19:36:10+01:00
categories: ["Adventures in Overengineering"]
description: "How to install Salt on Raspberry Pi machines."
draft: false
---

I have three machines with Ubuntu installs up and running but there is one thing that I really want to avoid: having to `ssh` into them to install software, since the effort of installing a given package/software is always multiplied by three. As such, I thought this was an appropriate time to refresh my Salt knowledge, which is an event-driven framework/automation tool whose name probably derives from the high blood pressure it induces whenever it is used to perform changes on production environment machines.

Just a reminder that all this is still for the same initial goal: over-engineering a basic Golang web server.

## Installing `salt-master`

I do not want to use any of the Raspberry Pis as a `salt-master`, so I'll be using my laptop for it. First, we import the Salt Project repository key using:

```plaintext
sudo curl -fsSL -o /etc/apt/keyrings/salt-archive-keyring-2023.gpg https://repo.saltproject.io/salt/py3/ubuntu/22.04/amd64/SALT-PROJECT-GPG-PUBKEY-2023.gpg
```

And then we create the `apt` sources list file using:

```plaintext
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.gpg arch=amd64] https://repo.saltproject.io/salt/py3/ubuntu/22.04/amd64/latest jammy main" | sudo tee /etc/apt/sources.list.d/salt.list
```

As usual with installing something new, we first update our packages with:

```plaintext
sudo apt-get update
```

To install the `salt-master`, we simply run:

```plaintext
sudo apt-get install salt-master
```

Since I do not want to have to start the `salt-master` service every time I want to do any Salt related operations, I'm going to have it start when my system boots up with:

```plaintext
sudo systemctl enable salt-master && sudo systemctl start salt-master
```

Finally, we can start the service with:

```plaintext
sudo systemctl start salt-master
```

## Configuring `salt-master`

The default configuration that comes with the install can be found in `/etc/salt/master`. We will not, for the time being deviate from it.



## Installing `salt-minion` on the Raspberry Pis

Now we need to install Salt in the minion nodes, *i.e.*, on the Raspberry Pi machines. The process is fairly similar to installing `salt-master`. I'm going to connect to `zeus` (one of my Raspberry Pi) via `ssh` and import the Salt Project repository key as before:

```plaintext
sudo curl -fsSL -o /etc/apt/keyrings/salt-archive-keyring-2023.gpg https://repo.saltproject.io/salt/py3/ubuntu/22.04/arm64/SALT-PROJECT-GPG-PUBKEY-2023.gpg
```

Followed by creating the `apt` sources list file with:

```plaintext
echo "deb [signed-by=/etc/apt/keyrings/salt-archive-keyring-2023.gpg arch=arm64] https://repo.saltproject.io/salt/py3/ubuntu/22.04/arm64/latest jammy main" | sudo tee /etc/apt/sources.list.d/salt.list
```

Note that, if you are copying the commands from the official documentation for ARM64 architectures, the docs have a typo where the install commands for ARM64 are the same as those for AMD64. To get around this, all you have to do is switch all occurrences of `amd64` to `arm64`. If you do not do this, you'll get the following error when you attempt to install `salt-minion`:

```plaintext
N: Skipping acquire of configured file 'main/binary-amd64/Packages' as repository 'https://repo.saltproject.io/salt/py3/ubuntu/22.04/arm64/latest jammy InRelease' doesn't support architecture 'amd64'
```

Update the packages and install `salt-minion` with:

```plaintext
sudo apt-get update
sudo apt-get install salt-minion
```

Exactly the same as before, I want this to run on start up, so I'll enable it with `systemctl`, and start the service because I want to use it now:

```plaintext
sudo systemctl enable salt-minion && sudo systemctl start salt-minion
```

## Configuring `salt-minion`

In theory, the `salt-master` service should be discoverable by the name `salt`. However, when I check the status of the `salt-minion` service with `systemctl`, I see a bunch of messages stating:

```
salt DNS lookup or connection check of 'salt' failed
```

The way I found to get around this is fairly simple. In `zeus`, I navigated to `/etc/salt/minion.d` and created as `master.conf` file with the following contents:

```yaml
master: <MASTER_IP>
```

Restarted the `salt-minion` service with:

```
sudo systemctl restart salt-minion
```

And *voilà*, the error message is gone!

## Accepting keys

Alas, an error has been replaced by another. It seems that my Salt minion is having trouble connecting to the master due to authentication issues. This is because of how Salt operates: minions and masters generate RSA keys on their first start up and these are then used for private key infrastructure (PKI) based authentication. On the Salt master, there is a client running called `salt-key` that allows for managing minion authentication.

If we run:

```plaintext
sudo salt-key
```

We get the following output:

```plaintext
Accepted Keys:
Denied Keys:
Unaccepted Keys:
zeus
Rejected Keys:
```

Now it has become extremely obvious why I was still getting an error: I needed my Salt master to accept `zeus` key! This boils down to a single command:

```
sudo salt-key -a zeus
```

Wonderful, I can now finally proceed to verifying the installation.

## Verification

We'll follow the usual process of verifying installations: attempting to get the version of the install. In Salt, we can achieve this with:

```plaintext
sudo salt '*' test.version
```

Which outputs the following:

```yaml
zeus:
    3006.3
```

Meaning that `zeus` is running version 3006.3 and that this whole process was successful! Great, now rinse and repeat this process for `poseidon` and `ares`, and I now have three Salt minions up and running!

## Next steps

I can now install stuff on my Raspberry Pi machines without having to `ssh` and do it manually, which is fantastic but I am still not ready to install and configure my Kubernetes cluster. Before that, I need a way to monitor the health of machines from my laptop so the next post will probably be about Grafana (and related software).
