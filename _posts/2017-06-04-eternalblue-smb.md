---
layout: post
title:  Memory forensics of Eternalblue
category: memory forensics
description: Techinal dive into the Eternalblue exploit that uses SMB v1 vulnerability.
---
![]({{site.baseurl}}/assets/img/eternal.png)


# Journey in to the SMB vulnerability that Enabled WanaCry

This post is a tale of my journey into trying to fully understand the vulnerability behind WanaCry(pt).

This post consists of the following parts.

1. Understanding the exploit method
2. Using the exploit and popping under the hood of a vulnerable machine to see what happens

## Part 1: understanding the exploit method

The exploit methodology in this post is based on two different exploit code. One
is the Metasploit plugin [[1](#sources)]
and the other one is by a Github user @worawit [[2](#sources)]. Worawit's code is very well documented and therefore it is going to be the primary source where we are going to begin. It gives us great hints about where to start this research.

### The exploit

Before Microsoft patching it, SMB version 1 was vulnerable to a buffer overflow attack [[1](#sources)]. The vulnerability is exploitable when a malformed Trans2 request is sent to the server which enables the attacker to overwrite another part of the memory [[3](#sources)]. The goal of the attacker (and how NSA did it) would be to overwrite some useful memory portion. This memory area in this attack is the buffer of another SMB connection which in this case enables arbitrary write and exection of shellcode in the Hardware abstraction layer's (HAL) memory address.
The exploit is happening in non-paged pool memory which the SMB server allocates for the large requests sent to it. This is quite important information as we will soon see.

### Finding the connections

I launched a Windows 7 SP1 virtual machine for testing this exploit. This is useful because it is easy to take a memory dump of the whole machine. After setting up my environment I took a memorydump in the middle of the exploit.

I am using [Volatility](http://www.volatilityfoundation.org/) to explore the memory dumps and trying to find the data that reside in the non-paged pool. All in all i'm using three primary memory dumps to explore the exploit code of the Metasploit plugin.

1. After the first "large buffer" packet is sent (Line 186 [[1](#sources)])
```
# Step 2: Create a large SMB1 buffer
print_status("Sending all but last fragment of exploit packet")
smb1_large_buffer(client, tree, sock)
```
2. After the first grooming packets are sent
```
# Step 3: Groom the pool with payload packets, and open/close SMB1 packets
print_status("Starting non-paged pool grooming")
# initialize_groom_threads(ip, port, payload, grooms)
fhs_sock = smb1_free_hole(true)
@groom_socks = []
print_good("Sending SMBv2 buffers")
smb2_grooms(grooms, payload_hdr_pkt)
```
3. After the second grooming packets are sent
4. After the malformed Trans2 packet is sent



<script src="https://gist.github.com/markusleh/9909454f19bb053458dd05dfe5e5e449.js"></script>


## Sources
[1] https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/windows/smb/ms17_010_eternalblue.rb

[2] https://gist.github.com/worawit/bd04bad3cd231474763b873df081c09a

[3] https://www.fireeye.com/blog/threat-research/2017/05/smb-exploited-wannacry-use-of-eternalblue.html
