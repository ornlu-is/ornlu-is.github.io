# Adventures in Overengineering 2: Installing an Operating System


Now that I have some hardware available, I need to install an operating system on these machines so that I can then install some software that will enable me to actually do something with these things. Since I do not want to have to connect a monitor to these machines every time I need to use them, I also have to set up some type of remote access.

{{< admonition type=tip title="Other posts in this series" open=true >}}
* [Adventures in Overengineering 1: Inventory](https://ornlu-is.github.io/overengineering_1/)
* [Adventures in Overengineering 3: Installing Salt to manage Raspberry Pi machines](https://ornlu-is.github.io/overengineering_3/)
* [Adventures in Overengineering 4: Installing Node Exporter via Salt](https://ornlu-is.github.io/overengineering_4/)
* [Adventures in Overengineering 5: Monitoring Raspberry Pi Machines with Prometheus and Grafana](https://ornlu-is.github.io/overengineering_5/)
{{< /admonition >}}

## Preparing the microSD cards

This part is extremely simple. On my Windows machine, I downloaded the Raspberry Pi imager, a software that burns Raspberry Pi compatible OS images onto storage devices. Flashing the OS is straightforward with this software: just plug in the storage device, select the operating system in the imager (in my case, I chose 64-bit Ubuntu Server 23.04), name the computers, enable `ssh` and configure Wi-Fi connection in the advanced configurations, and wait until the imager finished burning the OS image. Rinse and repeat for all three microSD cards, and then just plug them onto the Raspberry Pi machines.

## Checking the OS installs

During my childhood, I developed an obsession on Greek mythology thanks to the popular RTS game called Age of Mythology. As such, I decided to name my Raspberry Pi machines after Greek gods: Zeus, Poseidon, and Ares. After turning on the machines, these should be discoverable in my home network. To check if these machines are working properly, I'll make use of the `ping` Linux utility.

## ICMP

This is an excellent opportunity to review the ICMP protocol. The Internet Control Message Protocol is a network layer protocol used by network devices to diagnose network communication issues. It is connectionless in the sense that it is not required to have an open connection between two devices for one of them to send an ICMP message. Broadly speaking, an ICMP message looks like this:

{{< figure src="/images/overengineering_2/icmp_packet.png" title="ICMP message" >}}

The type field provides some broad categorization about the control message while the code contains additional context on what is being reported. The data section contains a copy of the IPv4 header as well as some bits of the IPv4 packet that caused the error, hence why it has variable (but limited) length.

This protocol has two main uses:
* Error reporting: when a device attempts to send data to another and something goes wrong, ICMP generates error messages to share with the sending device;
* Network diagnostics: the protocol can also be used to assert the health of network connections between devices;

## Pinging the machines

So, armed with this basic knowledge of ICMP, we can now look closely at the `ping` Linux utility. `man ping` reveals that all this does is send a ICMP ECHO_REQUEST message to the given host, so let's do that with `ping poseidon.local`, which yeilds the following result:

```plaintext
PING poseidon.local (192.168.1.76) 56(84) bytes of data.
64 bytes from 192.168.1.76: icmp_seq=1 ttl=64 time=71.8 ms
64 bytes from 192.168.1.76: icmp_seq=2 ttl=64 time=67.4 ms
64 bytes from 192.168.1.76: icmp_seq=3 ttl=64 time=18.4 ms
64 bytes from 192.168.1.76: icmp_seq=4 ttl=64 time=26.5 ms
64 bytes from 192.168.1.76: icmp_seq=5 ttl=64 time=12.2 ms
64 bytes from 192.168.1.76: icmp_seq=6 ttl=64 time=6.84 ms
64 bytes from 192.168.1.76: icmp_seq=7 ttl=64 time=8.46 ms
```

To check the remaining machines, I just have to run the same command for those hosts (`ares.local` and `poseidon.local`). And they are all working properly, yey!

## Connecting to the machines

Note that I selected the imager configurations that would enable `ssh` and configure. This means that I can use these machines from my laptop simply using my Wi-Fi connection! Let's check what we get when I try to `ssh` into one of them:

```plaintext
Welcome to Ubuntu 23.04 (GNU/Linux 6.2.0-1004-raspi aarch64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Oct  1 19:25:35 WEST 2023

  System load:  0.3               Temperature:            34.1 C
  Usage of /:   4.7% of 58.21GB   Processes:              147
  Memory usage: 2%                Users logged in:        1
  Swap usage:   0%                IPv4 address for wlan0: 192.168.1.76


100 updates can be applied immediately.
66 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


Last login: Thu Sep 28 22:22:07 2023
```

Great, we're all set to start using these machines!

## Next steps

Now that I have infrastructure, it is probably time I start thinking about how I will manage it. I have to do some research for this, so it will probably take some time until my next post.

