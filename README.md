# Ubuntu Virtual Network Lab

A two-system Ubuntu lab I built to make routing, NAT, DNS, SSH, services, and packet flow visible from the operating-system level.

## Topology

~~~text
Internet
   |
VirtualBox NAT
   |
Ubuntu Server
enp0s3: NAT-facing
enp0s8: 10.10.10.1/24
   |
lanlab
   |
Ubuntu Client
10.10.10.10/24
gateway: 10.10.10.1
~~~

The client has no direct NAT adapter. It reaches external networks through the Ubuntu Server, which acts as its router.

## What I configured

**Ubuntu Server**

- Static address on the internal interface
- IPv4 forwarding
- nftables NAT and forwarding
- OpenSSH for remote administration
- Apache for an internal web-service test
- tcpdump for packet capture

**Ubuntu Client**

- Static IPv4 configuration
- Default route through the server
- Manual DNS configuration
- Connectivity tests for local and external resources

## What I verified

- Client-to-server reachability on 10.10.10.0/24
- Internet access through the server's NAT path
- DNS resolution from the client
- SSH from client to server
- Apache content reachable from the client
- Packet capture on the server's internal interface

One of the useful parts of this lab was being able to trace the same connection across layers: client route, server forwarding decision, NAT, DNS, service reachability, and finally the packets themselves in tcpdump.

## Hybrid cloud extension

I am now extending this lab into AWS with a real IPsec Site-to-Site VPN.

The Ubuntu Server is becoming the on-premises VPN endpoint with strongSwan, while an AWS Virtual Private Gateway provides the cloud side. The local network is `10.10.10.0/24` and the AWS VPC is `10.20.0.0/16`.

Current work includes:

- strongSwan IPsec on the Ubuntu router
- AWS Virtual Private Gateway
- AWS Customer Gateway
- static Site-to-Site VPN routing
- AWS VPC routing and security controls
- end-to-end packet-flow validation

[Read the hybrid VPN build notes →](docs/hybrid-cloud-vpn.md)

## Tools

Ubuntu Server · Ubuntu Desktop · VirtualBox · Netplan · nftables · OpenSSH · Apache · tcpdump · curl · strongSwan · AWS VPC · AWS Site-to-Site VPN

## Example service test

From the client:

~~~bash
curl http://10.10.10.1
~~~

That request verifies more than Apache alone: the client address, local route, internal link, and server service all have to be working.

## What I learned

This was one of my earlier hands-on networking labs, but it gave me a foundation I still use in newer projects:

- A default gateway is a forwarding decision, not just a field in a settings screen.
- NAT and routing solve different problems.
- DNS success does not prove general network health, and network reachability does not prove DNS health.
- Packet capture is often the fastest way to separate what I think the network is doing from what it is actually doing.

The hybrid extension is adding a new lesson: a VPN can be healthy as a service while still having no configured or established tunnel. IKE, IPsec policy, routing, firewall behavior, and the return path all have to line up.

## Where it connects now

The same ideas show up in my newer work:

- [Hybrid Cloud VPN Extension](docs/hybrid-cloud-vpn.md) — strongSwan, IPsec, AWS Site-to-Site VPN, hybrid routing
- [Mini Internet](https://github.com/PatienceEnoch/mini-internet) — BGP, alternate paths, convergence, failure testing
- [Network Flight Recorder](https://github.com/PatienceEnoch/network-flight-recorder) — state capture, diagnosis, incident evidence, recovery verification
- [Cloud Network Architecture Journal](https://github.com/PatienceEnoch/Cloud-Network-Architecture-Journal) — architecture and routing notes
