# NordVPN Preventing Access to Internal VLANs

## Issue

While configuring AdGuard Home, I suddenly lost access from my Windows PC to devices on the Management VLAN.

The affected services included:

- AdGuard Home
- Uptime Kuma
- Proxmox

At first, I suspected an inter-VLAN routing or ACL issue.

## Environment

- Windows 11
- TP-Link ER605
- Omada Controller
- NordVPN
- AdGuard Home
- Uptime Kuma
- Proxmox

## Symptoms

From the Windows PC, I tested the Management VLAN gateway.

**Command:** `ping 192.168.50.1`

**Result:** Request timed out.

I then tested the AdGuard Home server.

**Command:** `ping 192.168.50.10`

**Result:** Request timed out.

I also ran a traceroute.

**Command:** `tracert 192.168.50.10`

**Result:** The trace timed out.

The PC could not reach any device on VLAN 50.

## Troubleshooting

### 1. Verified local connectivity

I confirmed the Windows PC could still reach its local gateway.

**Command:** `ping 192.168.10.1`

The gateway responded successfully, confirming the PC still had local network connectivity.

### 2. Checked Omada Gateway ACLs

I reviewed the active Gateway ACL rules.

The only blocking rules applied to the IoT and Guest VLANs. No rule was configured to block traffic between the Production VLAN and Management VLAN.

This ruled out the visible ACL configuration.

### 3. Tested from another device

I connected my phone to the Production Wi-Fi and attempted to access the same Management VLAN services.

The phone successfully reached:

- AdGuard Home
- Uptime Kuma

This confirmed that:

- VLAN 50 was online
- Inter-VLAN routing was working
- The servers were available
- The problem was isolated to the Windows PC

### 4. Reviewed the Windows routing table

I ran:

`route print`

The routing table showed two default routes.

- NordVPN gateway: `10.5.0.1`, metric `6`
- Local ER605 gateway: `192.168.10.1`, metric `25`

Because the NordVPN route had the lower metric, Windows preferred it over the local gateway.

### 5. Identified the virtual adapter

I ran:

`ipconfig /all`

A virtual adapter using the address `10.5.0.2` was present.

The adapter belonged to NordVPN.

## Root Cause

NordVPN created a virtual network adapter and installed a lower-metric default route.

Windows sent traffic for the Management VLAN through the VPN tunnel instead of the local ER605 gateway.

The VPN did not have a valid route to the internal VLANs, so the traffic failed.

## Resolution

I disconnected NordVPN.

After disconnecting, the Windows PC could immediately reach:

- `192.168.50.1`
- `192.168.50.10`
- AdGuard Home
- Uptime Kuma
- Proxmox

No router, ACL, VLAN, switch, or server configuration changes were required.

## Commands Used

- `ipconfig`
- `ipconfig /all`
- `route print`
- `ping 192.168.10.1`
- `ping 192.168.50.1`
- `ping 192.168.50.10`
- `tracert 192.168.50.10`

## Outcome

The issue was resolved by disconnecting NordVPN.

The network infrastructure was working correctly. The failure was caused by a VPN route conflict on one Windows workstation.
