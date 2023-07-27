## Understanding 'ip a' output.

`ip` is the successor of the deprecated `ifconfig` utility we are all familiar with. I have used this utility for a while to get some useful information about my network interface, though I never understood what each line _actually_ represents, and I thought maybe it's time to _really_ understand what `ip a` echoes out.

It's important to mention that you should have basic understanding of C code as well as basic networking terminology to understand the following concepts.

We use the following `ip a` example:

```
enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet xx.xx.xx.xx/24 brd xx.xx.xx.xxx scope global dynamic noprefixroute enp5s0
    valid_lft 2269sec preferred_lft 2269sec
```
---

### Quick side notes

Network interfaces are used when packets are exchanged with the outside world. In our case, I will talk about a 'real' network interface that represents an actual physical interface; unlike, for example, the `loopback`, that is actually just a virtual network interface.

The `ip a` _only_ needs to communicate with the software defined interface, through a socket. Just a side note: IIRC, `ifconfig` was using `ioctl`s to communicate with the interface; things have changed.

### Diving into the code

Each network interface is represented by the `net_device` structure defined in `include/linux/netdevice.h`, and contains all the necessary information required in order to understand, manipulate, and isolate network interfaces from each other. The `net_device` struct is huge, so we'll only cover what's needed to understand `ip a`.

Firstly, the actual name of the interface, is sometimes defined by the **Predictable Network Interface naming** mechanism (you can control the behaviour via systemd; not discussed here). In my case, my network interface name is `enp5s0` and the crafting goes like this:

- Since my network interface is of type Ethernet (See available types in `/include/uapi/linux/if_arp.h`), the first two chars are `en`.

- Next is the pci bus number; My Ethernet PCI controller device is at bus number `05` (can be seen via `lspci`). Hence, the next two chars are `p5`.

- Finally, the last two are defined by the slot index in the pci bus. In my case it's just 0. (`s0`)

I believe the kernel can handle this behaviour of systemd with the following flags:
```c
/* interface name assignment types (sysfs name_assign_type attribute) */
#define NET_NAME_UNKNOWN	    0	/* unknown origin (not exposed to userspace) */
#define NET_NAME_ENUM		    1	/* enumerated by kernel */
#define NET_NAME_PREDICTABLE	2	/* predictably named by the kernel */
#define NET_NAME_USER		    3	/* provided by user-space */
#define NET_NAME_RENAMED	    4	/* renamed by user-space */
```
Though I'm _really_ not sure, as I don't recall seeing any communication via systemd with one of those flags. I believe it's somehow related, that's why I leave it here for now.

---

**Moving next in line**, are the `flags`, describing what the network interface is capable of, its status, and so on. Some of these flags are volatile, meaning they are only handled by the kernel and cannot be manipulated, say, from user-space. Some can be changed via `sysfs` and other interfaces.

I'll only be showing the bit flags set on my interface. Goto `include/uapi/linux/if.h` if you wanna see more available flags and deeper description.

```c
IFF_UP: interface is up. Can be toggled through sysfs
IFF_BROADCAST: broadcast address valid. Volatile
IFF_MULTICAST: Supports multicast. Can be toggled through sysfs
IFF_LOWER_UP: driver signals L1 up. Volatile
```

**Next** is the `mtu`, or Maximum Transfer Unit, used by the **network layer** during packet transmission, indicates the largest possible packet length that can be used in one transmission/payload. Ethernet is set to `ETH_DATA_LEN` which is 1500 bytes, or when talking networking - octets.

**Next** is `qdisc`, which shows the current used queueing mechanism for incoming packets. There are a couple mechanisms, however, we won't cover them in here, but their general role is to define rules, such as priorities and efficiency rules. Mine uses the `CoDel - Fair Queuing (FQ) with Controlled Delay (CoDel)` (`fq_codel`). 

I believe `fq_codel` is the default used for network interfaces, though I'm not sure. I saw somewhere that systemd defines `net.core.default_qdisc = fq_codel`, but it may have been changed since then. In addition, For the curious reader, there is a great (!) cover up of mechanisms at *[mikrotik Queues Docs](https://help.mikrotik.com/docs/display/ROS/Queues)*.

**Next**, is `state` which simply shows the current state of the device. Mine shows `UP` since my network interface is online and setup, and able to transmit and receive packets.

Defined as such by the linux kernel at `include/uapi/linux/if.h`
```c
/* RFC 2863 operational status */
enum {
	IF_OPER_UNKNOWN,
	IF_OPER_NOTPRESENT,
	IF_OPER_DOWN,
	IF_OPER_LOWERLAYERDOWN,
	IF_OPER_TESTING,
	IF_OPER_DORMANT,
	IF_OPER_UP, /* Ours (: */
};
```

**We are almost finished**, the next one is `group`, set to default. The kernel describes it as below, defined in `include/uapi/linux/netdevice.h`

```c
/* Initial net device group. All devices belong to group 0 by default. */
#define INIT_NETDEV_GROUP	0
```
**Finally** is `qlen`, that is the `tx_queue_len`, this is the maximum number of frames used by the **link layer** that can be queued on the device's transmission queue. Set to 1000 by default for Ethernet devices. Changeable.

I have mentioned both frames/packet and both link layer and network layer. It's off-topic and we won't talk about it, but we can just say that it's related to the [Internet Protocol Suite](https://en.wikipedia.org/wiki/Internet_protocol_suite) and defined by the [Protocol Data Unit](https://en.wikipedia.org/wiki/Protocol_data_unit)

---
We have covered the first 'line' of `ip a`'s output, moving on to the next one.
**TBD**