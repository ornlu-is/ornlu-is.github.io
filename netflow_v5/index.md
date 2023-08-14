# Understanding Netflow v5


NetFlow is a protocol developed by Cisco that has had several iterations over the years. This network protocol's main focus is to collect IP traffic information and export it for monitoring purposes. A router configured to collect NetFlow will aggregate data packets into what is known as a flow, which is basically a summary of the traffic passing through the device in a given time span. Of the multiple iterations of the NetFlow protocol that Cisco has produced, two have cemented themselves as the most commonly used protocols: NetFlow v5 and NetFlow v9. Today, I'll focus on the NetFlow v5 description.

## The NetFlow v5 datagram

When a device is configured to collect and send (which is usually performed via UDP) flow data according to the NetFlow v5 protocol, that device will periodically send NetFlow v5 datagrams which as composed of a header and a variable number of flows, as depicted in the image below:

{{< figure src="/images/netflow_v5/netflow_v5_datagram.png" width=0.5 height=0.5 title="NetFlow v5 datagram" >}}

Meaning that for each UDP packet you receive from your collecting device, you might get several flows worth of information.

## Header datagram specification

The NetFlow v5 header is composed of 24 bytes, with each byte corresponding to the following information:

| Bytes | Field | Description |
| :--- | :--- | :--- |
| 0-1 | `version` | NetFlow export format version number |
| 2-3 | `count` | Number of flows exported in this packet |
| 4-7 | `SysUptime` | Current time in milliseconds since the export device booted |
| 8-11 | `unix_secs` | Current count of seconds since 0000 UTC 1970 |
| 12-15 | `unix_nsecs` | Residual nanoseconds since 0000 UTC 1970  |
| 16-19 | `flow_sequence` | Sequence counter of total flows seen |
| 20 | `engine_type` | Type of flow-switching engine |
| 21 | `engine_id` | Slot number of the flow-switching engine |
| 22-23 | `sampling_interval` | First two bits hold the sampling mode; remaining 14 bits hold value of sampling interval |

While some of these fields may seem obvious, other require a bit more clarification. Obviously, since we are dealing with NetFlow v5, the `version` field will always correspond to the number five. The `count` field, while it is made out of two bytes, which might lead some to deduce that it is a number between zero and 65,536, this is not the case. In fact, the maximum number of flow in each NetFlow v5 datagram is 30.

The `engine_type` field provides an identifier of how the collecting device handles flow-switching. Since NetFlow v5 is a proprietary protocol from Cisco, this is mapped into a Cisco-specific flow-switching technology. For monitoring purposes, this field requires a great deal of understanding of the underlying Cisco technologies and how your devices are configured to extract any type of meaningful information. For that reason, flow monitoring services tend to ignore this field. The `engine_id` field is a configurable identifier for the flow collector.

Finally, the last field that requires a bit more understanding is the `sampling_interval`. A NetFlow collector doesn't necessarily export flows that contain absolutely exact information on the observed traffic. For devices that handle large loads, this is usually not computationally efficient. As such, NetFlow collectors can be configured to employ sampling, where the flows will be calculated based on a reduced sample of the network's IP traffic. Broadly speaking, there are two main sampling modes:
* Sampled NetFlow - flows are calculated based on x-th packet, *i.e.*, if this is set to an interval of 10, then the flows reported contain are calculated based on the 1st, 10th, 20th, etc., packets. This sampling method is usually advised against, since it might hide periodic traffic patterns;
* Random Sampled NetFlow - flows are calculated based on random draws of incoming packets, much like random sampling. It provides on average consistency on the number of packets used to calculate the flows and is overall more statistically accurate than the Sampled NetFlow scheme.

## Flow datagram specification

The flow datagram is a fair bit larger than the header, with each flow containing 48 bytes of data organised as follows:

| Bytes | Field | Description |
| :--- | :--- | :--- |
| 0-3 | `srcaddr` | Source IP address |
| 4-7 | `dstaddr` | Destination IP address |
| 8-11 | `nexthop` | IP address of next hop router |
| 12-13 | `input` | SNMP index of input interface |
| 14-15 | `output` | SNMP index of output interface |
| 16-19 | `dPkts` | Packets in the flow |
| 20-23 | `dOctets` | Total number of Layer 3 bytes in the packets of the flow |
| 24-27 | `First` | SysUptime at start of flow |
| 28-31 | `Last` | SysUptime at the time the last packet of the flow was received |
| 32-33 | `srcport` | TCP/UDP source port number or equivalent |
| 34-35 | `dstport` | TCP/UDP destination port number or equivalent |
| 36 | `pad1` | Unused (zero) bytes |
| 37 | `tcp_flags` | Cumulative OR of TCP flags |
| 38 | `prot` | IP protocol type |
| 39 | `tos` | IP type of service (ToS) |
| 40-41 | `src_as` | Autonomous system number of the source, either origin or peer |
| 42-43 | `dst_as` | Autonomous system number of the destination, either origin or peer |
| 44 | `src_mask` | Source address prefix mask bits |
| 45 | `dst_mask` | Destination address prefix mask bits |
| 46-47 | `pad2` | Unused (zero) bytes |

Let us clarify some of these fields. As you might've already noticed, the `srcaddr` and `dstaddr` fields, which correspond, respectively, to the source and destination IP addresses of the traffic, are only four bytes long. This means that NetFlow v5 is limited uniquely to IPv4 traffic, which is a testament to the fact that it was released a long time ago.

The `tcp_flags` field is a single byte used to represent TCP flags via a cumulative OR. If you are not familiar with TCP, it is sufficient to understand that each TCP datagram contains information on the TCP flags, which describe what operation or outcome was performed/observed. There are eight possible flags, which makes it fairly straight-forward to one-hot encode these. Then, to get a single byte representation of all possible combinations, we simply take the cumulative OR function of their bytes.

One of the most important fields in NetFlow v5 is the `prot` field, which pertains to the possible protocol. While it might seem confusing that the protocol is a single number, we can easily get the corresponding name via the mapping published by the Internet Assigned Numbers Authority (IANA).

## Some considerations on using NetFlow v5 for analytics

There are two major considerations to take when building analytics with NetFlow v5. The first one is to understand that a flow is not an instant snapshot of your traffic. It is a summary of your traffic, collected over a given period of time, which is where the `First` and `Last` fields come in. These allow you to know for how long was data collected to calculate the given flow. However, you get absolutely no information on how the traffic was distributed inside that time range. This means that you have to make an assumption. A sensible one is to assume that, for short flow collection periods, traffic was more or less constant and, as such, you can uniformly distribute it across the time granularity you are using. There is nothing stopping you from considering more complicated schemes, but know that there is very little to gain from not taking the uniformly distributed approach.

The last consideration pertains to the `sampling_interval`. This value is basically the inverse of a sampling rate applied by the flow collector, and it is important because you cannot calculate the amount of bytes or packets in a flow without it. If a flow has `dOctets = 24` and the header has `sampling_interval=10`, then the number of bytes you have to report is 240, because your flow was calculated using only 10% of your network's traffic. You will obviously incur in an approximation error with this, but your scope of operation is after the flow is collected, so you have no control over its sampling strategy. And even if you had, keep in mind that flow exports have to be produced at very high speeds and with minimal computational effort, which immediately discards the vast majority, if not all, sampling schemes that are not simple random sampling.

