# openshift-ipi-bm-libvirt
Lab setup to test the IPI Bare Metal installer using libvirt

# What?

OpenShift can be installed on Bare Metal using Installer Provisioned infrastructure. What it really means is that Bare Metal becomes an actual platform, supporting MachineSets, their scaling and other beautiful features that UPI clusters don't have. Perhaps we'll soon get a decent Load Balancer for this platform in the form of MetalLB.

This project aims to provide means to test this all without having much of a metal: one physical machine or a VM that supports nested KVM (not just qemu) - that's all. Well, you might want to spend some extra $$ on a domain name, but you don't have to.

# Why?

Because otherwise you'll have to have a lot of actial and very much bare **metal**.

# How?

First, have a look at the [official guide](https://docs.openshift.com/container-platform/4.9/installing/installing_bare_metal_ipi/ipi-install-overview.html).
Just look at the pictures and come back.

Now, having you back, the installer is to run on a preconfigured host. It will start a libvirt guest that will become the quite familiar Bootstrap node. The installer expects to find two bridges on the host to patch up this VM to:

* The `baremetal` bridge
* The optional `provisioning` bridge

The names a tweakable. The `baremetal` bridge is essential - it's supposed to link the bootstrap VM with the rest of the cluster nodes. This network will become the HostNetwork of the OCP nodes. As for the `provisioning` bridge, it's only needed if you want to install CoreOS using PXE. I don't.

First, I don't want my Nodes to have another network connection. Second, I think the platform-inness of the Bare Metal in this case is a lot better off with the BMC-virtualmedia way of provisioning. We'll need the BMC access anyway, because the installer needs to be able to power On/Off the machines. Herewith I discard the `provisioning` network. Boom! Done.

## Challenge #1

Avoid virtualization inception.

If we want to do this on one single host, then our OCP Nodes and the installer-made bootstrap node will all be VMs on this host. Here we need to undestand the difference between the **bootstrap host** and a **bootstrap VM**. The first one is **the** thing - we run the installer command on it. Normally it's yet another BM host. The latter one is **an** ephemeral VM that installer created on the bootstrap host. It lives only untill the controll plane is self-content enough to evict it. I don't want to nest the virtualization here. I'd rather have my only machine itself act as the bootstrap host and let it create the bootstrap VM alongside the cluster node VMs.

Great idea! But this won't work just like that. As you remember, the installer will look for a bridge called `baremetal` by default. It will do that using `virsh iface-list` command. And virbr0 is not listed by this command. There are two options:

1. You create the virbr0 or whatever you want to call it and then create a libvirt network on top of it. [This way](https://libvirt.org/formatnetwork.html#examplesBridge). Then `virsh iface-list` will show it and the installer will understand that that's the `baremetal` network where you want to plug your bootstrap VM.

Problem is - you loose quite a bit of nice functionality of libvirt network definition here, including dnsmasq and firewalld integration. So you'll have to take care of DNS and DHCP. Too hard.

2. You create an extra bridge, call it something like `baremetal` and ...

## Challenge #2

How to link to linux bridges?

You can easily do this by creating a **veth** pair. It's a virtual Ethernet that comes as a pair, well sort of like two nics patched with a crossover. You can control the placement of each device in the pair. That's exactly what we're gonna do: we'll plug one into virbr0, that has a libvirt-managed dnsmasq with DHCP and DNS, and the other one to the so-far-hanging-in-the-air `baremetal` bridge.

Up next ...

## Challenge #3

BMC.

Seriously, there's no iDRAC or ILO for VMs. Wait, there is! [virtualbmc](https://docs.openstack.org/virtualbmc/latest/) from OpenStack can do as much as controlling the power of qemu quests. Not quite enough though. Okay, then how about [Virtual Redfish BMC](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html)? It can do [virtual media](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html#virtual-media-resource), and that's what we are after.

Having all challenges resolved, meet the big pic:

## The big pic

```
       BM host or VM supporting nested KVM

       ┌──────────────────────────────────────────────────────────────────────────────┐
       │eth0 (public IP)                                          VM ctrl-0           │
     ┌─┴─┐                                                       ┌────────────────┐   │
─────┤   │◄───────────────┐    dnsmasq: DHCP, DNS, NAT           │                │   │
     └─┬─┘                │    ┌─────────────────────────┐   ┌───┤                │   │
       │                  │    │virbr0 (192.168.122.1)   │   │   │                │   │
       │                  │  ┌─┴─┐                      ┌┼┐  │   └────────────────┘   │
       │                  └──┤   │                vnet0 │ ┼──┘     ...                │
       │                     └─┬─┘                      └┼┘                           │
       │                     ▲ │                         │        VM worker-n         │
       │                     │ │                        ┌┼┐      ┌────────────────┐   │
       │  ---------------    │ │                  vnetX │ ┼──┐   │                │   │
       │  virtual redfish    │ │      vethlv            └┼┘  └───┤                │   │
       │                     │ │       ┌──┐              │       │                │   │
       │  192.168.122.1:8000 │ └───────┼  ┼──────────────┘       └────────────────┘   │
       │                     │         └─┼┘  ▲                                        │
       │  qemu:///system     │           │   | DHCP, DNS                              │
       │  -------------------┘           │   |                                        │
       │                               ┌─┼┐  ▼                                        │
       │                       ┌───────┼  ┼──────────────┐        VM bootstrap        │
       │                       │       └──┘              │       ┌────────────────┐   │
       │                       │      vethbm            ┌┼┐      │                │   │
       │                       │                  vnetY │ ┼──────┤                │   │
       │                       │                        └┼┘      │                │   │
       │                       │baremetal                │       └────────────────┘   │
       │                       └─────────────────────────┘                            │
       │                                                                              │
       │                                                                              │
       └──────────────────────────────────────────────────────────────────────────────┘
       
```

back to challenges

## Challenge #4

Resources.

OCP 4.9 will not install if your control plane nodes don't have 12GB of RAM (you might be lucky with 10-11, but some manual intervention may be needed) and 4 CPU cores. Workers need as much as 8GB and 2 Cores, otherwise "can't write, not enough space" during provisioning and CPU pressure are guaranteed. 

Feel free to overcommit the CPU and rely on Hyperthreading.

You do want SSDs to avoid etcd issues. Feel free to thin-provision the qcow's.

My machine is a second hand herzner (auction) consumer grade server with 6 Cores, 64GB of RAM and 1TB SSD.

# TODO the rest
