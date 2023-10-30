---
title: "Adventures in Overengineering 1: Inventory"
date: 2023-09-28T16:17:45+01:00
categories: ["Adventures in Overengineering"]
description: "Acquiring a bunch of hardware to take my overengineering adventure to the next level."
draft: false
---

A few months ago, I started a series of posts about my attempt to completely overengineer a simple Go web server. After a few posts, I had to archive that series. I was not pleased with the result, it wasn't... enough. Last time, I began this adventure by running a Kubernetes instance on my laptop. This time, I've gone deeper into the overengineering madness. Keep in mind that the end goal is still to deploy a very basic Golang web server.

{{< admonition type=tip title="Other posts in this series" open=true >}}
* [Adventures in Overengineering 2: Installing an Operating System](https://ornlu-is.github.io/overengineering_2/)
* [Adventures in Overengineering 3: Installing Salt to manage Raspberry Pi machines](https://ornlu-is.github.io/overengineering_3/)
* [Adventures in Overengineering 4: Installing Node Exporter via Salt](https://ornlu-is.github.io/overengineering_4/)
* [Adventures in Overengineering 5: Monitoring Raspberry Pi Machines with Prometheus and Grafana](https://ornlu-is.github.io/overengineering_5/)
{{< /admonition >}}

## Acquiring hardware

Yep, you read that right, the first step is to acquire some hardware. How much hardware? Well, I want to create a three node Kubernetes cluster. As such, I need three machines. As with any engineering project, budget is something to keep in mind as I'd rather not go broke over this. This means that the most in-budget option for me is to buy a some Raspberry Pi machines. In particular, I bought 3 Raspberry Pi 4 Model B machines with the following specs:

* Processor: Broadcom BCM2711, quad-core Cortex-A72 (ARM v8) 64-bit SoC @ 1.5GHz
* Memory: 8GB LPDDR4
* Wireless: 2.4GHz and 5.0GHz IEEE 802.11b/g/n/ac

There are obviously some more details in the specs for these machines, but these are the main details I cared about. Additionally, for a first pass at this, I'll be using microSD storage for these machines, meaning I also had to buy an adapter for my laptop to be able to write into these cards. Each Raspberry Pi also requires its own power source and, for the first time OS install, I'll also need an HDMI to micro-HDMI cable. The full shopping list can be seen below:

| Item                                | Quantity | Price/item (€) |
| ----------------------------------- | -------- | -------------- |
| Raspberry Pi Model B 1.5GHz 8GB     | 3        | 85.49          |
| Joy-IT two fan acrylic case         | 3        | 14.35          |
| Sandisk microSD 64GB                | 3        | 15.75          |
| USB-C power source 3.0A 15.3W black | 3        | 9.75           |
| USB2.0 microSD reader               | 1        | 7.68           |
| HDMI 1.4 <-> micro-HDMI cable       | 1        | 3.75           |

Which means I ended up spending a grand total of 387.45€. My wallet weeps but the thirst for overengineering must be properly quenched. Here is a badly taken picture of the whole material:

{{< figure src="/images/overengineering_1/full_material_annotated.jpg" title="Hardware for overengineering" >}}

## Assembling the hardware

Putting these things together is quite simple. To my surprise, the acrylic cases also came with heat sinks, which is a nice touch. Here is a picture of what the Raspberry Pi looks like after adding the heat sinks, which are the things inside the red circles:

{{< figure src="/images/overengineering_1/raspberry_pi_heat_sink_annotated.jpg" title="Raspberry Pi with heat sinks" >}}

To connect the fans, I had to look up what the GPIO pin layout was for this model. It seems that the outermost second pin is 5V DC power, while the pin immediately to its right is the ground pin. Thus, connecting the fans is as simple as the picture below demonstrates:

{{< figure src="/images/overengineering_1/raspberry_pi_fan_pins_annotated.jpg" title="Connecting the case fans to the Raspberry Pi" >}}

Then I attempted to place the Raspberry Pi inside the case. I say attempted because the board didn't fit inside the case. This is because the board had some imperfections around its edges. No problem, this is very easy to solve. All I had to do was very calmly file these imperfections until they were gone and the board could be placed in the case. Assembling the acrylic case is easy since it's just three pieces that are held together with magnets. The end result looks something like this:

{{< figure src="/images/overengineering_1/raspberry_pi_assembled.jpg" title="Raspberry Pi machines inside their cases" >}}

## Next steps

I have inventory. The inventory is assembled. The next step will be to install an operating system in each of the Raspberry Pi boards.
