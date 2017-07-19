---
layout: post
title: "Scaling Docker's Network, Is it Enough Bandwidth for You?"
date: 2017-07-18
---


This post is focused on network throughput performance of Docker containers existing on the same physical machine, one node. I will briefly mention how these networks work and show how these containers can communicate with each other. I will provide my average throughput results for both my IPv4 and IPv6 tests as well as some insight into how these configurations will scale when running many thousands of containers on 1 single node.

First of all, here is the machine information I ran my tests on using docker info command:

<pre><code>
Docker Version: 1.12.1
Storage Driver: overlay
 Backing Filesystem: xfs
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge null host overlay
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Security Options: seccomp
Kernel Version: 3.10.0-514.el7.ppc64le.debug
Operating System: Red Hat Enterprise Linux
OSType: linux
Architecture: ppc64le
CPUs: 20
Total Memory: 251.2 GiB
</code></pre>

My results are the average of 10 netperf tests. Netperf ran for 5 minutes continually sending traffic to a specific IP address and port.

For this specific type of experiment you could measure the throughput of the lo interface on your node and compare your results from the container tests. This is a good test because it avoids *most* of the kernel networking stack. Lets first describe the bridge network.

# lo: 24.38 GB/s

## default Bridge Network
### IPv4: 2.23 GB/sec
### IPv6: 2.65 GB/sec
### IPv4 no iptables:  6.81Gb/s

![bridge](/docker-bridge.png)

2 containers communicating with a Linux bridge in between them. This means there are 2 veth (Virtual Ethernet) pairs with 1 side of the pair connected to a Linux bridge and the other side connected to the container. The containers exist in their own separate network namespaces which is why they have different IP addresses. 

This setup script may be helpful to some: https://github.com/mtarsel/link-containers/blob/master/bridge-it.sh

Features and Notes:
<ul>
<li>uses iptables</li>
<li>Can only scale to available ports on bridge</li>
<li>different options for bridge type</li>
</ul>

The NAT is handled by ```iptables```. 

The bridge network shown above does provide a way for the container to communicate with an outside network. The iptables can route traffic in and out with certain rules which I will not outline here.

The 2 networks described below do not give the containers access from another network; There is no communication with the outside world. There is no NAT table that a bridge provides because *there is no bridge*. 

## Container to Container via veth Pair
### IPv4: 2.49 GB/sec 
### IPv6: 2.91 GB/sec
### IPv4 no iptables: 4.97GB/s

 ![veth](/veth-docker.png)

Instead of connecting a container to a bridge via a veth pair, we can connect 2 containers together the same way except with no bridge. 
<ul>
<li> packets still go through veth driver</li>
<li> packets will travel through kernel networking stack </li>
<li> still need IP address for containers because they are in separate network namespaces</li>
<li> Not a docker command  https://github.com/mtarsel/link-containers/blob/master/tunneler.sh </li>
</ul>

## Shared Network Namespace
### IPv4: 22.45 GB/sec
### IPv6: 21.46 GB/sec

![shared](/shared-ns.png)

Pass the - -net=container:your_container1 option to docker run command
The veth interface exists in the network namespace. The namespace is shared by both containers so the two containers have the same veth interface - which means they have the same IP address. 

<ul>
<li>fastest throughput</li>
<li>very easy to setup</li>
<li>containers have the same IP address</li>
</ul>

Thanks for reading!

Here is a summary of the results (GB/sec):
| lo = 24.38 | IPv4 | IPv6 | IPv4 no iptables|
| --------- | --------- | --------- | ---------|
| docker0 Bridge | 2.23 | 2.65 | 6.81|
| veth pair | 2.49 | 2.91 | 4.97|
| Shared NS | 22.45 | 21.46|
