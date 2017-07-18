---
layout: post
title: "Scaling Docker's Network, Is it Enough Bandwidth for You?"
date: 2017-07-18
---


This post is focused on network throughput performance of Docker containers existing on the same physical machine, one node. I will briefly mention how these networks work and show how these containers can communicate with each other. I will provide my average throughput results for both my IPv4 and IPv6 tests as well as some insight into how these configurations will scale when running many thousands of containers on 1 single node.

First of all, here is the machine information I ran my tests on using docker info command:

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

 I will provide a very short description about each network setup however I am mostly focused on which network will give you the best throughput (or bandwidth) for containers communicating with each other on a single physical node. Lets first describe the bridge network.
 
My results are the average of 10 netperf tests. Netperf ran for 5 minutes continually sending traffic to a specific IP address and port.

For this specific type of experiment you could measure the throughput of the lo interface on your node and compare your results from the container tests. Why is this a good comparison? Because this test avoids all of the kernel networking stack and doesnt call syscalls with golang....

lo: 24.38 GB/s

default Bridge Network
IPv4: 2.23 GB/sec
IPv6: 2.65 GB/sec
IPv4 no iptables:  6.81Gb/s

[image of network setup]
2 containers communicating with a Linux bridge in between them. This means there are 2 veth pairs with 1 side of the pair connected to a Linux bridge and the other side connected to the container. The containers exist in their own separate network namespaces which is why they have different IP addresses. 

This setup script may be helpful to some: https://github.com/mtarsel/link-containers/blob/master/bridge-it.sh


Features and Notes:
uses iptables
Can only scale to available ports on bridge
different options for bridge type
 
The NAT is handled  by iptables. Running the same test again with iptables off shows:

The bridge network shown above does provide a way for the container to communicate with an outside network. The iptables can route traffic in and out with certain rules which I will not outline here.

The 2 networks described below do not give the containers access from another network; there is no communication with the outside world. There is no NAT table that a bridge provides because there is no bridge. You will still need to add another interface inside container or add the container to another network which does have some kind of bridge.

Container to Container via veth Pair
IPv4: 2.49 GB/sec 
IPv6: 2.91 GB/sec
IPv4 no iptables: 4.97GB/s

[image of network setup]


Instead of connecting a container to a bridge via a veth pair, we can connect 2 containers together the same way except with no bridge. 
packets still go through veth driver
packets will travel through kernel networking stack 
still need IP address for containers because they are in separate network namespaces
Not a docker command  https://github.com/mtarsel/link-containers/blob/master/tunneler.sh


Shared Network Namespace
IPv4: 2.23 GB/sec
IPv6: 2.65 GB/sec

[image of network setup]

Pass the - -net=container:your_container1 option to docker run command
Two containers have the same virtual Ethernet interfaces. Similar to the host network however the 2 containers do not have any physical interfaces because the containers network interfaces exist in a separate namespace from the node.
fastest throughput
very easy to setup
containers have the same IP address
