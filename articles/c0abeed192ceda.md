---
title: "Playing around with the FRR FPM (Forwarding Plane Manager)"
emoji: "👂"
type: "tech"
topics:
  - "go"
  - "linux"
  - "network"
  - "frrouting"
  - "containerlab"
published: true
published_at: "2022-12-25 18:29"
---

# Motivation
Recently I learned FRR has a feature called [FPM](https://docs.frrouting.org/projects/dev-guide/en/latest/fpm.html) (Forwarding Plane Manager) which allows the process out of FRR to learn the content of the RIB via RTNetlink (or Protobuf) message through a TCP socket. Since I'm interested in how the external software like Cilium, can integrate with FRR, this was an exciting stuff, so I tried it.

# Example Setup
Just follow the instruction of the [FRR document](https://docs.frrouting.org/projects/dev-guide/en/latest/fpm.html#dplane-fpm-nl). It's straightforward. Note that I only tried `dplane_fpm_nl` version of the FPM.

## Setup the Lab
I used [Containerlab](https://containerlab.dev/) for my experiment. This topology definition makes two FRR nodes and connect them using eBGP.

```yaml
name: fpm-demo
topology:
  kinds:
    linux:
      cmd: bash
  nodes:
    router0:
      kind: linux
      image: frrouting/frr:latest
      exec:
        # Boiler plate to make FRR work
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - sed -i -e 's/zebra_options.*/zebra_options=\"  -A 127.0.0.1 -s 90000000 -M dplane_fpm_nl\"/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        # FRR configuration
        - >-
          vtysh -c 'conf t'
          -c 'fpm address 127.0.0.1 port 2620'
          -c '!'
          -c 'router bgp 65000'
          -c '  no bgp ebgp-requires-policy'
          -c '  bgp router-id 10.0.0.1'
          -c '  neighbor PEERS peer-group'
          -c '  neighbor PEERS remote-as external'
          -c '  neighbor PEERS capability extended-nexthop'
          -c '  neighbor net0 interface peer-group PEERS'
          -c '!'
    router1:
      kind: linux
      image: frrouting/frr:latest
      exec:
        - touch /etc/frr/vtysh.conf
        - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
        - sed -i -e 's/zebra_options.*/zebra_options=\"  -A 127.0.0.1 -s 90000000 -M dplane_fpm_nl\"/g' /etc/frr/daemons
        - /usr/lib/frr/frrinit.sh start
        - >-
          vtysh -c 'conf t'
          -c 'router bgp 65001'
          -c '  no bgp ebgp-requires-policy'
          -c '  bgp router-id 10.0.0.1'
          -c '  neighbor PEERS peer-group'
          -c '  neighbor PEERS remote-as external'
          -c '  neighbor PEERS capability extended-nexthop'
          -c '  neighbor net0 interface peer-group PEERS'
          -c '  address-family ipv4 unicast'
          -c '    redistribute connected'
          -c '  exit-address-family'
          -c '!'
    fpm-logger:
      kind: linux
      image: yutarohayakawa/fpm-logger:latest
      network-mode: container:router0
      startup-delay: 3
      cmd: "bash -c \"fpm-logger | ip monitor all file /dev/stdin\""
  links:
    - endpoints: ["router0:net0", "router1:net0"]
```

### FRR Settings
To enable FPM support, I did following on the `router0`.
1. Add `-M dplane_fpm_nl` to Zebra's command line options
2. Add `fpm address 127.0.0.1 port 2620` to FRR configuration

### The `fpm-logger` container
Setup a server to interpret the FPM message. This time, I wrote a very simple Go program that strips the [FPM header](https://docs.frrouting.org/projects/dev-guide/en/latest/fpm.html#dplane-fpm-nl) and streams the RTNetlink message to the standard output. So that we can pipe it to the `ip monitor` command and don't have to decode Netlink message by ourselves. The container puts `ip monitor` logs to standard output, so we can check the decoded Netlink message with `docker logs`.

:::details Click to see the code
```go
package main

import (
	"encoding/binary"
	"io"
	"net"
	"os"
)

type FPMHeader struct {
	Version     uint8
	MessageType uint8
	MessageLen  uint16
}

func handleConnection(conn net.Conn) {
	for {
		h := FPMHeader{}
		binary.Read(conn, binary.BigEndian, &h.Version)
		binary.Read(conn, binary.BigEndian, &h.MessageType)
		binary.Read(conn, binary.BigEndian, &h.MessageLen)

		if h.Version != 1 {
			panic("Unsupported FPM frame version")
		}

		if h.MessageType != 1 {
			panic("Unsupported FPM frame type")
		}

		n, err := io.CopyN(os.Stdout, conn, int64(h.MessageLen-4))
		if err != nil {
			panic(err)
		}

		if n != int64(h.MessageLen-4) {
			panic("Couldn't read entire message")
		}
	}
}

func main() {
	ln, err := net.Listen("tcp", "127.0.0.1:12345")
	if err != nil {
		panic(err)
	}

	for {
		conn, err := ln.Accept()
		if err != nil {
			panic(err)
		}
		handleConnection(conn)
	}
}
```
:::

## See the initial output
This is the output of the `docker logs` of the `fpm-logger` container right after I build the lab.

```
[NEXTHOP]id 8 dev eth0 proto zebra
[NEXTHOP]id 9 dev net0 proto zebra
[NEXTHOP]id 10 via 172.20.20.1 dev eth0 proto zebra
[NEXTHOP]id 11 via fe80::a8c1:abff:fef8:f8db dev net0 proto zebra
[NEXTHOP]id 7 dev eth0 proto zebra
[ROUTE]0.0.0.0/0 nhid 10 proto kernel metric 20
[ROUTE]172.20.20.0/24 nhid 7 proto kernel metric 20
[ROUTE]::/0 nhid 11 proto kernel metric 20
[ROUTE]2001:172:20:20::/64 nhid 8 proto kernel metric 20
[ROUTE]fe80::/64 nhid 8 proto kernel metric 20
```

## Add/Delete a route
Let's add a route and see what happens. Add connected route to `router1` by assigning an address to loopback device. `router1` should advertise it to `router0`.

```
docker exec -it clab-fpm-demo-router1 ip addr add 10.0.0.0/32 dev lo
```

And on the `fpm-logger`'s log, I could see
```
[ROUTE]10.0.0.0 nhid 13 proto bgp metric 20
```

Delete also works similarly.

```
docker exec -it clab-fpm-demo-router1 ip addr del 10.0.0.0/32 dev lo
```

```
[ROUTE]Deleted none 10.0.0.0 proto bgp metric 20
```

# Use cases?
SONiC uses FPM (see [this presentation](https://f990335bdbb4aebc3131-b23f11c2c6da826ceb51b46551bfafdc.ssl.cf2.rackcdn.com/images/1d66e872946c15dc71031301994daba0cf21512f.pdf)) to synchronize the routing state with ASICs. As this usecase shows, FPM is originally invented for ASIC offloading.

However, I'm interested in another software usecases. During this experiment I tried to see if we can receive to the routes that are **rejected from the local kernel** (because of the lack of support, missing kernel configuration, etc...) from FPM. And that **partially worked**.

That means, if we could prepare the alternative dataplane implementation, we can support the features which are implemented in FRR (and latest kernel), but not yet implemented into the local kernel. I think this is an interesting use case of the programmable dataplane technologies such as eBPF.

However, as I wrote above, currently it only works partially, because the latest FRR still missing support of sending the `NextHop` netlink message when the underlying kernel doesn't support it ([see here](https://github.com/FRRouting/frr/blob/7b78f6c87f8e1e8cb6804ef7e8ea7dac43836593/zebra/rt_netlink.c#L2162-L2165)). I think this limitation is unnecessary for the FPM usecases, so I submitted a PR. I hope it will be merged.

https://github.com/FRRouting/frr/pull/12568

Looking forward to exploring more interesting usecases!

# Artifacts
All the artifacts in this post are available on this repo.
https://github.com/YutaroHayakawa/fpm-logger
