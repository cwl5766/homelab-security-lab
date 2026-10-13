# Lessons learned

Every lesson here comes from something that actually happened in this lab, and each one cost real time at least once. Decisions made for parts that are not built yet live in [design decisions](design-decisions.md) until they are tested for real. Each one has a one line summary, then three short parts:

* **What happened:** the situation in this lab.
* **Why it matters:** the concept behind it, explained without assuming background knowledge.
* **What to do:** the practical takeaway.

## Contents

* [Proxmox host](#proxmox-host)
* [Storage](#storage)
* [Networking and remote access](#networking-and-remote-access)
* [Applications](#applications)
* [Game server](#game-server)
* [Linux basics](#linux-basics)
* [Working method](#working-method)

---

## Proxmox host

### Give the host a static IP

**What happened:** The server got its address automatically from the home router (DHCP). About 90 minutes after a reboot, a routine lease renewal also handed it a new name: its network card's hardware address. Nobody noticed for eleven days. Proxmox suddenly saw two machines, and every VM was stranded under the old name. Full writeup: [incident 001](../incidents/001-host-renamed-by-dhcp.md).

**Why it matters:** DHCP is great for phones and laptops, but a server that other things depend on should never change its identity. Proxmox stores each VM's settings in a folder named after the host, so changing the name changes where it looks for its VMs.

**What to do:** Give the hypervisor a fixed (static) address in its own network settings, and reserve that same address on the router so no other device can take it.

### Check `/etc/hosts` after installing

**What happened:** During the first install attempt, the network didn't come up, so the installer filled in a placeholder address (192.168.100.2). The network was fixed later, but that placeholder stayed in `/etc/hosts` for months.

**Why it matters:** `/etc/hosts` is a small file that tells a machine "this name means this address." Proxmox uses it to find itself. A wrong entry sits silently until something else goes wrong, then makes the problem much worse.

**What to do:** After any install, open `/etc/hosts` and confirm the server's name points at its real address.

### A brand new network chip may run on a fallback driver

**What happened:** The motherboard's 5 gigabit network port is a very new Realtek chip. Linux didn't have its specific driver yet, so it loaded a generic one (`r8169`) instead.

**Why it matters:** A driver is the software that lets the operating system talk to hardware. A generic driver often works, but it may run the hardware slower than it's capable of, or behave unreliably.

**What to do:** Check the link speed (`ethtool <interface>`) rather than assuming the full rate, and install the proper driver when it's convenient.

### Without a subscription, the Enterprise repositories return 401

**What happened:** The first system update failed with "401 Unauthorized."

**Why it matters:** Proxmox is free, but by default it points updates at a paid "Enterprise" source. Without a subscription key, that source refuses the request.

**What to do:** Disable the Enterprise repositories and enable the free "No Subscription" one. Updates then work normally.

### A custom Proxmox role needs VM.Audit

**What happened:** A limited account was created for console access to the lab VMs only. After logging in as that account, the VMs didn't appear at all.

**Why it matters:** Permissions in Proxmox are fine grained. VM.Console lets you open a screen, and VM.PowerMgmt lets you start and stop, but VM.Audit is what lets you *see* that a VM exists. Without seeing it, the other permissions are useless.

**What to do:** For a "console only" role, grant VM.Audit, VM.Console and VM.PowerMgmt, and scope it to just the VMs that account should reach. This is least privilege: exactly enough access, nothing more.

### The Qemu Agent checkbox does nothing on its own

**What happened:** The "Qemu Agent" box was ticked when creating VMs, but Proxmox still couldn't talk to some of them properly.

**Why it matters:** The checkbox only opens a channel between Proxmox and the VM. A small program, the guest agent, has to be installed and running *inside* the VM to answer on that channel. It enables clean shutdowns and accurate backups.

**What to do:** Install `qemu-guest-agent` inside each Linux VM. On Windows, install it from the VirtIO driver ISO. If you tick the box but never install the agent, shutdowns can hang while Proxmox waits for a reply.

### Old operating systems need old virtual hardware

**What happened:** Metasploitable 2, a deliberately vulnerable Linux from 2008, misbehaved with the modern virtual disk and network card that VMs use by default.

**Why it matters:** VMs pretend to have hardware. The fast modern kind (VirtIO) needs drivers that old systems simply don't have. Older "emulated" hardware is slower but universally understood.

**What to do:** For old guests, use a SATA disk and an Intel e1000 network card. The same trick avoids driver hassles when installing Windows.

### Restarting the Proxmox services does not stop running VMs

**What happened:** Fixing the host name meant restarting Proxmox's own management services while Nextcloud and the apps VM were still running.

**Why it matters:** Proxmox is split into management services (the web interface, the config database) and the VMs themselves. The VMs keep running even when the management layer restarts.

**What to do:** Many host level repairs can be done without downtime. Still snapshot or back up first, and never reboot the host until the fix is confirmed.

### Use Shutdown, not Stop

**What happened:** The early question was simply "how do I turn a VM off?" Proxmox offers two buttons that sound alike.

**Why it matters:** Shutdown politely asks the operating system inside to close everything and save its work. Stop is the same as pulling the power cord, which can corrupt a database that was halfway through writing.

**What to do:** Always use Shutdown. A small script now shuts down every running VM cleanly before the host itself powers off.

### VMs do not start on their own after a host reboot

**What happened:** After the server restarted, the family cloud and the apps VM stayed off until someone started them by hand.

**Why it matters:** Proxmox doesn't assume every VM should run all the time, so each one needs to opt in.

**What to do:** Turn on "Start at boot" (Options tab, or `qm set <id> --onboot 1`) for anything that should always be running.

---

## Storage

### Put VM disks on the fast drive and keep the boot drive for the OS

**What happened:** The server has two NVMe drives: a budget one without its own memory cache (DRAMless) and a high end Samsung 990 PRO.

**Why it matters:** VMs read and write constantly, which punishes budget drives. The Proxmox operating system itself barely touches its disk once it's running.

**What to do:** Install Proxmox on the budget drive and put every VM disk on the fast one. Each drive gets the job it's suited for.

### Leave VM disk cache at the default

**What happened:** One suggestion was to turn on "write back" caching to speed up the budget drive.

**Why it matters:** Write back caching holds data in memory and reports it as saved before it actually reaches the disk. If power cuts out, that data is gone, and a database can end up corrupted.

**What to do:** Keep the default (no cache) unless the server sits behind a battery backup (UPS) that can shut it down cleanly.

### Buy CMR drives for ZFS

**What happened:** WD Red Plus drives were chosen deliberately over plain WD Red.

**Why it matters:** Hard drives record data in one of two ways. CMR writes tracks side by side; SMR overlaps them like roof shingles to fit more data, which makes rewriting slow. When a ZFS mirror rebuilds after a drive failure (a "resilver"), SMR drives can slow to a crawl or even fail the rebuild.

**What to do:** Check the recording technology before buying drives for ZFS or any RAID, because some product lines mix both.

### Keep family data off the VM's system disk

**What happened:** Nextcloud's files live on their own 5TB virtual disk carved from the mirrored hard drives, separate from the small disk the VM boots from.

**Why it matters:** If the VM breaks or needs rebuilding, the family's files aren't tangled up with it. It also lets backups treat "the app" and "the data" differently.

**What to do:** Give important data its own disk, and point the application at it when you first set it up. Nextcloud specifically warns not to change the data location later.

### A ZFS virtual disk reserves its full size up front

**What happened:** The storage pool reported about 70% full while holding only a few gigabytes of real files.

**Why it matters:** When Proxmox creates a VM disk on ZFS (a "zvol"), it reserves the entire size immediately by default, so the space is guaranteed to exist later. The pool counts that reservation as used.

**What to do:** Don't panic at the number. Check real usage with `zfs list` and look at the difference between USED and REFER. Keep the reservation if you want the space guaranteed.

### Docker's own storage lives on the VM system disk

**What happened:** Nextcloud's VM was first built with a 32GB system disk, then rebuilt at 64GB.

**Why it matters:** Pointing Nextcloud's *files* at the big data disk doesn't move Docker itself. Container images, databases and old update layers still pile up on the system disk, and a full system disk takes everything down.

**What to do:** Size a Docker host's system disk generously from the start. Growing it later is fiddlier than starting big.

### Prefer bind mounts over named Docker volumes

**What happened:** The apps stack originally used Docker "named volumes," and was switched to folders on disk ("bind mounts") under one directory.

**Why it matters:** Named volumes live in a hidden Docker managed location, so backing up "the app folder" quietly misses the actual data. With bind mounts, everything is in one visible folder.

**What to do:** Use bind mounts for any data you care about, so one folder (or one VM backup) captures it all.

### After editing fstab, check every line starts with `UUID=`

**What happened:** The new data disk failed to mount, with "Can't lookup blockdev."

**Why it matters:** `/etc/fstab` lists the disks to attach at boot. Disks are identified by a unique ID, and the line needs the `UUID=` label in front of that ID. The command used to add the line left the label off.

**What to do:** After editing fstab, test it with `sudo mount -a` *before* rebooting. A bad fstab line can stop a machine from booting.

---

## Networking and remote access

### Never port forward when a mesh VPN will do

**What happened:** Several ways to reach home services from outside were compared: opening ports on the router, Cloudflare Tunnel, and Tailscale.

**Why it matters:** Opening a port puts a service directly on the internet, where it gets scanned and attacked within minutes. A mesh VPN like Tailscale only lets in devices you've approved, and nothing is open on the router at all.

**What to do:** Use Tailscale (or similar) for remote access, and keep the router closed.

### Tailscale access rules add up; they don't override

**What happened:** A narrow rule was added for people the server is shared with, but they could still reach everything.

**Why it matters:** Tailscale grants are additive: if any rule allows something, it's allowed. The default "everyone can reach everything" rule was still there, so the narrow rule changed nothing.

**What to do:** To restrict access, remove the broad default rule entirely, then add specific rules for each group. Here, my own devices get full access, and shared users only reach web ports on Nextcloud or the game port on the game server.

### Email addresses in access rules must match exactly

**What happened:** A rule meant for one person didn't work because of a one letter typo in their email address.

**Why it matters:** Access rules match identities literally. There is no "close enough," and a typo fails silently.

**What to do:** Copy addresses from the admin console instead of typing them, and test the access after saving.

### Share invites must be accepted from the invite email

**What happened:** A family member was invited to the Nextcloud machine but signed up at tailscale.com directly. They got their own separate, empty network and couldn't see the shared machine.

**Why it matters:** The invite link connects the new account to the share. A fresh signup creates an unrelated account.

**What to do:** Tell people to click the link in the invite email itself. Use named email invites rather than reusable links, which anyone holding the link could use.

### Tailscale Funnel can't expose only part of a site

**What happened:** One idea was to share Nextcloud links publicly through Tailscale Funnel while keeping the rest private.

**Why it matters:** Funnel publishes a service to the whole internet. Splitting "public share links" from "private login pages" by URL path doesn't work reliably, so turning it on would effectively expose the entire Nextcloud instance.

**What to do:** Keep everything private behind Tailscale. Give each person a real account and a scoped invite.

### An exit node needs more than the toggle

**What happened:** The server was made a Tailscale exit node (a private VPN for travel), but phones connected to it had no internet.

**Why it matters:** An exit node forwards your traffic out through home. That needs the server allowed to forward traffic (IP forwarding), a rule that rewrites addresses on the way out (NAT, saved so it survives reboots), and DNS set up in the tailnet. A shared user also needs explicit permission to use the internet through it.

**What to do:** Treat "exit node" as a small router setup: forwarding, NAT, DNS, and an access grant for anyone who isn't a full member.

### Two services can't share port 443

**What happened:** Nextcloud's setup container and Tailscale both tried to listen on port 443 (the standard HTTPS port), producing a confusing TLS error.

**Why it matters:** Only one program can listen on a given port at a time. The second one fails, or traffic goes to the wrong one.

**What to do:** Let one service own the port and route the other behind it. Here, Nextcloud listens only locally and Tailscale publishes it over HTTPS.

### VM addresses on the home network can change

**What happened:** The router gave VMs new addresses after restarts, breaking saved bookmarks and SSH commands.

**Why it matters:** Routers hand out addresses from a pool, with no promise that a device gets the same one twice.

**What to do:** Reserve addresses in the router for anything you rely on, and use Tailscale addresses (which don't change) for remote logins.

### Dynamic DNS solves a different problem

**What happened:** Dynamic DNS came up as a possible fix for changing addresses.

**Why it matters:** Dynamic DNS keeps a domain name pointed at your home's *public* address, which your internet provider may change. It does nothing for *internal* addresses inside the house.

**What to do:** Use a DHCP reservation or a static IP for internal addresses. Dynamic DNS is only needed if you expose something to the internet, which this lab deliberately doesn't.

### A bridge with no ports, no IP and no gateway is a true air gap

**What happened:** The lab network (vmbr1) was built as a virtual switch connected to nothing outside the server.

**Why it matters:** With no physical port, no address on the host and no gateway, there is no path from the lab to the internet or the family network. Attack tools and vulnerable machines can't leak out.

**What to do:** Prove it rather than assume it: ping between lab VMs (should work), ping the internet from a lab VM (should fail), and watch the traffic with `tcpdump`.

### Nextcloud AIO needs a real domain with a valid certificate

**What happened:** Nextcloud wouldn't finish setting up when given only an IP address.

**Why it matters:** Nextcloud All in One requires HTTPS with a certificate a browser trusts, and certificates are issued to domain names, not bare addresses.

**What to do:** Tailscale can issue a valid certificate for a private name on your tailnet. That gives you trusted HTTPS without exposing anything to the internet.

### Store the Nextcloud AIO passphrase on day one

**What happened:** The setup screen showed a passphrase separate from the admin password.

**Why it matters:** That passphrase unlocks the management interface and is required to restore from Nextcloud's own backups. Lose it, and those backups can't be restored.

**What to do:** Save it in a password manager immediately. Treat it like a backup encryption key, because that's effectively what it is.

### Install Team Folders, not Organization Folders

**What happened:** The wrong app was installed first and threw errors.

**Why it matters:** The two have similar names but are different apps. "Team Folders" is the maintained one for shared group folders.

**What to do:** Check the exact app name and its maintainer before installing.

### Team Folders grants permissions to groups, not people

**What happened:** The goal was for most relatives to see the Extended Family folder read only, while a few people could add to it.

**Why it matters:** The basic permission screen only accepts groups, not individual users.

**What to do:** Create a small group (here, Curators) for the people who need extra rights, and give that group write access alongside the read only group. Managing access by group is also how it's done in companies, because it scales.

### Vaultwarden requires HTTPS, and signups should close immediately

**What happened:** The self hosted password manager wouldn't work over plain HTTP. After the first account was made, open signups were turned off.

**Why it matters:** Password managers refuse insecure connections by design. Open signups would let anyone who can reach the server create an account.

**What to do:** Serve it over HTTPS (Tailscale again), turn signups off right after creating your account, and invite later users individually through the admin panel.

### Point the Bitwarden client at your own server

**What happened:** Logging in kept failing with what looked like a wrong password.

**Why it matters:** The Bitwarden apps connect to Bitwarden's cloud by default. The account only existed on the self hosted server, so the cloud honestly had no record of it.

**What to do:** In the app or extension, set the server URL to your own instance *before* logging in.

### Homepage refuses addresses it doesn't know

**What happened:** The dashboard worked at home but refused to load through its Tailscale address.

**Why it matters:** Homepage only answers requests for hostnames on its allow list (`HOMEPAGE_ALLOWED_HOSTS`), which protects against a class of web attacks.

**What to do:** Add every address you'll open it from to the list, then restart the container.

### Update policy by label

**What happened:** Containers needed different update rules: some safe to update automatically, others holding important data.

**Why it matters:** An automatic update can break a service overnight. For something stateless like a dashboard, that's a minor annoyance. For a password vault, it's a serious problem.

**What to do:** Run Watchtower in "monitor only" mode for everything, and add an opt in label to the one stateless container allowed to update itself. Update the rest by hand, after a snapshot.

### The original Watchtower image is archived

**What happened:** Watchtower looped forever with a "client version too old" error.

**Why it matters:** The original project stopped being maintained, and newer Docker versions dropped support for the old interface it used.

**What to do:** Use a maintained fork, and check whether a project is still alive before depending on it.

### Run `docker compose` from the stack's folder

**What happened:** Docker reported "no configuration file provided."

**Why it matters:** `docker compose` looks for its configuration file in the current folder unless told otherwise.

**What to do:** `cd` into the stack's folder first (here `/opt/apps`).

### Live databases and copy backups don't mix

**What happened:** The first backup idea was to copy the apps' data folder while they were running.

**Why it matters:** Copying a database file mid write can capture it in a half finished state that won't open later.

**What to do:** Back up whole VMs through Proxmox (which coordinates with the guest agent), or stop the containers before copying their data.

---

## Game server

### The modpack installer's filename is part of the install

**What happened:** Renaming the installer file broke the installation.

**Why it matters:** This installer reads the pack ID and version from its own filename, so a renamed file doesn't know what to install.

**What to do:** Keep downloaded installers exactly as named.

### Set Java memory in `user_jvm_args.txt`

**What happened:** Memory settings in the start script were ignored.

**Why it matters:** This server type reads memory settings from a separate file. A `#` at the start of a line turns it into a comment that's ignored.

**What to do:** Set memory there, and make sure the line isn't commented out.

### A clean shutdown under systemd needs three things

**What happened:** Stopping the server service killed it before the world finished saving, risking corruption.

**Why it matters:** systemd, the Linux service manager, has to know which process is the real server and wait for it to exit. If the stop command returns immediately, systemd assumes the job is done and kills whatever is left.

**What to do:** Use a service of `Type=exec`, run the server with `screen -D -m` so systemd tracks the real process, and use a stop script that sends "stop" and then waits until the process has actually exited.

### Client and server versions must match exactly

**What happened:** Players couldn't join after the modpack updated on one side.

**Why it matters:** Modded servers refuse clients whose mod list differs, even slightly.

**What to do:** Update the server and every player's client together, and back up the server folder first.

---

## Linux basics

### A Debian install with a root password has no sudo

**What happened:** `sudo` didn't exist on a fresh Debian VM.

**Why it matters:** If you set a root password during install, Debian assumes you'll use root directly and skips installing `sudo`.

**What to do:** Run `su` to become root, install `sudo`, add your user to the sudo group, then log out and back in.

---

## Working method

### Pasting long commands into a terminal can corrupt files

**What happened:** Pasting several multi line file blocks at once scrambled them together, corrupting a configuration file.

**Why it matters:** Terminals process pasted text as if it were typed, and long pastes can interleave or drop characters.

**What to do:** Create files on your own computer and copy them over with `scp`. Then check them with `cat` before using them.

### Snapshot at every milestone

**What happened:** VMs were snapshotted after each working stage (clean install, configured, and so on).

**Why it matters:** A snapshot is an instant restore point. When the next step goes wrong, you roll back in seconds instead of rebuilding.

**What to do:** Take a snapshot whenever something works, and verify each stage with real output before moving on.

### Diagnose read only first

**What happened:** In incident 001, every command before the fix only *read* the system's state.

**Why it matters:** Changing things while you don't yet understand the problem can destroy the evidence, or make it worse.

**What to do:** Gather facts first (names, configs, logs), form an explanation, and only then change things, one at a time, with a backup.

### Verify claims against current documentation

**What happened:** Several plans changed after checking official docs or getting a second opinion, including the Funnel limitation and the correct Nextcloud app name.

**Why it matters:** Tools change quickly, and advice (from people, forums or AI) is often out of date or slightly wrong.

**What to do:** Check the official documentation for anything you're about to rely on, especially security settings.
