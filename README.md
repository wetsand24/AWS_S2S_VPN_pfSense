# pfSense to AWS site-to-site VPN

Every setting from the video, in order: an IPsec tunnel from pfSense at home to an AWS VPC, with static routing.

▶️ Video: https://youtu.be/p83RmeT2Q-A

Newer Azure version with BGP: https://youtu.be/0HYaSX_fvBE

## What you need
- pfSense CE on hardware that meets the minimum requirements: 64-bit amd64 (x86-64) CPU, 1 GB or more RAM, 8 GB or larger disk, one or more compatible network cards
- An AWS account
- The public IP of your pfSense WAN

> **Cost:** AWS charges an hourly fee for a Site-to-Site VPN connection while it's active ($0.05 per hour in US East (Ohio)). The EC2 instance is billed too. Delete the VPN connection when you're done testing.

## Addressing used in the video

Change these to yours.

| What | Value |
|---|---|
| Home server VLAN behind pfSense | 10.37.0.0/20 (pfSense 10.37.0.1, test PC 10.37.0.98) |
| AWS VPC and subnet | 192.168.0.0/24 |
| AWS test instance | 192.168.0.230 |
| AWS region | US East (N. Virginia), us-east-1 |

## 1. Create the VPC
VPC → Your VPCs → Create VPC:
- Name tag = `Azure-Net`
- IPv4 CIDR = `192.168.0.0/24`

Create VPC.

## 2. Create the subnet
VPC → Subnets → Create subnet:
- VPC ID = `Azure-Net`
- Subnet name = `Azure-subnet`
- IPv4 CIDR block = `192.168.0.0/24`

Create subnet.

## 3. Internet gateway and route table
VPC → Internet gateways → Create internet gateway:
- Name tag = `azure-internet`

Create, then Actions → Attach to VPC → `Azure-Net`.

VPC → Route tables → the VPC's route table → Actions → Edit routes → Add route:

| Destination | Target |
|---|---|
| 0.0.0.0/0 | the internet gateway (`igw-...`) |

Save changes.

## 4. Launch the EC2 instance
EC2 → Instances → Launch instances:

| Field | Value |
|---|---|
| Name | `AWS Server` |
| Image | Ubuntu Server 22.04 LTS, 64-bit (x86) |
| Instance type | t2.micro |
| Key pair | Create new key pair, name `key`, download it |
| Network settings → Edit → VPC | `Azure-Net` |
| Subnet | `Azure-subnet` |
| Auto-assign public IP | Enable |
| Firewall (security groups) | Create security group (`launch-wizard-1`) |
| Storage | default (8 GiB) |

Launch instance. When it shows **Running**, select it → Connect → EC2 Instance Connect → Connect. Note its private IP (192.168.0.230 in the video).

## 5. Customer gateway
VPC → Customer gateways → Create customer gateway:
- Name tag = `pfsense`
- BGP ASN = `65000` (default, not used with static routing)
- IP address = `YOUR-PUBLIC-IP` (pfSense WAN public IP)

Create customer gateway. Wait until the state is **available**.

## 6. Virtual private gateway
VPC → Virtual private gateways → Create virtual private gateway:
- Name tag = `Virtual gateway`
- ASN = Amazon default ASN

Create virtual private gateway.

## 7. Site-to-site VPN connection
VPC → Site-to-Site VPN connections → Create VPN connection:

| Field | Value |
|---|---|
| Name tag | `pfsense` |
| Target gateway type | Virtual private gateway |
| Virtual private gateway | `Virtual gateway` |
| Customer gateway | Existing → `pfsense` |
| Routing options | Static |
| Static IP prefixes | `10.37.0.0/20` (your pfSense LAN/VLAN) |
| Local IPv4 network CIDR | `10.37.0.0/20` |
| Remote IPv4 network CIDR | `192.168.0.0/24` (the VPC) |

Create VPN connection. It takes about 5 to 10 minutes to become **available**.

## 8. Download the VPN configuration
Select the VPN connection → Download configuration:
- Vendor = Generic
- Platform = Generic
- Software = Vendor Agnostic
- IKE version = ikev2

Download. From the file you need, for **tunnel 1** only:
- Outside IP of the **Virtual Private Gateway** (goes into pfSense as Remote Gateway)
- **Pre-Shared Key**
- **Diffie-Hellman** group (Group 2 in the video's file)

The file has settings for two tunnels. The video only builds tunnel 1.

## 9. pfSense IPsec phase 1
VPN → IPsec → Tunnels → Add P1:

| Field | Value |
|---|---|
| Description | `AWS - P1` |
| Key Exchange version | IKEv2 |
| Interface | WAN |
| Remote Gateway | `YOUR-AWS-TUNNEL1-IP` (Virtual Private Gateway outside IP from the file) |
| Authentication Method | Mutual PSK |
| My identifier | My IP address |
| Peer identifier | Peer IP address |
| Pre-Shared Key | `YOUR-PSK` (from the file) |
| Encryption Algorithm | AES, 128 bits, SHA256, DH Group 2 (1024 bit) |
| Life Time | 28800 |

Everything else default. Save, Apply Changes.

pfSense marks DH group 2 as weak on this page. The video uses it because it's the group in the AWS file.

## 10. pfSense IPsec phase 2
Show Phase 2 Entries → Add P2:

| Field | Value |
|---|---|
| Description | `AWS-P2` |
| Mode | Tunnel IPv4 |
| Local Network | your LAN/VLAN subnet (`SERVERS subnet` in the video) |
| Remote Network | Network, `192.168.0.0` / `24` |
| Protocol | ESP |
| Encryption Algorithms | AES 128 bits, AES128-GCM 128 bits (defaults) |
| Hash Algorithms | SHA256 |
| PFS key group | 14 (2048 bit) (default) |
| Life Time | 3600 |

Save, Apply Changes.

## 11. Connect the tunnel
Check the VPN connection in AWS shows **available**. In pfSense: Status → IPsec → Connect. The tunnel shows **Established**.

## 12. Ping test and Windows Firewall
From the AWS instance:
```bash
ping 10.37.0.98
```
It fails until the PC at home allows the AWS network. On the Windows PC: Windows Defender Firewall with Advanced Security → Inbound Rules → New Rule:
- Rule Type = Custom
- Program = All programs
- Protocol and Ports = Any
- Scope → Remote IP address → This IP address or subnet = `192.168.0.0/24`
- Action = Allow the connection
- Name = `AWS`

Finish.

## 13. Attach the VGW and add the route
The ping still fails until the virtual private gateway is attached and the VPC knows the way home.

VPC → Virtual private gateways → `Virtual gateway` → Actions → Attach to VPC → `Azure-Net` → Attach.

VPC → Route tables → the VPC's route table → Edit routes → Add route:

| Destination | Target |
|---|---|
| 10.37.0.0/20 | Virtual private gateway (`vgw-...`) |

Save changes. Give it a few minutes. Then check: Site-to-Site VPN connections → `pfsense` → Tunnel details (tunnel is up) and Static routes (10.37.0.0/20). The ping from AWS to home now answers.

## 14. AWS security group (ICMP)
From home to AWS, `ping 192.168.0.230` fails until the security group allows it.

VPC → Security groups → `launch-wizard-1` → Edit inbound rules → Add rule:

| Type | Source |
|---|---|
| All ICMP - IPv4 | Custom, `10.37.0.0/20` |

Save rules.

## 15. Working
```
ping 192.168.0.230
```
Pings now work both ways.

## Links
- Same setup with Azure: https://youtu.be/sEgKqQ8Gm3Y
- Azure with BGP (2026): https://youtu.be/0HYaSX_fvBE
- Tailscale on pfSense: https://youtu.be/HDFw4EVDZj8
