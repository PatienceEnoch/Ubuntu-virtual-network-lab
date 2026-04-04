# Ubuntu-virtual-network-lab
Virtual Ubuntu home lab with routing, NAT, SSH, Apache, and tcpdump packet capture in VirtualBox

## Lab Overview

This lab includes two virtual machines:

- Ubuntu Server Lab
- Ubuntu Client

The Ubuntu Server VM has two network interfaces:
- one connected to NAT for internet access
- one connected to an internal network called `lanlab`

The Ubuntu Client VM is connected only to the internal `lanlab` network and uses the Ubuntu Server VM as its gateway.

## Topology

- Ubuntu Server Lab
  - `enp0s3` = NAT-facing interface
  - `enp0s8` = internal lab interface
  - Internal IP: `10.10.10.1/24`

- Ubuntu Client
  - `enp0s3` = internal lab interface
  - Internal IP: `10.10.10.10/24`
  - Default gateway: `10.10.10.1`

## What I Configured

### Ubuntu Server Lab
- Static IP on internal interface
- IP forwarding enabled
- `nftables` configured for NAT and forwarding
- `openssh-server` installed for remote access
- `apache2` installed for internal web hosting
- `tcpdump` used for packet capture and traffic observation

### Ubuntu Client
- Static IP configuration
- Default gateway pointed to Ubuntu Server Lab
- DNS configured manually
- Tested access to internal and external resources

## Services and Tools Used

- Ubuntu Server
- Ubuntu Desktop
- VirtualBox
- Netplan
- nftables
- OpenSSH
- Apache2
- tcpdump
- curl

## Validation and Testing

The following tests were completed successfully:

- Client could ping the server at `10.10.10.1`
- Client could reach the internet through the server
- Client could resolve DNS successfully
- SSH from client to server worked
- Apache web page hosted on the server loaded from the client
- `tcpdump` captured traffic on the internal server interface

## Example Internal Web Page Test

Apache was configured on the Ubuntu Server VM and tested from the Ubuntu Client VM using:

```bash
curl http://10.10.10.1

Skills Practiced
Linux server administration
Virtual machine networking
Static IP configuration
Routing and NAT
SSH remote access
Web server setup
Packet capture and troubleshooting
Next Steps

Planned improvements for this lab:

Add a second internal server
Configure local DNS for internal hostnames
Add file sharing
Add firewall filtering rules
Run containerized services with Docker
Expand documentation with diagrams and screenshots
Notes

This lab was built to create a realistic hands-on environment for practicing networking and Linux administration in a fully virtual setup.
