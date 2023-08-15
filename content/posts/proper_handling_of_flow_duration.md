---
title: "Proper handling of flow duration in NetFlow"
date: 2023-08-15T00:30:24+01:00
draft: true
description: "How to calculate the time a flow collection started and ended, and how to spread traffic over that duration"
categories: ["Computer Networks"]
---

Time in NetFlow v5

Time in NetFlow v9. Note that NetFlow v9 is compatible with IPFIX, and thus time calculations may vary depending on the configuration used by your vendor. This applies only to the standard use case

Bring it all together because essentially you want to calculate the time flow collection started and the time it ended.

Explain how the way you split traffic is intrinsically tied to how you are displaying/analysing your data.
