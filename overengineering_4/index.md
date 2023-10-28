# Adventures in Overengineering 4: Installing Node Exporter via Salt


I have a bunch of physical machines and it is important to keep track of how healthy these machines are. More specifically, I want to keep track of some key metrics such as how much CPU or memory is being used and disk space. Additionally, I also want to keep track of the temperature of these machines because I do not want to accidentally burn down my house (that would really destroy the budget for this project).

## Collecting metrics with Node Exporter

There is this fantastic piece of software by Grafana Labs called Prometheus Node Exporter. What it does is very simple: it is a single static binary that exposes a bunch of hardware and kernel related metrics. This is precisely what I want to monitor my Raspberry Pi machines! If we look at [Node Exporter's official documentation](https://prometheus.io/docs/guides/node-exporter/), we can see that it is fairly easy to install directly: just download, extract and run the binary. However, we're here to overengineer stuff, so, instead of installing it directly, I'm going to install it via Salt, which I have previously configured in the [previous post](https://ornlu-is.github.io/overengineering_3/) of this series.

## Installing Node Exporter with Salt

This is also the first piece of software I'm going to install on my machines with Salt, which is exciting! On my laptop, *i.e.*, the Salt master, I under the `/srv/` directory, I create a `salt/` directory with the following contents: 

```plaintext
.
├── node-exporter
│   ├── files
│   │   └── node_exporter.service
│   └── init.sls
└── top.sls
```

In other words, I created a Salt highstate, which is nothing more than a way to let Salt know which formulas should be applied to which minions. This targeting is achieved with a top file (`top.sls`). Since I have three minions named `zeus`, `ares`, and `poseidon`, my top file looks as follows:

```yaml
base:
  '*':
    - node-exporter
  zeus:
    - node-exporter
  ares:
    - node-exporter
  poseidon:
    - node-exporter
```

Meaning that, whenever I call salt, I can choose to target any of my minions or all of them with the Kleene star. This also defined what is applied to each target which, in this case, is the `node-exporter` formula for all targets. This formula is defined under `node-exporter/init.sls` and is responsible for installing and running Node Exporter. Let's take a closer look at it:

```yaml
{% if not salt['file.file_exists']('/usr/local/bin/node_exporter') %}

retrieve_node_exporter:
  cmd.run:
    - name: wget -O /tmp/node_exporter.tar.gz https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-arm64.tar.gz

extract_node_exporter:
  archive.extracted:
    - name: /tmp
    - enforce_toplevel: false
    - source: /tmp/node_exporter.tar.gz
    - archive_format: tar
    - user: root
    - group: root

move_node_exporter:
  file.rename:
    - name: /usr/local/bin/node_exporter
    - source: /tmp/node_exporter-1.6.1.linux-arm64/node_exporter

delete_node_exporter_dir:
  file.absent:
    - name: /tmp/node_exporter-1.6.1.linux-arm64

delete_node_exporter_files:
  file.absent:
    - name: /tmp/node_exporter.tar.gz

{% endif %}

/etc/systemd/system/node_exporter.service:
  file.managed:
    - source: salt://node-exporter/files/node_exporter.service
    - user: root
    - group: root
    - mode: 0644

node_exporter_service_enable:
  cmd.run:
    - name: systemctl daemon-reload
    - watch:
      - file: /etc/systemd/system/node_exporter.service

node_exporter_service:
  service.running:
    - name: node_exporter
    - enable: True
```


The first part, `retrieve_node_exporter`, simply downloads the `.tar.gz` file from the compressed Node Exporter binary into the `tmp/` directory. Since it is compressed, we have to then, you've guessed it, decompress the file using `extract_node_exporter`. The extracted binary is now inside the `tmp/` directory so we move it to `usr/local/bin/` using `move_node_exporter`. Since we have already extracted and moved the binary, we can delete the directory created during the extraction process and `.tar.gz` file, respectively, with `delete_node_exporter_dir` and `delete_node_exporter_files`. Note that all these states are enclosed in an if statement, meaning that these are only executed in case `/usr/local/bin/node_exporter` does not exist. In other words, these commands are only executed if Node Exporter is not installed.

Now we need a way to actually run the service. Since I'm running Ubuntu on my Raspberry Pi machines, all the processes running on them are managed by the init process known as `systemd`, which means that I need a way to let `systemd` know that Node Exporter exists and I also need a way for `systemd` to run Node Exporter whenever the system starts up. For that purpose, I have defined a very simple `systemd` unit file called `node_exporter.service` under `/srv/salt/node-exporter/files/`, which looks like this:

```plaintext
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter
```

All it does is say that there is a service called `node_exporter` that is started after the system's network is up. We then tell Salt to add this unit file as a managed file to `systemd` and that the service this file refers to must be ran at high priority, which is achieved by having it placed under the `/etc/systemd/system/` directory. Then, we reload `systemd` to get these unit files in memory with `node_exporter_service_enable` which simply runs `systemctl daemon-reload`. Lastly, we run the service with `node_exporter_service`.

Finally, it's time to push the button, by calling:

```
sudo salt '*' salt.highstate
```

And let Salt work its magic!

## Verifying the result

We can verify this result in a fairly straightforward manner. The Node Exporter essentially exposes an HTTP endpoint that publishes the metrics collected by it. This HTTP endpoint is running, by default, on port 9100 and the path to it is just `/metrics`. Since all my Raspberry Pi machines are running on my local network, I can verify if Node Exporter is running properly by simple `curl`ing this endpoint on each of the hosts:

```
curl poseidon.local:9100/metrics
curl ares.local:9100/metrics
curl zeus.local:9100/metrics
```

If you get a wall of text with a bunch of metric names and numbers on your terminal for each of these commands then congratulations, Node Exporter is up and running!

## Next steps

Ideally, I would have these files in a repository somewhere. This can be achieved by specifying a different backend for Salt and using `gitfs` to mount the `/srv/salt/` directory from a `git` remote repository. I haven't done that yet, but it is part of my future plans. Mainly because the next post will be about actually scraping the metrics collected by the several Node Exporter instances and visualizing them in a single Grafana instance.

