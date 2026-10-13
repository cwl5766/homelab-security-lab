# Design decisions for the parts not built yet

These are choices made while planning the blue team phases (Active Directory, Sysmon, traffic mirroring and Security Onion). They come from research and plan reviews, not from running the systems yet. Once a part is built, each decision either holds up or gets corrected, and anything learned along the way moves to [lessons learned](lessons-learned.md).

Each entry has the decision, the reasoning, and its status.

## Active Directory

### Name the domain `ad.test`, not `lab.local`

**Decision:** The domain will be `ad.test` (NetBIOS name `AD`). The first draft of the plan used `lab.local`.

**Why:** `.local` is reserved for automatic device discovery on home networks (mDNS, the protocol that finds printers and Apple devices without any setup). A Windows domain using `.local` competes with that and can cause intermittent name lookup failures. `.test` is officially reserved for testing (RFC 6761), so it can never clash with a real internet domain either.

**Status:** Planned.

### Use a SATA disk and an E1000 network card for the domain controller

**Decision:** The domain controller VM gets older style virtual hardware instead of the faster VirtIO devices.

**Why:** Windows doesn't include VirtIO drivers, so the installer can't see a VirtIO disk without loading extra drivers mid install. A lab domain controller doesn't need the extra speed, and the same approach already works for Metasploitable 2.

**Status:** Planned. The VirtIO driver ISO is still needed for one thing: the guest agent, which lets Proxmox shut the VM down cleanly.

### Turn on advanced audit logging on day one

**Decision:** Right after the domain is created, enable advanced audit policy (logons, account management, process creation with command lines, policy changes, privilege use) and enlarge the Security log.

**Why:** By default a domain controller records very little worth analyzing. Without this, the later detection work would have almost nothing to find.

**Status:** Planned.

## Getting files into the air gapped lab

### Bring files in on an ISO, never by moving a VM onto the home network

**Decision:** Tools like Sysmon will be packaged into a small ISO image on the host and attached to the VM like a CD.

**Why:** The lab is safe because it has no path to the internet or the family network. Moving a VM onto the home network "just for a few minutes" breaks that, and for the domain controller it wouldn't even work, because its static address has no gateway.

**Status:** Planned. Kali is the one exception: it needs the package manager to install software, so it will briefly move networks with its settings switched to match.

## Network monitoring

### Choose Security Onion as the SIEM

**Decision:** Security Onion over Wazuh.

**Why:** It bundles the network tools that show up most often in SOC job postings (Suricata alerts, Zeek logs, and Elastic and Kibana for searching), on top of collecting host logs.

**Status:** Planned.

### Install Security Onion 3.0, not 2.4

**Decision:** Start on version 3.0.

**Why:** Version 2.4 reached end of life on October 1, 2026. Moving from 2.4 to 3.0 later would require a fresh install, so starting on 3.0 avoids rebuilding.

**Status:** Planned.

### Convert the lab bridge to Open vSwitch so traffic can be mirrored

**Decision:** Rebuild the lab network (vmbr1) as an Open vSwitch bridge and mirror all its traffic to Security Onion's listening port.

**Why:** Security Onion can only analyze traffic it can see. A standard Linux bridge can't copy ("mirror") traffic between other machines to a monitoring port, while Open vSwitch can.

**Status:** Planned. This is expected to be the hardest phase.

### Recreate the mirror with a hookscript, not a reboot job

**Decision:** A Proxmox hookscript rebuilds the mirror every time the Security Onion VM starts.

**Why:** The mirror is attached to the VM's virtual network port, and Proxmox deletes that port whenever the VM stops. A job that only runs when the host boots would miss every VM restart.

**Status:** Planned.

### Give Security Onion two network cards, with no address on the lab side

**Decision:** One card on the home network for its dashboard, one on the lab network that only listens.

**Why:** This is the only machine with a connection to both networks, so its lab side has no IP address and can't send anything. It receives mirrored traffic and nothing else, which keeps the air gap intact.

**Status:** Planned.
