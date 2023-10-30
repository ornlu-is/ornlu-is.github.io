---
title: "Adventures in Overengineering 5: Monitoring Raspberry Pi Machines with Prometheus and Grafana"
categories: ["Adventures in Overengineering"]
description: "Installing and configuring Prometheus and Grafana to monitor Raspberry Pi machines."
date: 2023-10-30T10:46:29Z
draft: false
---

With Node Exporter installed in each of the Raspberry Pi machines, I can now start scraping these metrics and actually using them to build a dashboard to monitor my machines. And this is also the step in which I create a GitHub repository where all scripts and configuration files used are stored, just in case my laptop decides to perish a second time and I have to go through this all over again (RIP my old SSD).

{{< admonition type=tip title="Other posts in this series" open=true >}}
* [Adventures in Overengineering 1: Inventory](https://ornlu-is.github.io/overengineering_1/)
* [Adventures in Overengineering 2: Installing an Operating System](https://ornlu-is.github.io/overengineering_2/)
* [Adventures in Overengineering 3: Installing Salt to manage Raspberry Pi machines](https://ornlu-is.github.io/overengineering_3/)
* [Adventures in Overengineering 4: Installing Node Exporter via Salt](https://ornlu-is.github.io/overengineering_4/)
{{< /admonition >}}

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/olympus
{{< /admonition >}}

## Installing and configuring Prometheus

Node Exporter functions by periodically publishing metrics to an HTTP endpoint that is queriable by other services. This means that all we need is something that periodically scrapes and stores these metrics and then we need another something to actually visualize the metrics we've stored. Once again, Grafana labs comes to the rescue with Prometheus, an open-source monitoring system and time series database, and with Grafana, an open-source analytics and interactive visualization web application. 

After some tinkering and online research, I came up with the following script `bash` script for installing Prometheus:

```bash
#!/bin/bash

# Download and extract most recent Prometheus version
VERSION=$(curl https://raw.githubusercontent.com/prometheus/prometheus/master/VERSION)
wget https://github.com/prometheus/prometheus/releases/download/v${VERSION}/prometheus-${VERSION}.linux-amd64.tar.gz
tar xvzf prometheus-${VERSION}.linux-amd64.tar.gz

# Create directories and files for Prometheus configuration
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo touch /etc/prometheus/prometheus.yaml

# Copy the Prometheus binary to the appropriate location
sudo cp prometheus-${VERSION}.linux-amd64/prometheus /usr/local/bin/

# Write the configuration file
cat ./infrastructure/prometheus/prometheus.yaml | sudo tee /etc/prometheus/prometheus.yaml

# Write the systemd unit file for Prometheus
cat ./install-scripts/prometheus/prometheus.service | sudo tee /etc/systemd/system/prometheus.service

# Reload systemd, enable and start Prometheus
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

# Cleanup installation files
rm prometheus-${VERSION}.linux-amd64.tar.gz
rm -rf prometheus-${VERSION}.linux-amd64
```

The steps it performs are fairly simple:
* Download and extract the Prometheus binary;
* Create the `/etc/prometheus` directory and the `prometheus.yaml` file inside it, which will hold all the basic configuration for my Prometheus instance;
* Create the `/var/lib/prometheus` directory, which will contain the actual Prometheus time series database;
* Copy the Prometheus binary to the appropriate location so that it can be executed;
* Write the contents of the Prometheus configuration file;
* Write the `systemd` unit file for Prometheus;
* Reload `systemd` for it to load the Prometheus unit file, enable the Prometheus service (for it to run on startup), and start the service;
* Cleanup the installation files.

My Prometheus configuration file is going to be very simple. All I want is for Prometheus to scrape metrics every 5 seconds, and I also need to tell Prometheus whose metrics to scrape. In other words, I have to point Prometheus to the `/metrics` HTTP endpoint of each Raspberry Pi. This results in the following `prometheus.yaml` file:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'zeus'
    metrics_path: /metrics
    static_configs:
      - targets: ['192.168.1.75:9100']
  
  - job_name: 'poseidon'
    metrics_path: /metrics
    static_configs:
      - targets: ['192.168.1.76:9100']
  
  - job_name: 'ares'
    metrics_path: /metrics
    static_configs:
      - targets: ['192.168.1.77:9100']
```

You might be wondering why I am referring to the machines by their IP address and not their name in my local network. This is because when I used, for example, `zeus.local`, I got an error from Prometheus with a message stating "server misbehaving". After some online digging I figured that it might be some kind of issue with my local DNS resolution, so I just switched to the local IP address, and the error was gone. I plan to eventually do some more investigation on this issue but, for now, this solution is good enough.

In my install script there is also reference to a `systemd` unit file for Prometheus. The unit file I came up is the following:

```plaintext
[Unit]
Description=Prometheus
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yaml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --storage.tsdb.retention.time=1d

[Install]
WantedBy=multi-user.target
```

There is nothing out of ordinary with this unit file. The only thing in it that is worth mentioning is the `--storage.tsdb.retention` parameter for when we are starting the Prometheus service. This parameter tells Prometheus how long it should store the metric values that it scrapes. Since I only turn on the Raspberry Pi machines when I'm using them, I have little to no use for metrics that are older than a day (and even that might be too much).

Finally, we can check if Prometheus was installed properly by navigating to `localhost:9090`, where we should see the Prometheus UI. Additionally, we also need to check if it is scraping metrics correctly. We can do this by navigating to `http://localhost:9090/targets` and checking if there is some message in the error column. If not, this whole process was successful!

## Installing Grafana with a prebuilt dashboard

The Grafana installation is a bit more straightforward. Taking the official install documentation and placing it a `bash` script produces the following outcome:

```bash
#!/bin/bash

# Install the prerequisite packages
sudo apt install -y apt-transport-https software-properties-common wget

# Import the GPG key
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add the Grafana stable release repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Update the list of available packages
sudo apt update

# Install Grafana
sudo apt install grafana

# Configure data sources
sudo touch /etc/grafana/provisioning/datasources/datasources.yaml
cat ./infrastructure/grafana/provisioning/datasources/datasources.yaml | sudo tee /etc/grafana/provisioning/datasources/datasources.yaml

# Configure pre-built dashboard
sudo mkdir /var/lib/grafana/dashboards
cat ./infrastructure/grafana/dashboards/dashboard.yaml | sudo tee /etc/grafana/provisioning/dashboards/dashboard.yaml
cat ./infrastructure/grafana/dashboards/nodes.json | sudo tee /var/lib/grafana/dashboards/nodes.json

# Reload systemd, enable and start Grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Running this script will install Grafana as well as configure it to run on startup with `systemd`. It also configures our Prometheus instance as the default data source for Grafana using the `datasources.yaml` file:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
```

Additionaly, this script also creates a fully functional Grafana dashboard to monitor my Raspberry Pi machines! I found this dashboard in a [GitHub repository](https://github.com/rfmoz/grafana-dashboards/blob/master/prometheus/node-exporter-full.json) that is filled with panels with just about anything one might want to visualize that is exposed by Node Exporter.

Since Grafana stores dashboards as JSON files, all we have to do is download the dashboard's JSON representation from the repository, place the following configuration in the appropriate directory:

```yaml
apiVersion: 1

providers:
  - name: 'Prometheus'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

And then place the dashboard's JSON file in the path specified above. My script already does all this for me, which is great! After running the script, you can navigate to `localhost:3000` where you'll be prompted for your Grafana credentials. Both the username and the password are `admin`. After that, you'll be prompted to change your password, which you should probably do. Now, Grafana is up and running and we have a dashboard that looks like this:

{{< figure src="/images/overengineering_5/grafana_dashboard.png" title="Node Exporter Grafana dashboard" >}}

Pretty neat!

## Convenience is king

I'm not very good at memorising bash commands and where every single file is. Fortunately, I do not have to be, I can just create a Makefile that abbreviates some of the commands I usually run. Currently, my project is structured as follows:

```plaintext
.
├── infrastructure
│   ├── grafana
│   │   ├── dashboards
│   │   │   ├── dashboard.yaml
│   │   │   └── nodes.json
│   │   └── provisioning
│   │       └── datasources
│   │           └── datasources.yaml
│   └── prometheus
│       └── prometheus.yaml
├── install-scripts
│   ├── grafana
│   │   └── grafana.sh
│   ├── prometheus
│   │   ├── prometheus.service
│   │   └── prometheus.sh
│   └── README.md
├── Makefile
└── README.md
```

This Makefile exists in the top level directory and allows me to apply changes to Prometheus or Grafana configs seemlessly as well as run a Salt highstate:

```Makefile
salt-highstate:
	sudo salt '*' state.highstate

install-prometheus:
	sudo sh ./install-scripts/prometheus/prometheus.sh

install-grafana:
	sudo sh ./install-scripts/grafana/grafana.sh

apply-grafana:
	sudo mkdir /var/lib/grafana/dashboards
	sudo touch /etc/grafana/provisioning/datasources/datasources.yaml /etc/grafana/provisioning/dashboards/dashboard.yaml
	cat ./infrastructure/grafana/provisioning/datasources/datasources.yaml | sudo tee /etc/grafana/provisioning/datasources/datasources.yaml
	cat ./infrastructure/grafana/dashboards/dashboard.yaml | sudo tee /etc/grafana/provisioning/dashboards/dashboard.yaml
	cat ./infrastructure/grafana/dashboards/nodes.json | sudo tee /var/lib/grafana/dashboards/nodes.json
	sudo systemctl restart grafana-server

apply-prometheus:
	sudo touch /etc/prometheus/prometheus.yaml
	cat ./infrastructure/prometheus/prometheus.yaml | sudo tee /etc/prometheus/prometheus.yaml
	sudo systemctl restart prometheus
```

Fundamentally, the most complex targets in this Makefile are selected snippets of the install scripts that allow me to change configs on the fly.

## Next steps

While having a pre-built dashboard saved me a lot of time and effort, this dashboard has a bit too much information. What I really want is something more like a wall TV, where I can look at a single screen (without scrolling) and receive all the information I require to ascertain the status of my devices. Additionally, I also want to include the Salt files in my repository, so I'll probably be exploring that next.
