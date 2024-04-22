---
title: "Understanding the Segmentation Offload in Linux Virtual Devices"
emoji: "🕳️"
type: "tech"
topics:
  - "linux"
  - "network"
  - "kernel"
  - "gso"
published: false
---

This post primarily focuses on explaining the high-level behavior of Segmentation Offload for virtual network devices, keeping in mind the use cases in orchestrators for VMs or containers.

Modern web applications are predominantly run on VM/cotainer orchestrators such as Kubernetes. These orchestrators often utilize virtual devices. It's common to see configurations where packets flow from application containers or VMs through veth or virtio-net, undergo encap/decap with VXLAN devices, and then are transmitted from physical devices.

Understanding how Segmentation Offload behaves in such infrastructures can help maximize application throughput and avoid unexpected bottlenecks.

# Refreshers

[Segmentation Offload](https://www.kernel.org/doc/html/next/networking/segmentation-offloads.html) in Linux refers to the technique of improving throughput by offloading the segmentation process of transport layer protocols such as TCP to the NIC layer. When using Segmentation Offload, the transport layer protocol can transmit data up to the maximum size allowed by the IP layer (approximately 64KB) in a single packet. Receiving side does an opposite; segmented packets coming from the network are aggregated into one large packet at the NIC layer before being processed by the protocol stack. This reduces the number of packets processed by the protocol stack, resulting in increased throughput.

![](https://storage.googleapis.com/zenn-user-upload/f014e09ceb67-20240421.png)

Linux has the concept of [virtual network devices](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking). Network devices in Linux do not necessarily correspond to physical NICs. For example, tunneling devices such as VXLAN or GRE devices performs tunnel encap/decap instead of sending/receiving packet through physical NICs. There are many kinds of virtual devices, like virtio-net, bridge, and VRF, are abstracted as network devices in Linux alongside physical NICs, and the protocol stack does not distinguish between physical and virtual devices.

# How Linux Implements Segmentation Offload

Roughly speaking, in Linux, Segmentation Offload has two implementations. Software-based and hardware-based implementations. The software implementation for the sending side is called Generic Segmentation Offload (GSO), and for the receiving side, it's called Generic Receive Offload (GRO). The hardware implementation is called differently depending on the vendor, but a vendor-independent naming convention often refers to the sending side as Transmission Segmentation Offload (TSO) and the receiving side as Large Receive Offload (LRO). Hardware implementation offers the advantage of not consuming host CPU compared to software implementation ([there doesn't seem to be a significant difference in throughput](https://docs.openstack.org/developer/performance-docs/test_results/hardware_features/hardware_offloads/test_results.html#hw-features-offloads)).

You can check what kind of Segmentation Offload a device supports using `ethtool`. Here's an example for the Wi-Fi module on my Linux laptop:

```shell-session
$ ethtool -k wlp0s20f3 | grep -e "receive-offload" -e "segmentation"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
```

::: message
You'll notice that `generic-segmentation-offload: on` indicates GSO can be turned on or off, but to my knowledge, since [this commit](https://github.com/torvalds/linux/commit/0a6b2a1dc2a2105f178255fe495eb914b09cb37a) introduced in Linux 4.17, at least for TCP, this flag is meaningless, and GSO cannot be turned off. Considering various benefits mentioned in the commit message, they decided to always left it on. Additionally, LRO is often off by default for physical NIC devices. It seems there are many NICs where LRO on the receiving side is complex and doesn't work well.

Update: I received a note from @q2ven that GSO can be disabled for TCP when features like MD5 or TCP-AO are enabled. Thank you!
:::

The support for Segmentation Offload for each protocol is indicated in the output. `[fixed]` indicates an unchangeable parameter. Thus, `off [fixed]` means "not supported". The kernel checks whether the device can perform segmentation for the packets it sends based on this information. If the device cannot, it uses GSO for segmentation; otherwise, it passes the packet to the device driver without segmentation. Conversely, on the receiving end, if the packets are already aggregated, they are passed directly to the protocol stack; otherwise, GRO is used for aggregation. The segmentation process within the device driver is device-dependent.

# Principles of Segmentation Offload in Virtual Devices

A basic rule of Segmentation Offload in virtual devices is, Linux delays segmentation until packets reach the physical NIC whenever possible. This is because packets entering virtual devices are often further processed in software. Therefore, segmenting the packet doesn't have any performance benefit. While the behavior of Segmentation Offload in virtual devices varies by device, this principle is the same.

Let's take the implementation of veth as the simplest example. Veth devices simply passes through the packet to the peer device without any segmentation. To achieve this, veth appears to support segmentation offloading for most of the available protocols from the kernel's perspective.

```shell-session
$ ethtool -k veth0 | grep -e "receive-offload" -e "segmentation"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: on
	tx-tcp-mangleid-segmentation: on
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: off
large-receive-offload: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: on
tx-gre-csum-segmentation: on
tx-ipxip4-segmentation: on
tx-ipxip6-segmentation: on
tx-udp_tnl-segmentation: on
tx-udp_tnl-csum-segmentation: on
tx-tunnel-remcsum-segmentation: on
tx-sctp-segmentation: on
tx-esp-segmentation: on
tx-udp-segmentation: on
```

Veth doesn't do actual segmentation, but behaving like this lets kernel to pass unsegmented packet to the veth driver. Once the device driver receives the packet, it can do anything. Kernel doesn't even check whether the actual segmentation occured.

This is the common trick used by virtual network device drivers. Declare the support of Segmentation Offload for certain protocols, let the kernel to pass unsegmented packets, and pass through it without segmentation.

# Tunneling Devices

Tunnel devices are implementations of tunneling protocols such as IPIP, GRE, VXLAN, and Geneve in Linux. Packets entering tunnel devices via routing are encapsulated with the tunnel protocol header and then reprocessed by the protocol stack. The same applies to segmented packets. The driver of the tunnel device passes the packets to the protocol stack without segmentation after encapsulating them.

![](https://storage.googleapis.com/zenn-user-upload/dbcd5e242b46-20240421.png)

So far, it's straightforward, but the issue arises when dealing with the processing of encapsulated packets upon reaching the physical NIC. To segment encapsulated packets, it's necessary to segment the inner packets and then encapsulate each packet with the outer tunneling protocol header. In other words, to enable segmentation on the physical NIC, the NIC must support this process. In ethtool output, the following features need to be enabled. If the physical NIC does not support these features, processing will be handled by GSO.

```
tx-gre-segmentation: on <= GRE
tx-ipxip4-segmentation: on <= IPv4 in IPv4 (IPIP)
tx-ipxip6-segmentation: on <= IPv4 in IPv6 or IPv6 in IPv4
tx-udp_tnl-segmentation: on <= UDP encapsulation for VXLAN, Geneve, etc.
```

## Nested Encapsulation

Segmentation Offload for tunnels nested more than one level is [not supported](https://www.kernel.org/doc/html/next/networking/segmentation-offloads.html#ipip-sit-gre-udp-tunnel-and-remote-checksum-offloads). When the packet is forwarded to the tunnel device for the second encapsulation, segmentation is performed by GSO.

![](https://storage.googleapis.com/zenn-user-upload/c83223987fb4-20240421.png)

The way it works is that tunnel devices do not support segmentation like tx-gre-segmentation for encapsulation, and the kernel falls back to GSO. The following is an example of IPIP device.

```shell-session
$ ethtool -k tunl0 | grep -e "receive-offload" -e "segmentation"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: on
	tx-tcp-mangleid-segmentation: on
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: on
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: on
```

Therefore, encapsulation processing and subsequent protocol stack processing are performed individually for each segmented packet after the second encapsulation. This incurs a significant performance penalty. Below are the results of running `iperf` with increasing levels of tunneling on a local machine connected between two Netns. Although these results are for local measurement and may differ significantly from actual machine results, they are sufficient to observe the impact on performance.

![](https://storage.googleapis.com/zenn-user-upload/dc274d04d0fb-20240420.png)

A significant decrease in throughput is observed in the Double Tunnel case. This is because in the case of No Tunnel and Single Tunnel, segmentation is not performed end-to-end, whereas in the case of Double Tunnel and beyond, segmentation occurs in the second encapsulation and onwards. While there may be cases where nested tunnels are unavoidable, but it's generally advisable to avoid them whenever possible.

# Bridge and VRF

Bridge and VRF (Virtual Routing and Forwarding) devices are devices that perform switching or routing based on the L2 FDB (Forwarding Data Base) or L3 FIB (Forwarding Information Base) they maintain internally for incoming packets. There are two cases: packets arriving from devices attached to Bridge or VRF, and packets being forwarded by Bridge or VRF due to routing, each with different behaviors.

When packets arrive from attached devices, only forwarding processing is performed, and segmentation is not performed. On the other hand, packets that arrive due to routing are processed similarly to other devices. While Bridge supports segmentation for almost all protocols, VRF does not support encapsulation-related segmentation. Unfortunately, I couldn't investigate the reasons behind this difference this time.

```shell-session
$ ethtool -k bridge | grep -e "receive-offload" -e "segmentation"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: on
	tx-tcp-mangleid-segmentation: on
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
tx-fcoe-segmentation: on
tx-gre-segmentation: on
tx-gre-csum-segmentation: on
tx-ipxip4-segmentation: on
tx-ipxip6-segmentation: on
tx-udp_tnl-segmentation: on
tx-udp_tnl-csum-segmentation: on
tx-tunnel-remcsum-segmentation: on
tx-sctp-segmentation: on
tx-esp-segmentation: on
tx-udp-segmentation: on
```

```shell-session
$ ethtool -k vrf0 | grep -e "receive-offload" -e "segmentation"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: on
	tx-tcp-mangleid-segmentation: on
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: on
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: on
```

# virtio-net

The virtio-net specification includes [the specification for Segmentation Offload](https://docs.oasis-open.org/virtio/virtio/v1.2/csd01/virtio-v1.2-csd01.html#:~:text=5.1.6.2%20Packet%20Transmission), and virtio-net implementations can pass packets without segmentation across guest and host. In the guest-side Linux, virtio-net appears as follows:

```
$ ethtool -k ens2 | grep -e "segmentation" -e "receive-offload"
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: on
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
```

::: message
The output may vary depending on QEMU settings. In my case, I specify `-device virtio-net-pci,netdev=user.0,csum=on,guest_csum=on,gso=on,host_tso4=on,host_tso6=on,host_ecn=on,guest_tso4=on,guest_tso6=on,guest_ecn=on` as an option for QEMU. In my environment, `tx-udp-segmentation` was `off [fixed]`, but it seems to be supported under the name USO (UDP Segmentation Offload) according to the specification. Perhaps this is mainly used for protocols like QUIC.
:::

Since TCP Segmentation Offload is supported, the guest kernel passes unsegmented packets to the virtio-net guest-side driver. The virtio-net guest-side driver writes metadata related to Segmentation Offload and the packet to the shared memory (virt-queue) between the guest and host and sends the packet to the host.

When Linux is the host, tun/tap devices are commonly used as the backend for virtio-net. Tun/tap devices have the ability to send and receive Segmentation Offload metadata along with packets by adding a special [header](https://github.com/torvalds/linux/blob/13a2e429f644691fca70049ea1c75f135957c788/include/uapi/linux/virtio_net.h#L188) to the packets as specified in the virtio-net standard. QEMU's virtio-net driver and vhost-net utilize this feature to exchange information about Segmentation Offload with the host's Linux kernel. Further details about this mechanism are well summarized in [this blog](https://blog.cloudflare.com/virtual-networking-101-understanding-tap).


# Conclusion

We've walked through the high-level behavior of Segmentation Offload in various virtual devices in Linux. While there was once a discourse suggesting that disabling Segmentation Offloading was a best practice (at least around me), I believe that disabling it has become a disadvantageous as server bandwidth increases. Over the years, I've encountered numerous issues with Segmentation Offloading while developing network layers for platforms like OpenStack and Kubernetes. I have plenty of stories on this topic, so I hope to write about them somewhere when I feel motivated.
