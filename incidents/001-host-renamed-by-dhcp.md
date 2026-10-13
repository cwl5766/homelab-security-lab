# Incident 001: Proxmox host renamed itself, stranding every VM

| | |
| --- | --- |
| Date found | 2026-10-13 |
| Date it started | 2026-10-02 |
| Impact | VMs kept running, but could no longer be managed from the web UI; the next reboot would likely have left all VMs unstartable |
| Data loss | None |
| Time to resolve | About 30 minutes |

## Symptom

The Proxmox web UI showed two nodes instead of one:

* `nas`, the real host name, holding every VM but none of the storage
* `aabbccddeeff`, an unknown node holding the storage pools and the default SDN zone

Opening a shell on the unknown node failed:

```
Connection failed (Error 500: hostname lookup 'aabbccddeeff' failed -
failed to get address info for: aabbccddeeff: Name or service not known)
```

The unknown name matched the server's network card MAC address with the colons removed (shown here as a placeholder).

## Diagnosis

Since the web shell was broken, I connected over SSH through Tailscale and only ran read only commands first.

```
$ hostname
aabbccddeeff
$ cat /etc/hostname
nas
$ ls /etc/pve/nodes/
aabbccddeeff  nas
$ ls /etc/pve/nodes/*/qemu-server/
/etc/pve/nodes/aabbccddeeff/qemu-server/:

/etc/pve/nodes/nas/qemu-server/:
100.conf  101.conf  102.conf  105.conf  106.conf
```

What this showed:

* The **saved** host name (`/etc/hostname`) was still `nas`, but the **running** host name had changed.
* Proxmox stores each VM's config in a folder named after the host. When the running name changed, Proxmox created a new, empty node folder and treated the VMs' real folder as a different machine.
* `/etc/hosts` was also wrong: it mapped `nas` to `192.168.100.2`, the Proxmox installer's fallback address from the very first install attempt, not the real LAN address. The current name was not in it at all, which is why the web shell's lookup failed.

The system journal pinned down when and why:

```
Oct 02 15:27:55 nas systemd[1]: Hostname set to <nas>.
...
Oct 02 17:02:35 aabbccddeeff tailscaled: trample: resolv.conf changed ... current contents: "nameserver <router>"
Oct 02 17:02:35 aabbccddeeff kernel: audit: ... profile="/{,usr/}sbin/dhclient" ... comm="dhclient"
```

The host booted as `nas`. About 90 minutes later the DHCP client renewed its lease, rewrote `/etc/resolv.conf` with the router's DNS server, and from that moment the host logged under its MAC based name. `dpkg -l` showed two DHCP clients installed (`isc-dhcp-client` and `dhcpcd-base`).

## Root cause

The Proxmox host's management bridge (`vmbr0`) was configured for **DHCP**. The mesh router labels devices it has no name for by their MAC address, and a lease renewal handed that label back to the host, which applied it as its running host name.

A second, latent problem made it worse: the stale `/etc/hosts` entry meant Proxmox could not resolve its own new name.

Why VMs 100 and 105 kept working: they were already running when the name changed, so their processes were unaffected. Starting, stopping or rebooting would have failed, and after a host reboot `qm list` would likely have shown no VMs at all.

## Fix

Running VMs were not touched at any point. Restarting the Proxmox services does not stop guests.

1. Back up and correct `/etc/hosts` so the name points at the real address:
   ```bash
   cp /etc/hosts /etc/hosts.bak
   sed -i 's/^192\.168\.100\.2 /<LAN IP> /' /etc/hosts
   ```
2. Set the running name back:
   ```bash
   hostnamectl set-hostname nas
   ```
3. Restart the Proxmox services so they pick up the name:
   ```bash
   systemctl restart pve-cluster pvedaemon pveproxy pvestatd
   ```
4. Confirm in the web UI that all VMs and all storage appear under `nas`. The empty node folder was cleaned up automatically by the restart.

## Prevention

Switched the host from DHCP to a **static IP** (Datacenter, nas, System, Network, vmbr0), keeping the same address and gateway, and added a matching reservation on the router so nothing else can take the address.

Verification:

```
$ grep -A3 "auto vmbr0" /etc/network/interfaces
auto vmbr0
iface vmbr0 inet static
        address <LAN IP>/22
        gateway <router IP>
$ pgrep -a dhclient
(no output: no DHCP client running)
$ ping -c 3 1.1.1.1
3 packets transmitted, 3 received, 0% packet loss
```

With no DHCP client on the host, nothing can rename it or overwrite its DNS settings again, which also ended the DNS tug of war between the DHCP client and Tailscale seen in the logs.

As a related hardening step, Start at boot was enabled for the always on VMs so they recover on their own after a reboot.

## Lessons

* **A hypervisor's management interface should have a static IP.** Proxmox ties its cluster filesystem to the host name, so anything that changes the name changes where Proxmox looks for VM configs.
* **Check `/etc/hosts` after install.** An installer fallback address can survive long after the network is fixed.
* **Diagnose read only first.** Every command before the fix only read state, so the investigation could not make things worse.
* **Logs answer "when" and "why."** The journal showed the exact minute the name changed and which process was active at that moment.
