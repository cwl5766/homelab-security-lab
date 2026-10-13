# Lessons learned

Each of these cost real time once. They are grouped by area.

## Proxmox host

* **Give the host a static IP.** On DHCP, a router lease renewal renamed the host and stranded every VM config. See [incident 001](../incidents/001-host-renamed-by-dhcp.md).
* **Check `/etc/hosts` after install.** When the installer's DHCP request failed, it fell back to a placeholder address (192.168.100.2), and that address stayed in `/etc/hosts` long after the network was fixed.
* **A brand new network chip may come up on a fallback driver.** The board's Realtek 5GbE port loaded the generic `r8169` driver. It worked for the install, but check link speed before assuming the full rate.
* **Without a subscription, the Enterprise repositories return 401.** Disable them and add the No Subscription repository.
* **A custom Proxmox role needs VM.Audit,** or the VMs do not even appear in the tree for that user.
* **The Qemu Agent checkbox does nothing on its own.** The `qemu-guest-agent` package (or the Windows installer from the VirtIO ISO) has to run inside the guest.
* **Old guests need old virtual hardware.** Metasploitable 2 only works reliably with a SATA disk and an e1000 network card.
* **Restarting the Proxmox services does not stop running VMs,** which makes host level repairs possible without downtime.
* **Use Shutdown, not Stop.** Stop is pulling the power cord on the guest. A script that runs a clean shutdown of every VM before the host powers off avoids that entirely.
* **VMs do not start on their own after a host reboot** unless Start at boot is turned on for each one.

## Storage

* **Put VM disks on the fast NVMe and keep the boot drive for the OS.** The DRAMless boot drive would suffer under VM load; the OS barely notices.
* **Leave VM disk cache at the default.** Write back caching without a UPS risks data loss on a power cut.
* **Buy CMR drives for ZFS.** Some drives in the same product family use SMR, which performs badly during a resilver.
* **Keep family data off the VM's system disk.** Nextcloud's files live on a separate data disk from the ZFS mirror, so the VM can be rebuilt without touching them.
* **A zvol reserves its full size up front.** The pool looked 70% full while holding a few GB of real data.
* **Docker's own storage lives on the VM system disk,** regardless of where Nextcloud's data directory points, so size the system disk generously.
* **Prefer bind mounts over named Docker volumes.** With every container's data under one folder, a backup of that folder really contains everything.
* **After editing fstab, check every line starts with `UUID=`.** A missing prefix caused a mount failure.

## Networking and remote access

* **Never port forward when a mesh VPN will do.** Everything is reachable through Tailscale, and nothing is open on the router.
* **Tailscale grants are additive.** To restrict shared users, remove the broad default grant entirely instead of adding a narrower one beside it.
* **Email addresses in grants must match the account exactly.**
* **A person who signs up fresh on tailscale.com gets a separate tailnet.** They must accept the share from the invite email itself, and use named email invites rather than reusable links.
* **Tailscale Funnel cannot expose only part of a site.** Path based public and private splitting does not work reliably, so turning it on would expose the whole Nextcloud instance. It was ruled out.
* **An exit node needs more than the toggle:** IP forwarding, a persistent NAT rule, and DNS override in the tailnet. Sharing it also needs an explicit grant to `autogroup:internet`.
* **Two services cannot share port 443.** Nextcloud AIO's domain check container collided with Tailscale's HTTPS listener; the fix was to drop AIO's public ports and publish it through `tailscale serve`.
* **VM LAN addresses move.** Use Tailscale addresses for SSH and DHCP reservations for anything you rely on.
* **Dynamic DNS solves a different problem.** It tracks a changing public IP; a changing LAN IP needs a DHCP reservation or a static address.
* **Use `.test` for a lab domain, never `.local`,** which is reserved for mDNS.
* **A Linux bridge with no ports, no IP and no gateway is a true air gap.** Proving it with ping and tcpdump between the lab VMs is worth the few minutes.
* **Keep the air gap closed for lab downloads.** Bring files in on an attached ISO instead of moving a VM onto the home network.

## Applications

* **Nextcloud AIO needs a real domain with a valid certificate.** An IP address alone does not work; Tailscale HTTPS solved this without exposing anything publicly.
* **Nextcloud AIO's passphrase is separate from the admin password** and is required to restore from backup. Store it in the password manager on day one.
* **Install Team Folders, not Organization Folders.** They are different apps and the second is incompatible.
* **Team Folders grants permissions to groups, not people.** A small Curators group was the way to let a few people write to an otherwise read only folder.
* **Vaultwarden requires HTTPS.** Close signups right after creating the first account; invite later people through the admin panel instead of reopening signups.
* **Point the Bitwarden client at your own server.** Logging in at the default cloud address looks like a wrong password when the account only exists on the self hosted instance.
* **Homepage refuses hosts it does not know.** Every address you open it from must be listed in `HOMEPAGE_ALLOWED_HOSTS`.
* **Update policy by label:** Watchtower set to monitor only globally, with an opt in label on the one stateless container that may auto update. Stateful services get updated by hand after a snapshot.
* **The original Watchtower image is archived** and loops with a "client version too old" error; use a maintained fork.
* **Run `docker compose` from the stack's folder,** or it cannot find the configuration.
* **Live databases and file copy backups do not mix.** Back up whole VMs, or stop containers before copying their data.

## Game server

* **The modpack installer's filename encodes the pack and version.** Renaming it breaks the install.
* **Set Java memory in `user_jvm_args.txt`,** not the start script, and make sure the line is not commented out.
* **A clean server shutdown under systemd needs three things:** a service of `Type=exec`, `screen -D -m` so systemd tracks the real process, and a stop script that waits for the process to exit before returning. Without the wait, systemd kills the server before the world finishes saving.
* **Client and server modpack versions must match exactly,** or players cannot join.

## Linux basics

* **A Debian install with a root password set has no sudo.** Install it as root and add your user to the sudo group, then log in again.

## Working method

* **Pasting multi line heredocs into a terminal corrupts files.** Copy files over with `scp` instead.
* **Snapshot at every working milestone,** and verify each stage with command output before moving on.
* **Diagnose read only first.** Change nothing until the cause is understood.
* **Verify claims against current documentation.** Several plans changed after checking official docs or a second opinion, including the Funnel limitation and the correct Nextcloud app name.
