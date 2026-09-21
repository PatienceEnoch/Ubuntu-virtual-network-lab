# Hybrid Cloud VPN Extension

I am extending the original Ubuntu virtual network lab into a hybrid network that connects my local routed lab to AWS over an IPsec Site-to-Site VPN.

## Goal

Build a working path between a local Ubuntu network and an AWS VPC, then verify the routing, encryption, and failure behavior instead of treating the VPN as a black box.

## Topology

~~~text
Ubuntu Client
10.10.10.10/24
      |
      | internal lab network
      |
Ubuntu Server
10.10.10.1/24
strongSwan + nftables
      |
      | IPsec Site-to-Site VPN
      |
AWS Virtual Private Gateway
      |
hybrid-cloud-lab-vpc
10.20.0.0/16
~~~

The Ubuntu Server is still the router for the local client. The new piece is an encrypted IPsec path from that router into AWS.

## Local validation milestone

Before using the AWS tunnel, I built and completed a local site-to-site IPsec version of the path with a second Linux gateway and a simulated cloud workload.

That validation covered the pieces I wanted to understand before adding cloud-provider routing:

- IKEv2 and CHILD SA establishment
- XFRM policy and state
- nftables forwarding and VPN NAT exemption
- return-route troubleshooting
- packet capture across the cloud-side interface
- Linux network namespaces and veth pairs
- reboot persistence on both gateways

The final local test passed end to end from `10.10.10.10` to `10.20.0.10` with 0% packet loss after rebooting the lab components.

[Read the local site-to-site IPsec validation →](local-site-to-site-ipsec-validation.md)

## Local side

The existing lab already provides:

- Ubuntu Server routing between interfaces
- IPv4 forwarding
- nftables forwarding policy
- NAT for normal Internet access
- Ubuntu Client using 10.10.10.1 as its default gateway
- SSH and packet-level troubleshooting
- strongSwan installed and running for IPsec

The local routed network is:

~~~text
10.10.10.0/24
~~~

At this stage strongSwan is healthy, but no tunnel is established yet. That is expected until the AWS-generated tunnel parameters are added to the server configuration.

## AWS side

The AWS network uses:

~~~text
VPC CIDR: 10.20.0.0/16
Local lab CIDR: 10.10.10.0/24
Routing mode: static
VPN type: AWS Site-to-Site VPN
~~~

Resources created so far:

- VPC for the AWS side of the lab
- Virtual Private Gateway attached to the VPC
- Customer Gateway representing the on-premises/home VPN endpoint
- Site-to-Site VPN connection using static routing

I am intentionally leaving public endpoint addresses, pre-shared keys, tunnel inside addresses, and AWS resource IDs out of the repository.

## Why IPsec matters here

IPsec provides the encrypted and authenticated path across the public Internet.

The local addresses such as 10.10.10.1 and 10.10.10.10 are private addresses and are not directly reachable from AWS over the Internet. The AWS Customer Gateway represents the public-facing side of the local VPN endpoint, while strongSwan handles the IPsec tunnel on the Ubuntu Server.

## What I am testing

This lab is meant to make hybrid connectivity observable rather than just functional.

I want to verify:

- tunnel establishment
- route selection between 10.10.10.0/24 and 10.20.0.0/16
- nftables forwarding behavior
- encrypted traffic crossing the tunnel
- AWS route-table behavior
- security-group effects
- packet flow from the Ubuntu Client into AWS
- tunnel failure and recovery behavior

## Current stopping point

The AWS VPN connection is being provisioned.

The next steps are:

1. Read the AWS tunnel details.
2. Configure the AWS-generated tunnel parameters in strongSwan.
3. Add the required AWS route-table entry for 10.10.10.0/24.
4. Verify security-group rules for test traffic.
5. Bring up the tunnel.
6. Confirm the IPsec Security Association.
7. Test traffic between the Ubuntu lab and an AWS workload.
8. Capture evidence with strongSwan status output, routing tables, nftables counters, and packet captures.

## What this adds to the original lab

The original lab taught me how a Linux host forwards and translates traffic locally.

This extension adds another boundary:

**local routing -> NAT/public Internet -> IPsec -> AWS routing -> cloud workload**

That gives me a way to troubleshoot the same packet across Linux, VPN, and cloud networking layers instead of learning each one separately.
