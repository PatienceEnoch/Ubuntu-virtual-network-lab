# Local Site-to-Site IPsec VPN Validation

Before pushing the hybrid lab into AWS, I built a local stand-in for the cloud side so I could see the entire VPN path and troubleshoot it without treating IPsec as a black box.

The result is a working routed path from an Ubuntu client, through a strongSwan gateway, across an IPsec tunnel, into a simulated cloud network, and back again.

## Topology

~~~text
Ubuntu Client
10.10.10.10/24
      |
      | gateway: 10.10.10.1
      |
Ubuntu Server / VPN Gateway
LAN: 10.10.10.1/24
VPN side: 172.31.255.1/30
      |
      |  IKEv2 / IPsec
      |  10.10.10.0/24 <-> 10.20.0.0/24
      |
Fake Cloud Gateway
VPN side: 172.31.255.2/30
Cloud side: 10.20.0.1/24
      |
      | veth pair
      |
fake-ec2 network namespace
10.20.0.10/24
~~~

The fake cloud workload is a Linux network namespace rather than another VM. That kept the lab small while still giving the cloud side its own interface, IP address, route, ARP table, and isolated network stack.

## What I configured

### Ubuntu Server

- IPv4 forwarding
- strongSwan IKEv2/IPsec
- policy-based tunnel between `10.10.10.0/24` and `10.20.0.0/24`
- nftables forwarding rules
- NAT for normal Internet traffic
- NAT exemption for VPN-bound traffic
- automatic VPN startup
- one active Security Association per peer

The NAT exemption matters because traffic going to the fake cloud must keep its original `10.10.10.x` source address so it still matches the IPsec policy.

~~~text
10.10.10.0/24 -> 10.20.0.0/24   do not NAT
other traffic leaving Internet side   masquerade
~~~

### Fake Cloud Gateway

- VPN-facing address `172.31.255.2/30`
- IPv4 forwarding
- route back to `10.10.10.0/24`
- `cloud0` veth endpoint at `10.20.0.1/24`
- `fake-ec2` network namespace at `10.20.0.10/24`
- systemd service that recreates the namespace, veth pair, addressing, and return route after reboot

The namespace default route points to `10.20.0.1`.

## VPN parameters

The lab uses IKEv2 with a pre-shared key.

~~~text
IKE:  AES-256 / SHA-256 / MODP-2048
ESP:  AES-256 / SHA-256
Mode: tunnel
~~~

Pre-shared keys and other secrets are intentionally not stored in this repository.

## How I verified it

I did not stop at "the tunnel says established."

I checked the path at several layers:

- client routing with `ip route get`
- strongSwan Security Association state
- Linux XFRM policy and state
- nftables forwarding and NAT behavior
- packet counters
- ARP/neighbor state
- `tcpdump` on the fake cloud side
- end-to-end ICMP
- reboot persistence

A successful final test from the Ubuntu Client returned:

~~~text
4 packets transmitted, 4 received, 0% packet loss
~~~

The replies arrived with `ttl=62`, consistent with the two routed hops in the return path:

~~~text
fake-ec2
   |
Fake Cloud Gateway
   |
Ubuntu Server
   |
Ubuntu Client
~~~

TTL was useful supporting evidence for the routed path, but not proof of encryption. XFRM/strongSwan state and counters were the evidence that the traffic was actually being handled by IPsec.

## The failures that taught me the most

### The tunnel was up, but traffic still failed

The IKE and CHILD SAs were established, but that did not guarantee useful traffic.

The client route was correct and the Ubuntu Server was encrypting packets, yet the end-to-end ping still failed. That forced me to trace the packet instead of assuming the VPN was the problem.

### NAT was too broad

The Ubuntu Server originally had a general masquerade rule for traffic leaving its Internet-facing interface.

VPN-bound traffic needed an exemption:

~~~text
source:      10.10.10.0/24
destination: 10.20.0.0/24
action:      do not NAT
~~~

Without preserving the original source address, the packet would no longer match the intended IPsec selectors correctly.

### The forward path worked, but the return path did not

A packet capture on the fake cloud gateway showed:

~~~text
10.10.10.10 > 10.20.0.10   ICMP echo request
10.20.0.10 > 10.10.10.10   ICMP echo reply
~~~

The workload was replying, but the gateway initially had no route back to `10.10.10.0/24`.

That was a useful reminder that a healthy VPN and a healthy route are separate things:

- IPsec answers **how should this traffic be protected?**
- routing answers **where should this packet go next?**

Both have to be correct.

### The simulated cloud disappeared after reboot

The `fake-ec2` namespace and veth pair were originally created manually. Linux network namespaces and manually created veth interfaces are runtime objects, so they disappeared when the VM rebooted.

I fixed that by creating a systemd oneshot service that rebuilds:

- the `fake-ec2` namespace
- the `cloud0 <-> ec2eth0` veth pair
- both IP addresses
- the namespace default route
- the gateway return route

I then rebooted the fake cloud gateway and verified that the topology returned automatically.

### Duplicate VPN sessions

During testing I had two copies of the same tunnel active.

The strongSwan configuration allowed duplicate identities with:

~~~text
uniqueids=no
~~~

I changed that to:

~~~text
uniqueids=yes
~~~

After restarting strongSwan, only one Security Association remained active.

## Persistence test

I rebooted both sides separately.

After the fake cloud gateway reboot:

- `fake-ec2` returned automatically
- `cloud0` returned as `10.20.0.1/24`
- the return route returned
- the gateway could reach `10.20.0.10`

After the Ubuntu Server reboot:

- strongSwan automatically re-established one tunnel
- nftables reloaded the VPN NAT exemption
- the Ubuntu Client again reached `10.20.0.10` with 0% packet loss

That was the point where I considered the local validation lab complete.

## What this taught me

The biggest lesson was that **"VPN up" is not an end-to-end test**.

A working site-to-site path depends on several pieces lining up at the same time:

~~~text
client route
-> gateway forwarding
-> NAT policy
-> IPsec policy
-> encryption
-> remote forwarding
-> workload reachability
-> return route
-> return IPsec path
~~~

The troubleshooting became much easier once I stopped asking "why is the VPN broken?" and started asking "how far did this packet actually get?"

## Next step

This local setup gives me a controlled reference path before I continue the AWS version of the lab.

The next phase is to apply the same troubleshooting method to AWS Site-to-Site VPN, VPC routing, security controls, tunnel telemetry, and an actual cloud workload.
