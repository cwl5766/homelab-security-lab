# Lessons learned

Each of these cost real time once. They are grouped by area.

## Proxmox host

* **Give the host a static IP.** On DHCP, a router lease renewal renamed the host and stranded every VM config. See [incident 001](../incidents/001-host-renamed-by-dhcp.md).
* **Check `/etc/hosts` after install.** The installer's fallback address stayed there long after the network was fixed.
* **Without a subscription, the Enterprise repositories return 401.** Disable them and add the No Subscription repository.
* **A custom Proxmox role needs VM.Audit,** or the VMs do not even appear in the tree for that user.
* **The Qemu Agent checkbox does nothing on its own.** The `qemu-guest-agent` package (or the Windows installer from the VirtIO ISO) has to run inside the guest.
* **Old guests need old virtual hardware.** Metasploitable 2 only works reliably with a SATA disk and an e1000 network card.
* **Restarting the Proxmox services does not stop running VMs,** which makes host level repairs possible without downtime.

## Storage

* **Put VM disks on the fast NVMe and keep the boot drive for the OS.** The DRAMless boot drive would suffer under VM load; the OS barely notices.
* **Leave VM disk cache at the default.** Write back caching without a UPS risks data loss on a power cut.
* **A zvol reserves its full size up front.** The pool looked 70% full while holding a few GB of real data.
* **Docker's own storage lives on the VM system disk,** regardless of where Nextcloud's data directory points, so size the system disk generously.
* **After editing fstab, check every line starts with `UUID=`.** A missing prefix caused a mount failure.

## Networking and remote access

* **Tailscale grants are additive.** To restrict shared users, remove the broad default grant entirely instead of adding a narrower one beside it.
* **Email addresses in grants must match the account exactly.**
* **A person who signs up fresh on tailscale.com gets a separate tailnet.** They must accept the share from the invite email itself.
* **An exit node needs more than the toggle:** IP forwarding, a persistent NAT rule, and DNS override in the tailnet. Sharing it also needs an explicit grant to `autogroup:internet`.
* **VM LAN addresses move.** Use Tailscale addresses for SSH and DHCP reservations for anything you rely on.
* **Use `.test` for a lab domain, never `.local`,** which is reserved for mDNS.
* **Keep the air gap closed for lab downloads.** Bring files in on an attached ISO instead of moving a VM onto the home network.

## Applications

* **Nextcloud AIO needs a real domain with a valid certificate.** Tailscale HTTPS solved this without exposing anything publicly.
* **Install Team Folders, not Organization Folders.** They are different apps and the second is incompatible.
* **Vaultwarden requires HTTPS.** Close signups right after creating the first account.
* **The original Watchtower image is archived** and loops with a "client version too old" error; use a maintained fork.
* **Run `docker compose` from the stack's folder,** or it cannot find the configuration.
* **Live databases and file copy backups do not mix.** Back up whole VMs, or stop containers before copying their data.

## Working method

* **Pasting multi line heredocs into a terminal corrupts files.** Copy files over with `scp` instead.
* **Snapshot at every working milestone,** and verify each stage with command output before moving on.
* **Diagnose read only first.** Change nothing until the cause is understood.
