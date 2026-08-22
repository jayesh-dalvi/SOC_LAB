# Troubleshooting: AD-Joined Workstation Unreachable via Computer Management

**Lab:** SOC_LAB (VirtualBox — Windows AD, Kali Linux, Ubuntu/Wazuh)
**Environment:** `corp.local` domain — `DCG-DC01` (Domain Controller) managing `DCG-WS01` (Windows 10 workstation)
**Date:** August 2026

## Summary

While attempting to manage `DCG-WS01` remotely from `DCG-DC01` using **Computer Management**, the connection failed with a firewall-related error. Investigation showed the real problem was not the firewall at all — it was a chain of three separate network misconfigurations that had to be diagnosed and fixed one at a time.

## The Initial Symptom

Opening **Computer Management** on `DCG-DC01` and connecting to `DCG-WS01.corp.local` returned:

> *"Computer DCG-WS01.corp.local cannot be managed. Verify that the network path is correct, the computer is available on the network, and that the appropriate Windows Firewall rules are enabled on the target computer."*

The message pointed specifically at Windows Firewall (COM+ Network Access, Remote Event Log Management), which is a reasonable first guess — but turned out to be a red herring. Windows shows this same generic message even when the target machine is completely unreachable on the network.

## Diagnosis

### Step 1 — Confirm basic connectivity

Before touching any firewall settings, a basic `ping` test was run from `DCG-DC01` to rule out (or confirm) a network-layer problem first:

```powershell
ping DCG-WS01.corp.local
```

![Ping fails with Destination host unreachable](images/01-ping-fails-destination-unreachable.png)

DNS resolved the hostname to an IP (`192.168.100.30`), but every reply came back **"Destination host unreachable"** from the gateway itself — meaning there was no network route to the target at all. This confirmed the firewall message was misleading; the real issue was connectivity, not inbound rules.

### Step 2 — Check the VirtualBox network adapter

With a routing-level failure confirmed, the next place to check was whether the VM's virtual NIC was even active. In VirtualBox Settings → Network for `DCG-WS01`, **"Enable Network Adapter" was unchecked** — the adapter was fully disabled at the hypervisor level, despite Windows still holding a static IP configuration internally.

**Fix:** Ticked "Enable Network Adapter," saved settings, and power-cycled the VM (a running VM needs a restart for this change to take effect in VirtualBox).

### Step 3 — Check DNS and IP configuration on the workstation

With the adapter enabled, the VM came back online — but running `ipconfig /all` on `DCG-WS01` revealed two further problems:

![DCG-WS01 wrong DNS server and DHCP-assigned IP](images/02-ws01-wrong-dns-and-dhcp-ip.png)

| Setting | Found | Should be |
|---|---|---|
| DNS Servers | `192.168.1.1` (generic router) | `192.168.100.10` (the DC) |
| IPv4 Address | `192.168.100.4` (via DHCP) | `192.168.100.30` (matching the existing AD DNS record) |

Pointing a domain-joined machine at the wrong DNS server breaks AD-integrated name resolution and SRV/Kerberos lookups. Separately, AD DNS still had `DCG-WS01` registered against its old IP (`192.168.100.30`), while DHCP had since handed the machine a different address (`192.168.100.4`) — a classic stale-record mismatch.

**Fix (part 1) — correct the DNS server**, run as Administrator on `DCG-WS01`:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.10
```

> **Note:** This command requires an **elevated** PowerShell session. Running it as a standard user returns `Access to a CIM resource was not available to the client` (`PermissionDenied`, `CimException`).

![DNS corrected to point at the DC](images/03-ws01-dns-corrected-to-dc.png)

**Fix (part 2) — resolve the IP mismatch.** Rather than forcing AD DNS to update to the DHCP-assigned address, the workstation's IP was set statically to match the IP AD DNS already expected, giving `DCG-WS01` a stable, predictable address going forward:

![Final correct static IP and DNS configuration](images/04-ws01-static-ip-set-matching-dns-record.png)

- IPv4 Address: `192.168.100.30` (static, DHCP disabled)
- DNS Servers: `192.168.100.10` (the DC)
- Default Gateway: `192.168.100.1`

### Step 4 — Re-test

With all three issues resolved, the ping from `DCG-DC01` was re-run:

```powershell
ping DCG-WS01.corp.local
```

![Ping succeeds with real replies](images/05-ping-succeeds-after-fix.png)

Real replies (`bytes=32 time<1ms TTL=128`, 0% loss) confirmed the workstation was fully reachable. Computer Management was then able to connect to `DCG-WS01` without any further firewall changes required.

## Root Causes (in order discovered)

1. **VirtualBox network adapter disabled** on the target VM — no network path existed at all, regardless of any OS-level configuration.
2. **DNS server pointing at the wrong host** (a generic router address instead of the domain controller) — broke AD-integrated name resolution.
3. **Stale AD DNS record vs. actual DHCP-assigned IP** — the hostname resolved to an IP the machine wasn't actually using.

## Key Takeaway

The Windows Firewall error shown by Computer Management is generic and can appear even when the real problem is that the target machine is unreachable on the network. **Always verify basic connectivity (`ping`, `ipconfig /all`) before troubleshooting firewall rules** — working top-down from network reachability, to DNS resolution, to IP configuration, to firewall rules avoids chasing the wrong fix first. This mirrors a realistic diagnostic order a SOC analyst or IT support technician would follow for any "can't manage this remote host" ticket.
