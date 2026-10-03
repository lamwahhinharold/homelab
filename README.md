# Homelab

[![Proxmox VE](https://img.shields.io/badge/Hypervisor%20Proxmox%20VE-E57008?logo=proxmox&logoColor=white)](https://www.proxmox.com)
[![Pi-hole](https://img.shields.io/badge/DNS%20Pi--hole-F60D5F?logo=pi-hole&logoColor=white)](https://pi-hole.net)
[![TrueNAS SCALE](https://img.shields.io/badge/Storage%20TrueNAS%20SCALE-0095D5?logo=truenas&logoColor=white)](https://www.truenas.com)
[![Debian](https://img.shields.io/badge/Guests%20Debian-A81D33?logo=debian&logoColor=white)](https://www.debian.org)
[![Docker](https://img.shields.io/badge/Containers%20Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com)
[![Jellyfin](https://img.shields.io/badge/Media%20Jellyfin-00A4DC?logo=jellyfin&logoColor=white)](https://jellyfin.org)

Everything I run at home, on one refurbished office desktop. **Proxmox VE** is the hypervisor.
Three guests sit on top of it, and each one exists because of a specific constraint rather than
because it was convenient to add.

| Role | What runs there |
| :--- | :--- |
| **Hardware** | HP EliteDesk 800 G3 SFF, i7-7700, 32 GB RAM, one 512 GB SSD, two 1 TB HDDs |
| **Hypervisor** | Proxmox VE on the SSD, at `192.168.50.10` |
| **Storage** | TrueNAS SCALE, holding both HDDs as a raw ZFS pool |
| **DNS** | Pi-hole in a container, serving every device on the LAN |
| **Applications** | Jellyfin in Docker, reading the NAS over a read-only mount |

The interesting problems are not the boxes. They are the questions you have to answer when you only
have one machine: **what starts first**, **what happens when one service is down but another still
needs it**, and **what a given credential is actually allowed to reach**.

## Contents

- [Hardware](#hardware)
- [Access map](#access-map)
- [Network design](#network-design)
- [Topology](#topology)
- [Step 1: Proxmox VE](#step-1-proxmox-ve)
- [Step 2: TrueNAS SCALE](#step-2-truenas-scale)
- [Step 3: Pi-hole](#step-3-pi-hole)
- [Step 4: Debian and Docker](#step-4-debian-and-docker)
- [Design decisions](#design-decisions)
- [Planned: the *arr stack](#planned-the-arr-stack)

---

## Hardware

| Component | Specification |
| :--- | :--- |
| Chassis | HP EliteDesk 800 G3 SFF, refurbished |
| CPU | Intel Core i7-7700, 4 cores / 8 threads, with Intel HD Graphics 630 |
| RAM | 32 GB DDR4 |
| System disk | 512 GB Samsung SSD, running Proxmox and both guests' disks |
| Data disks | 2 × 1 TB Seagate HDDs, handed straight to TrueNAS |
| NIC | Intel I219-LM |

Two things about the disks are worth stating up front, because they explain most of the layout
later.

**No user data ever lands on the SSD.** It carries the hypervisor plus the guests' own virtual
disks, so anything worth keeping is elsewhere by construction rather than by policy.

**The two HDDs are never touched by Proxmox at all.** They are passed through to the TrueNAS guest
whole, so Proxmox never writes a partition table to them and never has an opinion about what lives
on them. If the pool is destroyed and rebuilt, that is TrueNAS doing it, not the hypervisor.

The installer also offers to cap the filesystem a little under the full device size. Worth taking:
it leaves a little headroom at the end of the device instead of running the partition flush against
it, which is the kind of edge some firmware and disk tools handle badly.

![Proxmox installer, Harddisk options, filesystem capped on the Samsung SSD](assets/proxmox/installer-harddisk-options-hdsize.jpeg)

### Host installation

Set the `root` password and a notification address:

![Proxmox installer, Administration Password and Email Address](assets/proxmox/installer-admin-password-and-email.jpeg)

Management networking. The **FQDN**, the **static address**, and especially the **DNS server**
all get set here:

![Proxmox installer, Management Network Configuration, FQDN, static IP, DNS server](assets/proxmox/installer-management-network-config.jpeg)

That screenshot shows the host pointing at the router, which is what the installer asks for and
what it was originally set to. **It was later changed to `8.8.8.8` directly**, for the reason below.

> **Why the host uses neither Pi-hole nor the router**
>
> Pi-hole runs *inside* Proxmox, as a guest container, so the host cannot resolve through it.
> The obvious replacement is the router, and that is what this host used at first.
>
> The router is the wrong answer for a less obvious reason. A router that forwards DNS uses its
> own upstream list, and on this network that list is Pi-hole first with a public resolver behind
> it. Asking the router therefore still means asking Pi-hole, only with a retry on failure.
>
> That retry is the part that hurts. During a Proxmox reboot CT 101 is not running, so every
> lookup the host makes stalls on a dead upstream before falling through to `8.8.8.8`, including
> the lookups needed to download the very template that brings Pi-hole back.
>
> So the host is pointed straight at a public resolver. The rule is unchanged and now actually
> enforced rather than approximated: **anything that hosts Pi-hole must not depend on Pi-hole, and
> must not depend on anything that depends on Pi-hole.** Everything downstream of the host is free
> to use it.
>
> The side effect is that the hypervisor's own queries stop passing through the ad blocker and
> stop appearing in Pi-hole's query log. Infrastructure traffic has no business in either.
>
> This comes up again in [Design decisions](#design-decisions).

Review and install:

![Proxmox installer, Summary screen](assets/proxmox/installer-summary.jpeg)

---

## Access map

| Service | ID | Address | Hostname | Login | Notes |
| :--- | :---: | :--- | :--- | :--- | :--- |
| **Proxmox VE** | host | `192.168.50.10:8006` | `hl-pve.home.arpa` | `root` | Hypervisor web UI |
| **TrueNAS SCALE** | `100` | `192.168.50.12` | `truenas.home.arpa` | `truenas_admin` | `root` login disabled |
| **SMB share** | `100` | `\\192.168.50.12\share` | same dataset | `nasuser` | For Windows desktops |
| **NFS export** | `100` | `truenas:/mnt/storage/data` | same dataset | by host | For the Docker host |
| **Pi-hole** | `101` | `192.168.50.11` | `pihole.home.arpa` | `root` | Web UI has no password |
| **Docker host** | `102` | `192.168.50.30` | `docker-vm.home.arpa` | `haroldlam` | Debian, SSH or console |
| **Jellyfin** | `102` | `192.168.50.30:8096` | `docker-vm.home.arpa:8096` | `jellyfin_admin` | Container on host network |

The IDs are worth a note, because they are the reason the wiring is easy to follow later.
**`100` is storage, `101` is DNS, `102` is applications.** Anything else that gets built later
should follow the same idea: group by role, so the number alone tells you what something is.

---

## Network design

### Address plan (`192.168.50.0/24`)

The whole plan follows from one idea: **the things other things depend on get the lowest, most
predictable addresses.**

| Range | Purpose | Who gets it |
| :--- | :--- | :--- |
| `192.168.50.1` | Gateway | The router |
| `192.168.50.2`–`.9` | Network gear | Switches, access points |
| `192.168.50.10` | **Hypervisor** | Proxmox, the thing everything depends on |
| `192.168.50.11` | **DNS** | Pi-hole, depended on by nearly everything else |
| `192.168.50.12` | **Storage** | TrueNAS, where the data lives |
| `192.168.50.13`–`.29` | Reserved | Infrastructure that must exist *before* the guests |
| `192.168.50.30` | **Applications** | The Docker host |
| `192.168.50.31`–`.49` | Reserved | Future guests |
| `192.168.50.50`–`.99` | Static clients | My own machines, by fixed address |
| `192.168.50.100`–`.249` | **DHCP pool** | Everything transient |
| `192.168.50.250`–`.254` | Testing | Temporary diagnostics |

Three consequences worth naming:

1. **The two things everything else needs are `.10` and `.11`.** They are the first numbers I
   memorise and the first two I check when something is broken.
2. **`.13` to `.29` is deliberately empty.** It is reserved for services that must come up
   *before* the guests, like a VPN or a reverse proxy. Leaving that gap means adding one later
   never forces me to renumber anything.
3. **Static clients sit above guests but below DHCP.** My own machines get a predictable address
   in `.50` to `.99`, and anything temporary gets handed one from the pool. Nothing has to be
   guessed.

### DNS: one authority, two layers

Every device on the LAN gets its DNS from the router, and the router gets its DNS from Pi-hole. The
hypervisor is the deliberate exception. What follows is the part that is easy to get wrong:

| Component | Primary | Fallback | Reason |
| :--- | :--- | :--- | :--- |
| Proxmox host | `8.8.8.8` | none | It hosts Pi-hole, so it must not depend on it, directly or through the router |
| TrueNAS | Pi-hole | `8.8.8.8` | Needs local names, but must keep updating if Pi-hole is down |
| Docker host | Pi-hole | none | Needs to resolve `truenas` for the NFS mount |
| Pi-hole | `8.8.8.8` | none | It is the resolver; it forwards upstream |
| DHCP clients | Pi-hole | `8.8.8.8` | Ad blocking, with working internet if it goes away |

Two things in that table are worth stating out loud.

**The host skips the router, which looks like the safe middle ground but is not.** A router
forwarding DNS uses its own upstream list, and that list contains Pi-hole. Querying the router is
querying Pi-hole with a retry, so a host configured that way is still coupled to the guest it
hosts. Pointing straight at a public resolver is what makes the independence real. The cost is
that the host cannot resolve `home.arpa` names, which costs nothing here because the NFS mount
happens inside the Docker VM and nothing on the hypervisor looks up a guest by name.

**Whether a guest gets a fallback depends on what has to keep working during Pi-hole downtime.**
TrueNAS needs it, because it still has to install and update apps while the resolver is down. The
Docker host does not, because the only thing it needs to resolve is the NAS, and the NAS is already
running by the time it boots. Giving it a fallback would be a second configuration to keep
consistent for no benefit.

Every guest uses **`home.arpa`** as its DNS search domain. That name is reserved by
[RFC 8375](https://www.rfc-editor.org/rfc/rfc8375) for exactly this purpose, so it can never
collide with a real domain someone might actually visit.

Pi-hole also acts as the local name authority, which replaces editing `hosts` files on individual
machines:

![Pi-hole, Local DNS Records for the hypervisor and the NAS](assets/pihole/local-dns-records-home-arpa.png)

![Pi-hole, DNS settings showing upstream choice and the home.arpa local domain](assets/pihole/dns-settings-upstream-and-home-arpa.png)

The router hands Pi-hole to every device that asks for an address, with a public resolver behind
it as a fallback, and does not advertise its own address:

![Router, LAN DHCP server handing out Pi-hole as DNS server 1](assets/router/lan-dhcp-server-dns-1-pihole.png)

The result is that if Pi-hole ever stops, the network loses ad blocking and nothing else. That
fallback is the whole reason for pointing DHCP at a second resolver, and it is why the router
does not advertise itself as a third option.

---

## Topology

Read it in layers: the physical hardware, the hypervisor sitting on it, the three guests that
hypervisor runs, and the network those guests serve.

| Reading | What it means |
| :--- | :--- |
| **`VM 100` and `VM 102`** | Guests. Everything the boxes contain is a consequence of a choice explained later |
| **`host DNS ► 8.8.8.8`** | The hypervisor asks a public resolver, because it cannot depend on a guest it hosts, nor on a router that forwards to one |
| **`DNS ► .50.11`** | Both guests ask Pi-hole, which is a guest they do not host |
| **`◄` / `►` on a line** | That line carries data in the direction of the arrowhead |
| **`───►` inside the hardware box** | A device handed whole to a guest: PCI for the GPU, raw block for the drives |
| **`+ 8.8.8.8`** | A fallback resolver, so one guest being down does not stop the other |

Only two connections in that drawing carry meaning, and they are the two that could break:

- **NFS, from TrueNAS into the Docker host.** Storage and applications are separate machines, so
  one has to reach the other over the network.
- **DNS, from Pi-hole into both VMs.** The dependency runs in one direction only, and that
  direction is the point.

```
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ HP EliteDesk 800 G3 SFF     node hl-pve     192.168.50.10:8006                                     │
 │ i7-7700 (4c / 8t)    32 GB DDR4    512 GB SSD   (Proxmox OS + LVM-thin)                            │
 │ iGPU   Intel HD Graphics 630   PCI 00:02.0   ── passthrough ──►  VM 102                            │
 │ HDD    2 x 1 TB SATA drives     ─── raw block ───►  VM 100                                         │
 └─────────────────────┬──────────────────────────────────────────────────────────────────────────────┘
                       │
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ Proxmox VE        host DNS ► 8.8.8.8  (a public resolver, never Pi-hole)                           │
 │ every guest below runs on this host         guests use DNS ► 192.168.50.11                         │
 └─────────────────────┬────────────────────────────────────────────────────────┬─────────────────────┘
                       │                                                        │
 ┌───────────────────────────────────────────┐            ┌───────────────────────────────────────────┐
 │ VM 100   TrueNAS SCALE                    │            │ VM 102   Debian "docker-vm"               │
 │ 192.168.50.12   4 vCPU   8 GB             │            │ 192.168.50.30   4 vCPU   16 GB            │
 │ DNS ► .50.11 (Pi-hole), + 8.8.8.8         │            │ DNS ► .50.11 (Pi-hole)                    │
 ├───────────────────────────────────────────┤            ├───────────────────────────────────────────┤
 │ pool      storage   stripe, 1.76 TiB      │            │ Docker                                    │
 │ dataset   data                            │            │   jellyfin   :8096                        │
 │ NFS       /mnt/storage/data   read-only   │ ── NFS ──► │   /media     read-only NFS from VM 100    │
 │ SMB       \\192.168.50.12\share           │            │   config, cache   bind mounts             │
 └───────────────────────────────────────────┘            └───────────────────────────────────────────┘
                       │                                                        │
                       └───────DNS from CT 101──────┼───────────────────────────┘
                                                    │
                            ┌───────────────────────┴──────────────────────┐
                            │ CT 101   Pi-hole   unprivileged container    │
                            │ 192.168.50.11   1 vCPU   512 MB              │
                            ├──────────────────────────────────────────────┤
                            │ LAN DNS authority  +  ad blocking            │
                            │ local zone  home.arpa   (RFC 8375)           │
                            └──────────────────────────────────────────────┘

 ┌────────────────────────────────────────┐
 │ The router   gateway   192.168.50.1    │
 │ DHCP pool  .50.100 to .50.249          │
 │ hands out  DNS 1 = .50.11 (Pi-hole)    │
 │            DNS 2 = 8.8.8.8, not itself │
 └────────────────────────────────────────┘

 └─►  every DHCP client on the LAN gets ad blocking and
      home.arpa names automatically, with nothing to
      configure on each device
```

The NFS arrow sits on the row where the export path and the mount point are written, and it points
one way only. Jellyfin reads media off the NAS and never writes back to it, which is the reason
that mount is read-only at every layer it can be.

The DNS labels are deliberately not symmetrical, and that is the most important thing in the
picture. The hypervisor reads `host DNS ► 8.8.8.8` while both guests read `DNS ► .50.11`.
Both guests can use Pi-hole because they do not host it. The hypervisor cannot, because it does,
and it cannot go via the router either, because the router forwards to Pi-hole.

### Boot order

The NFS dependency has a consequence at startup. Docker starts its containers early in boot, but
the `/media` mount inside Jellyfin has to come from a different virtual machine that is not running
yet. If the Docker host finishes booting first, Jellyfin comes up with an empty library.

Proxmox solves this with a field called **startup delay**, which holds the boot sequence before
moving to the next guest:

| Guest | Start at boot | Order | Startup delay |
| :--- | :---: | :---: | :---: |
| `100` TrueNAS | Yes | `2` | **70 s** |
| `101` Pi-hole | Yes | `1` | none |
| `102` Docker host | Yes | `any` | none |

![TrueNAS VM 100, Options tab with Start at boot enabled](assets/truenas/vm-100-options-start-at-boot.png)

![TrueNAS VM 100, startup delay set to 70 seconds](assets/truenas/vm-100-startup-delay-70s.png)

Seventy seconds was measured rather than guessed: long enough for TrueNAS to boot, bring up its
network interface, start the NFS service, and register the export.

What makes this worth doing is where the fix lives. Nothing inside either guest has to know the
other exists, because the ordering is one field in the hypervisor. The alternative would be a
`systemd` drop-in on the Docker host that waits for a network share, which spreads one dependency
across two systems and silently stops working the next time that VM is rebuilt.

---

## Step 1: Proxmox VE

The base install is the standard installer. One fix is needed afterwards before anything else will
work.

### Making package updates work without a subscription

A fresh Proxmox install points APT at the enterprise repository, which requires a paid
subscription key. With no key, every package operation fails:

![Proxmox, task viewer showing apt-get update failing with exit code 100](assets/proxmox/apt-get-update-exit-code-100.png)

The fix is four clicks in the web UI. Disable the two enterprise repositories, add the
no-subscription one, then reload:

![Proxmox, APT repositories with enterprise disabled and no-subscription enabled](assets/proxmox/apt-repositories-enterprise-disabled.png)

![Reference card, the four repository actions and how to verify each](assets/proxmox/note-repository-fix-procedure.png)

| # | Action | File | How to tell it worked |
| :---: | :--- | :--- | :--- |
| 1 | Disable | `pve-enterprise.sources` | The `Enabled` tick disappears |
| 2 | Disable | `ceph.sources` | Ceph enterprise stops being queried |
| 3 | Add | `proxmox.sources`, **No-Subscription** | A new `pve-no-subscription` entry appears |
| 4 | Reload | | The amber warning clears |

Disabling the Ceph repository is the row that surprises people. It is switched on by default even
though this host runs no storage cluster at all. Leaving it enabled means APT queries a repository
that can never succeed, which produces update failures and `ceph` upgrade prompts that have
nothing to do with anything running here.

> An amber banner saying the no-subscription repository is not recommended for production stays
> there permanently. On a node without a subscription that is expected, not a problem.

### Fetching the TrueNAS image

Proxmox can download an ISO straight to its own storage, so the file never has to be moved between
machines:

![Proxmox, Download from URL dialog for the TrueNAS image](assets/proxmox/download-truenas-iso-from-url.png)

![Proxmox, TrueNAS image download complete](assets/proxmox/truenas-iso-download-complete.png)

Later the same view holds the Debian image next to it:

![Proxmox, ISO images list holding both TrueNAS and Debian](assets/proxmox/iso-images-list-debian-and-truenas.png)

---

## Step 2: TrueNAS SCALE

This guest owns the two HDDs and serves everything else.

### Why SCALE and not CORE

![Reference card, what SCALE stands for and why it was chosen over CORE](assets/truenas/note-why-truenas-scale.png)

TrueNAS has two lineages. **CORE** runs on FreeBSD. **SCALE** runs on Linux. That sounds like a
detail, but it decides how much of my own tooling applies. With SCALE, the storage appliance and
any Linux workloads share one kernel, one package manager, and one set of commands. Choosing CORE
would mean maintaining a FreeBSD box and a Debian box side by side for no benefit, since this host
has exactly one storage job and several Linux ones.

### 2.1 Finding the HDDs by their real identity

This is the step most guides skip, and skipping it causes problems months later.

Linux hands out names like `/dev/sda` and `/dev/sdc` by counting devices as it finds them. Those
names are not attached to the drives. Add a disk, move a cable, or reboot and the counting can
change, and the drive that was `sda` is now `sdc`. A passthrough config pointing at `sda` would
then hand TrueNAS the wrong disk.

Linux also provides stable names derived from what the hardware actually is, in
`/dev/disk/by-id/`:

```bash
ls -l /dev/disk/by-id/
```

![Proxmox shell, output of ls -l /dev/disk/by-id/ with the two data drives highlighted](assets/proxmox/list-dev-disk-by-id-output.png)

The output mixes several kinds of entry, and most of them can be ignored. Drive models are real;
the serials in angle brackets are not, and stand in for values I have not published.

```text
ata-SAMSUNG_MZ7LH512HALU-000H1_<SSD_SERIAL>     -> ../../sdb     <- the 512 GB OS SSD
ata-ST1000DM010-2EP102_<DISK1_SERIAL>           -> ../../sda     <- target
ata-ST1000DM010-2EP102_<DISK1_SERIAL>-part1     -> ../../sda1    <- ignore
ata-ST1000DM010-2EP102_<DISK2_SERIAL>           -> ../../sdc     <- target
ata-ST1000DM010-2EP102_<DISK2_SERIAL>-part1     -> ../../sdc1    <- ignore
ata-ST1000DM010-2EP102_<DISK2_SERIAL>-part2     -> ../../sdc2    <- ignore
dm-name-pve-root / dm-name-pve-swap / lvm-pv-uuid-* / wnn-*      <- host-managed, ignore
```

![Reference card, passing only whole-disk paths and checking the OS drive is excluded](assets/proxmox/note-drive-id-selection-and-verification.png)

The `dm-`, `lvm-` and `wwn-` entries are things the host manages itself and are never targets.
The two `ST1000DM010` entries are the drives I want. Two rules govern which of those to use:

- **Use the whole-disk path, never the `-partN` one.** ZFS manages the entire physical device: it
  writes its own labels and keeps its own record of which disk is which. Hand it a partition
  instead and it gets a slice of a disk with no way to see the rest, which breaks the checksumming
  it depends on. TrueNAS wipes existing partitions itself when it builds a pool, so nothing is lost
  by pointing at the whole disk.
- **Check that the OS drive is not in the list.** The Samsung SSD is in that same output. Pointing
  TrueNAS at it would overwrite the hypervisor. `lsblk` in the Proxmox shell shows which device
  is mounted as `/` or `pve-root`.

The two paths I need:

```text
/dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK1_SERIAL>
/dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK2_SERIAL>
```

### 2.2 Creating the VM

![Reference card, Create VM parameter sheet noting the CPU should be 4 not 2](assets/truenas/note-create-vm-parameters.png)

| Setting | Value | Why |
| :--- | :--- | :--- |
| VMID | `100` | Storage role |
| Name | `TrueNAS-SCALE` | |
| SCSI controller | **VirtIO SCSI single** | Keeps device names stable when disks change |
| BIOS | Default (SeaBIOS) | Matches the i440FX machine type |
| QEMU guest agent | **off** | TrueNAS manages its own guest integration |
| scsi0 | `local-lvm`, **32 GiB** | Boot disk only, never data |
| CPU | 1 socket, **4 cores**, type `host` | ZFS is CPU-hungry during scrubs and compression |
| Memory | **8 GiB** | ZFS keeps a cache in RAM and will claim a share of it |
| Network | `vmbr0`, VirtIO, firewall on | |
| Start after created | **off** | The HDDs get attached first |

![Create VM, General tab showing VMID 100](assets/truenas/create-vm-general-vmid-100.png)
![Create VM, OS tab with the TrueNAS image selected](assets/truenas/create-vm-os-iso.png)
![Create VM, System tab with VirtIO SCSI selected](assets/truenas/create-vm-system-scsi-controller.png)
![Create VM, Disks tab, 32 GiB boot disk](assets/truenas/create-vm-disks-32g.png)
![Create VM, CPU tab, 4 cores of type host](assets/truenas/create-vm-cpu-4-cores-host.png)
![Create VM, Memory tab, 8192 MiB](assets/truenas/create-vm-memory-8192.png)
![Create VM, Network tab on vmbr0](assets/truenas/create-vm-network-vmbr0.png)
![Create VM, Confirm summary for vmid 100](assets/truenas/create-vm-confirm-summary.png)

**Why 4 cores.** My own notes flag this as a correction I made partway through. ZFS spends CPU in
bursts: checksumming incoming data, compressing it, and especially running a scrub, which reads the
entire pool to verify it. On 2 vCPUs the pool felt visibly sluggish during scrubs and heavy writes.
Four cores of `host` type is the point where that stops being noticeable.

It is also the honest ceiling, and it is worth being clear about. VM 102 is given four cores as
well, so all four physical cores are committed and the hypervisor itself runs on whatever the
guests are not currently using. There is no spare capacity parked here, which is a large part of
why [boot order](#boot-order) matters rather than being a nicety.

**Why the guest agent is off here but on for the Docker VM.** The agent is what lets Proxmox ask a
guest "what IP addresses do you have" and "please shut down cleanly", and that is what makes the
stop button and snapshots behave. TrueNAS is an appliance that already manages its own disk flushes
and shutdown ordering, so giving the hypervisor a second shutdown path into it buys nothing.

### 2.3 Handing the HDDs to the VM

Before the first boot, attach both physical drives from the Proxmox shell:

```bash
qm set 100 -scsi1 /dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK1_SERIAL>
qm set 100 -scsi2 /dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK2_SERIAL>
```

![Proxmox web shell, qm set commands attaching both drives to VM 100](assets/truenas/proxmox-shell-qm-set-scsi1-scsi2.png)

![Reference card, the passthrough commands and how to verify them in the Hardware tab](assets/truenas/note-pass-through-both-hdds.png)

The command line rather than the web UI is deliberate here. The UI's disk tools are built around
creating virtual disks, and this needs the opposite: the physical disk itself, handed over whole.
Afterwards both drives show up under **Hardware** as real disks rather than images:

![VM 100 Hardware tab, showing the boot disk and both passed-through drives](assets/truenas/vm-100-hardware-after-hdd-passthrough.png)

### 2.4 Installing TrueNAS

![VM 100 console, GRUB menu for the TrueNAS installation](assets/truenas/vm-100-console-grub-install-menu.png)

Choose **1. Install/Upgrade**:

![TrueNAS installer, console menu with Install/Upgrade selected](assets/truenas/installer-menu-select-install-upgrade.png)

**Read the destination media list carefully.** Both 1 TB drives already carry signatures from a
previous life, one a ZFS pool and one an NTFS volume:

![TrueNAS installer, Choose Destination Media showing both drives with existing signatures](assets/truenas/installer-choose-destination-media.png)

TrueNAS is telling me two things at once: it can *import* these, and it will *destroy* them. On
this build that data is disposable, so proceeding is a deliberate choice rather than an accident.
Had the data mattered, the correct button would be **Import Pool**, which adopts the existing pool
instead of erasing it. The default path here is **Create Pool**, and understanding which one you
are clicking is the actual decision.

Use **1. Administrative user** rather than configuring authentication afterwards. This forces a
named admin account and disables `root` login completely, which is free security:

![TrueNAS installer, Web UI authentication method with an administrative user](assets/truenas/installer-web-ui-auth-method.png)

![TrueNAS installer, setting the admin password with root login disabled](assets/truenas/installer-set-truenas-admin-password.png)

Answer **No** to EFI boot. This VM uses the default i440FX machine type with SeaBIOS and no EFI
firmware, so a legacy BIOS install is what matches the hardware it is running on:

![TrueNAS installer, legacy boot prompt with No selected](assets/truenas/installer-legacy-boot-no-efi.png)

![TrueNAS installer, installation succeeded on the boot disk](assets/truenas/installer-installation-succeeded.png)

![TrueNAS installer, console menu with Reboot System selected](assets/truenas/installer-menu-reboot-system.png)

Once it boots, detach the installer image so a later reboot cannot start from the install media
instead of the installed system:

![VM 100 Hardware tab, CD/DVD drive set to use no media](assets/truenas/vm-100-hardware-detach-iso.png)

### 2.5 Setting the address before the web UI exists

There is a window before the web interface is reachable where the only thing that can set the
network is the bootloader menu, reached with option `1` on the console. Miss it and the machine
looks dead, because nothing is listening.

![TrueNAS bootloader, interface settings with a static address and no DHCP](assets/truenas/bootloader-interface-settings-static-ip.png)

![TrueNAS bootloader, network interface list confirming the static address](assets/truenas/bootloader-network-interface-list.png)

Both DHCP and automatic IPv6 are switched off. The reason is that this address is a dependency for
half the network: if the NAS were handed a new address by DHCP after a router restart, everything
pointing at `.50.12` would break silently, and the one machine that could fix it would itself be
reached by the wrong address.

### 2.6 Building the pool, and the error worth understanding

With the web UI up, the first attempt at creating a pool fails:

![TrueNAS, storage dashboard with no pools yet](assets/truenas/storage-dashboard-no-pools.png)

Start the pool wizard. The first step shows which drives it will consume:

![Pool creation wizard, step 1 with the pool name and available drives](assets/truenas/pool-wizard-step1-general-info.png)

**Layout: stripe, two disks wide, one data VDEV.**

![Pool creation wizard, step 2 with a stripe layout across two drives](assets/truenas/pool-wizard-step2-data-stripe.png)

The review step shows both warnings together, the stripe warning and the pre-existing signatures:

![Pool creation wizard, review step with topology and both warnings](assets/truenas/pool-wizard-review.png)

> **Why stripe, when the tool warns against it**
>
> TrueNAS displays a red warning that a stripe vdev *"is highly discouraged and will result in data
> loss if it fails."* That is correct, and it is worth understanding rather than clicking past.
>
> A stripe across two drives gives **1.76 TiB**. A mirror gives about **0.87 TiB** and survives the
> loss of one disk. So the question is not "which is better" but "what is this pool for."
>
> This pool holds media. Any file here can be downloaded again, and the whole library is backed up
> to offline storage anyway. Against that background, halving the capacity to survive a disk
> failure that the backups already cover would be paying twice for the same protection. The
> honest cost is that a rebuild is no longer possible without the backup, which makes that backup
> mandatory rather than optional. It is the redundancy.
>
> If this pool held photographs or documents, stripe would be the wrong answer, and the correct
> move would be mirror.
>
> What I do keep is the weekly scrub. It reads the entire pool once a week to verify the data still
> matches its checksums, which catches the slow failure that a dead-disk warning never will. On a
> pool of replaceable media that is worth more than redundancy would be.

Confirm the erase, and the real error appears:

![Pool creation wizard, warning that all disks will be erased](assets/truenas/pool-wizard-erase-disks-warning.png)

![TrueNAS, error that disks have duplicate serial numbers, all None](assets/truenas/pool-wizard-error-duplicate-serials.png)

**This is the most instructive error in the whole build**, so it is worth following properly.

Proxmox's virtual disk controller does not pass the physical drive's serial number through to the
guest. Inside the VM, all three disks report the serial `None`. TrueNAS refuses to continue,
because it has no way to tell the disks apart, and ZFS identifies each member of a pool by the
pair of (pool name, disk serial). If two disks share a serial, a later swap would be
indistinguishable from a mistake, and that is precisely the kind of silent data loss ZFS exists to
prevent.

The refusal is the system working correctly. The fix is to give every virtual disk a serial that
is genuinely unique. This is a change to the VM's definition on the hypervisor, so the VM has to be
shut down first:

> Connecting over SSH from a Windows workstation. `ssh root@hl-pve` does not resolve until Pi-hole
> knows the name, so the first connection uses the IP address and shows the host key to verify.

![Windows PowerShell, SSH to the Proxmox host then opening the VM config in nano](assets/truenas/ssh-into-proxmox-to-edit-vm-config.png)

```bash
nano /etc/pve/qemu-server/100.conf
```

Add `serial=` to all three disks. The boot disk needs one too, otherwise it collides with the two
data drives:

![The VM config with serial parameters added to all three disks](assets/truenas/edit-vm-config-add-serial-params.png)

```ini
scsi0: local-lvm:vm-100-disk-0,iohread=1,size=32G,serial=BOOTDRIVE
scsi1: /dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK1_SERIAL>,size=976762584k,serial=<DISK1_SERIAL>
scsi2: /dev/disk/by-id/ata-ST1000DM010-2EP102_<DISK2_SERIAL>,size=976762584k,serial=<DISK2_SERIAL>
```

Save with `Ctrl+O` then `Enter`, exit with `Ctrl+X`, and start the VM again. Each serial is reused
from the `by-id` path on purpose: the real hardware serial is already a unique identity, so there
is no reason to invent a second one.

Before starting it back up, turn on autostart:

![VM 100 options, start at boot enabled](assets/truenas/vm-100-options-start-at-boot.png)

The pool now builds. Healthy, 1.76 TiB usable, weekly scrub scheduled:

![TrueNAS, storage dashboard with the pool online and its scrub schedule](assets/truenas/storage-dashboard-pool-online.png)

> **One honest limitation of passing disks through.** The disk health panel reports that no disk
> temperature is available. Passing a disk straight through gives the guest the ability to read and
> write blocks, but not the channel that carries SMART data, and SMART is where temperature lives.
> So for these two drives, health monitoring has to happen at the hypervisor rather than in the NAS.
> It is a genuine trade: giving TrueNAS the disk directly gets it features that no amount of
> indirection would, and the cost is that it cannot see the disk's own health reporting.

### 2.7 Dataset, SMB share, NFS export

Create a child dataset instead of sharing the pool root. It costs nothing, keeps the pool's own
system data separate from mine, and leaves room for snapshots, quotas, and permissions later:

![Reference card, creating a child dataset then the SMB and NFS shares](assets/truenas/note-create-dataset-and-shares.png)

![TrueNAS, datasets list showing details for the root dataset](assets/truenas/dataset-storage-details.png)

![TrueNAS, add dataset dialog with the name data](assets/truenas/add-dataset-data.png)

Add the SMB share, pointing at that dataset:

![TrueNAS, add SMB share dialog for /mnt/storage/data](assets/truenas/add-smb-share-share.png)

TrueNAS warns that no user can reach the share yet. That is left for [2.8](#28-a-user-that-can-do-exactly-one-job)
rather than clicked past, and then the service is started with autostart enabled:

![TrueNAS, prompt to start the SMB service with autostart](assets/truenas/start-smb-service-prompt.png)

![TrueNAS, shares page with SMB running and NFS still stopped](assets/truenas/shares-smb-running-nfs-stopped.png)

Add the **NFS** share for the Docker host. Same path, different protocol, chosen because SMB is
built for Windows file sharing while NFS is what a Linux host mounts natively into a container:

![TrueNAS, add NFS share panel for the dataset](assets/truenas/add-nfs-share-panel.png)

![TrueNAS, prompt to start the NFS service](assets/truenas/start-nfs-service-prompt.png)

![TrueNAS, shares page with both SMB and NFS running](assets/truenas/shares-smb-and-nfs-running.png)

### 2.8 A user that can do exactly one job

`nasuser` exists to read and write the media share from Windows. That is the entire job, so that is
all it is given. SMB access, and nothing else. No web UI, no shell, no SSH, no API key:

![Reference card, creating the SMB user, setting ownership, connecting from Windows](assets/truenas/note-create-smb-user-and-connect-windows.png)

![TrueNAS, credentials showing root has no password, no SMB and no SSH access](assets/truenas/credentials-users-root-locked-down.png)

![TrueNAS, add user dialog for nasuser with only SMB access ticked](assets/truenas/add-user-nasuser-smb-only.png)

The value of this is entirely in what it rules out. If that password ever leaks, whoever has it
gets a file share. They cannot open the TrueNAS interface, cannot get a shell on the NAS, and
cannot reach any of its configuration. Meanwhile the built-in `root` account is locked down
completely, which is why an ordinary admin account is the way in.

Now the dataset ownership, so the share is actually writable:

![TrueNAS, dataset view showing SMB and NFS share badges and the permissions button](assets/truenas/dataset-data-shares-attached.png)

![TrueNAS, edit permissions setting nasuser as owner, applied recursively](assets/truenas/edit-permissions-set-nasuser-owner.png)

> Two details in that dialog matter. **Recursive** is ticked, so existing files inherit the change
> rather than needing it applied file by file. And `Other` is granted read and execute but **not**
> write, while the owning group gets all three. That is the difference between "the service account
> can write" and "anything that connects can write", and it is the whole reason to set ownership
> explicitly instead of leaving the permissive default in place.

![TrueNAS, permissions now showing nasuser as owner](assets/truenas/dataset-permissions-nasuser-owner.png)

From Windows: `Win+R`, then `\\192.168.50.12\share`, then authenticate as `nasuser`.

### 2.9 Pointing TrueNAS at Pi-hole (After Pi-hole is ready)

With Pi-hole running, TrueNAS can resolve the local `home.arpa` names. It gets a **public resolver
as a fallback** so that app installs and updates keep working while Pi-hole is down:

![TrueNAS, edit global configuration with Pi-hole as DNS and a public fallback](assets/truenas/edit-global-configuration-dns.png)

And also change TrueNAS to light mode:

![TrueNAS, general settings showing version, machine type, memory and theme](assets/truenas/general-settings-system-info.png)

---

## Step 3: Pi-hole

Pi-hole runs as a container rather than a virtual machine. It is one process listening on one
port, and a container shares the host kernel instead of running its own, so it starts in
milliseconds and stays small. An unprivileged container is still isolated, because Linux user
namespaces give it its own view of users and files that the host cannot reach into casually.

### 3.1 Template

Download a Debian template:

![Proxmox, CT templates list with the Debian standard template selected](assets/pihole/download-debian-13-ct-template.png)

### 3.2 Creating the container

| Setting | Value |
| :--- | :--- |
| CTID | `101` |
| Hostname | `pihole` |
| Unprivileged | **yes** |
| Nesting | **yes** |
| Disk | `local-lvm`, 8 GiB |
| CPU | 1 core |
| Memory / Swap | 512 MiB / 512 MiB |
| Network | `eth0`, static `192.168.50.11/24`, gateway `192.168.50.1` |
| DNS | `8.8.8.8` |

![Create CT, General tab showing CTID 101 and hostname pihole](assets/pihole/create-ct-general-vmid-101.png)
![Create CT, template tab with the Debian template selected](assets/pihole/create-ct-template-tab.png)
![Create CT, disks tab with an 8 GiB root filesystem](assets/pihole/create-ct-disks-8g.png)
![Create CT, CPU tab with 1 core](assets/pihole/create-ct-cpu-1-core.png)
![Create CT, memory tab with 512 MiB](assets/pihole/create-ct-memory-512.png)
![Create CT, network tab with the static address](assets/pihole/create-ct-network-static-ip.png)
![Create CT, DNS tab set to 8.8.8.8](assets/pihole/create-ct-dns-8888.png)
![Create CT, confirm summary for CTID 101](assets/pihole/create-ct-confirm-summary.png)

**On the small allocation.** Pi-hole's resolver is a small, tightly written service that is
completely comfortable inside half a gigabyte. Giving it 512 MiB leaves the host's 32 GB almost
entirely for the ZFS cache and the Docker VM.

**On nesting.** This lets a container start its own lightweight init system, which some software
expects. It does weaken isolation a little, because a container that can nest can do more inside
itself. It is a convenience I chose knowingly, not something I overlooked.

**On DNS being set to `8.8.8.8` here.** At creation time this container is *about to become* the
resolver for the whole network, so it has to be able to reach an upstream that is not itself.

Start it and log in at the console:

![CT 101 console, Debian login prompt for the pihole container](assets/pihole/ct-101-console-root-login.png)

### 3.3 Installing Pi-hole

```bash
apt update && apt install -y curl
curl -sSL https://install.pi-hole.net | bash
```

**Upstream resolver: Google with ECS and DNSSEC.** ECS lets the resolver send part of the client's
address so the answer comes back from a nearby server. DNSSEC means the resolver can verify that a
"this domain does not exist" answer is genuine rather than forged:

![Pi-hole installer, upstream DNS provider selection](assets/pihole/installer-select-upstream-dns-google.png)

**Blocklist: StevenBlack's Unified Hosts List.** One well-maintained list rather than a dozen
overlapping ones. Overlapping lists look safer but tend to block things that only one of them
objected to:

![Pi-hole installer, blocklist selection](assets/pihole/installer-blocklists-stevenblack.png)

**Query logging: enabled.** This is the only way to answer "why does this one domain not work",
and a black box resolver cannot tell you that:

![Pi-hole installer, query logging enabled](assets/pihole/installer-enable-query-logging.png)

**Query logging privacy level: show everything.** Pi-hole stores a history of every name looked up
on the network, and level 0 turns off the rotation that anonymises it. That is fine here because
everything stays on the LAN, but it is a decision about what gets recorded rather than a default:

![Pi-hole installer, query log privacy level selection](assets/pihole/installer-ftl-privacy-level.png)

Installation finishes by generating an admin password and showing it **once**:

![Pi-hole installer, installation complete screen](assets/pihole/installer-complete-admin-password.png)

![Reference card, the generated password cannot be recovered, only reset](assets/pihole/note-pihole-forgot-password.png)

Since it cannot be recovered, the options are to write it down somewhere safe or remove it. It gets
removed:

```bash
pihole setpassword
```

![Pi-hole, setpassword command confirming the password was removed](assets/pihole/pihole-setpassword-removed.png)

> **Why the Pi-hole web interface has no password**
>
> The installer produces a random password that cannot be looked up later. If it is lost, the only
> route back in is the console, which means a physical connection to the machine. On a service that
> is supposed to be reachable from every device on the LAN, that is a self-inflicted outage waiting
> to happen.
>
> So the trade is deliberate: give up authentication on the admin interface, and get back
> recoverability. The interface is only reachable from `192.168.50.0/24`, which is a real boundary
> but not a strong one.
>
> **If Pi-hole is ever exposed beyond the LAN, this stops being a reasonable trade and the web
> interface needs a real password.** None of this applies to a publicly reachable service.

Turn on autostart:

![CT 101 options, start at boot enabled](assets/pihole/ct-101-options-start-at-boot.png)

### 3.4 The NTP warning is expected

The diagnosis page shows a permanent error:

![Pi-hole, diagnosis page showing an NTP permission error](assets/pihole/pihole-diagnosis-ntp-insufficient-permissions.png)

**This is correct behaviour, not a fault.**

A container shares the host kernel, which means it shares the host clock. The kernel is
deliberately built so that a container cannot change that clock, because doing so would change the
time for every other container and the host itself. Pi-hole's resolver runs its own time
synchronisation, tries to adjust the clock, is refused, and writes down that it was refused.

The host is already synchronised, so the container's clock is right. The message is noise from a
process trying to do something it correctly cannot do. Ignore it.

> If a container genuinely needed its own clock, the fix would be on the host, not by relaxing the
> container's permissions.

### 3.5 Local names, and which machines should ask Pi-hole

At this point the natural question is whether to point everything at Pi-hole now that it is
working. The answer is yes for almost everything, and there is one important exception:

![Reference card, the DNS dependency analysis and the resulting split](assets/pihole/note-dns-dependency-deadlock.png)

| Machine | Use Pi-hole? | Reason |
| :--- | :--- | :--- |
| **Proxmox host** | **No**, use `8.8.8.8` | It hosts Pi-hole, and the router forwards to Pi-hole, so neither is safe |
| **TrueNAS** | **Yes**, plus a public fallback | Needs local names, but must keep updating during Pi-hole downtime |
| **Docker host** | **Yes** | Needs to resolve `truenas` for the NFS mount |
| **LAN clients** | **Yes**, plus a public fallback | Handed out by the router |

The rule that produces this table: **anything which hosts Pi-hole must not depend on it, and must
not depend on anything that depends on it.** Everything downstream is free to use it.

Add local records for the hypervisor and the NAS:

![Pi-hole, local DNS records mapping hostnames to addresses](assets/pihole/local-dns-records-home-arpa.png)

Then point the router's DHCP at Pi-hole, shown in [Network design](#dns-one-authority-two-layers).

---

## Step 4: Debian and Docker

Containers run on a Debian VM rather than directly on Proxmox. Two things come out of that: the
host's own storage stays out of reach of the container runtime, and the iGPU can be handed to a
single guest cleanly.

### 4.1 Creating the VM

| Setting | Value |
| :--- | :--- |
| VMID | `102` |
| Name | `docker-vm` |
| SCSI controller | VirtIO SCSI single |
| QEMU guest agent | **on** |
| Disk | `local-lvm`, **80 GiB** |
| CPU | 1 socket, **4 cores**, type `host` |
| Memory | **16 GiB** |
| Network | `vmbr0`, VirtIO, firewall on |

![Create VM, General tab showing VMID 102](assets/docker-host/create-vm-general-vmid-102.png)
![Create VM, OS tab with the Debian netinst image](assets/docker-host/create-vm-os-debian-13-iso.png)
![Create VM, System tab with the guest agent enabled](assets/docker-host/create-vm-system-qemu-agent.png)
![Create VM, disks tab with an 80 GiB disk](assets/docker-host/create-vm-disks-80g-lvm-thin.png)
![Create VM, CPU tab with 4 cores of type host](assets/docker-host/create-vm-cpu-4-cores-host.png)
![Create VM, memory tab with 16384 MiB](assets/docker-host/create-vm-memory-16g.png)
![Create VM, network tab on vmbr0](assets/docker-host/create-vm-network-vmbr0-virtio.png)
![Create VM, confirm summary for VMID 102](assets/docker-host/create-vm-confirm-summary.png)

> The guest agent is **on** here and **off** on TrueNAS, for the reason given in
> [2.2](#22-creating-the-vm). It is worth enabling on an ordinary Linux server because it is what
> makes Proxmox's stop button and snapshots behave correctly.

**On the sizing.** 16 GiB is more than Jellyfin needs, but it is a ceiling rather than a
reservation. Proxmox only hands the guest what it actually uses, so the memory stays available to
the pool when it is not in use. The 80 GiB disk holds the OS, the container images, and the
container configuration.

### 4.2 A minimal Debian install

<details>
<summary><b>Full installer walkthrough (click to expand)</b></summary>

![Installer, language selection](assets/docker-host/installer-select-language.png)
![Installer, location selection](assets/docker-host/installer-select-location.png)
![Installer, keyboard configuration](assets/docker-host/installer-configure-keyboard.png)
![Installer, hostname set to docker-vm](assets/docker-host/installer-set-hostname-docker-vm.png)
![Installer, domain name set to home.arpa](assets/docker-host/installer-set-domain-home-arpa.png)

**Skip the root password.** Press Enter twice at the prompt. Everything is administered through
the single `haroldlam` account, which has full `sudo`. That removes a credential and removes the
question of which account to log in as:

![Installer, user setup with the root password left empty](assets/docker-host/installer-skip-root-password.png)
![Installer, full name for the new user](assets/docker-host/installer-full-name.png)
![Installer, username set to haroldlam](assets/docker-host/installer-username-haroldlam.png)
![Installer, user password entry, masked](assets/docker-host/installer-user-password-confirm.png)

**Partitioning: one partition for everything.** No separate boot, home, or LVM layers. On a
machine with one job, a single filesystem has fewer places to go wrong than a layered layout, and
nothing here needs the flexibility:

![Installer, partitioning method selection](assets/docker-host/installer-partition-method.png)
![Installer, disk selection](assets/docker-host/installer-select-disk-sda.png)
![Installer, partitioning scheme, all files in one partition](assets/docker-host/installer-partitioning-scheme.png)
![Installer, partition overview showing root and swap](assets/docker-host/installer-partition-overview.png)
![Installer, confirmation before writing changes](assets/docker-host/installer-confirm-write-changes.png)

Use the standard Debian mirror, no proxy, and **decline the usage survey**, since a container host
has no reason to phone home:

![Installer, package manager configuration](assets/docker-host/installer-configure-package-manager.png)
![Installer, mirror country selection](assets/docker-host/installer-apt-mirror-country.png)
![Installer, mirror selection](assets/docker-host/installer-apt-mirror-debian-org.png)
![Installer, proxy left blank](assets/docker-host/installer-http-proxy-blank.png)
![Installer, usage survey declined](assets/docker-host/installer-popularity-contest-declined.png)

**Software selection: nothing beyond standard utilities.** No desktop, no web server, not even SSH
in the installer. This machine has no display, and every package on it should be something added
deliberately and therefore accounted for:

![Installer, software selection left minimal](assets/docker-host/installer-software-selection-minimal.png)

Point the bootloader at the virtual disk, then finish:

![Installer, bootloader installation prompt](assets/docker-host/installer-grub-install-to-primary-drive.png)
![Installer, bootloader target device](assets/docker-host/installer-grub-target-device-sda.png)
![Installer, installation complete](assets/docker-host/installer-finish-installation.png)

</details>

### 4.3 Static networking

> This screenshot is the **host workstation's** WSL resolver file, not the VM's. It is included
> because it is where this check was originally captured.

![Host workstation WSL resolv.conf, captured by mistake](assets/docker-host/host-pc-wsl-resolv-conf.png)

Edit `/etc/network/interfaces`:

![Reference card, the static address and DNS settings for the interface](assets/docker-host/note-configure-static-ip-and-dns.png)

```ini
auto ens18
iface ens18 inet static
    address 192.168.50.30/24
    gateway 192.168.50.1
    dns-nameservers 192.168.50.11
    dns-search home.arpa
```

![Debian VM, /etc/network/interfaces with the static configuration applied](assets/docker-host/etc-network-interfaces-static.png)

Two small changes from the default. The address moves from the template's `.102` to `.30` to match
the address plan. And `allow-hotplug` becomes `auto`: `hotplug` waits for a link event before
bringing the interface up, which behaves differently and less predictably than `auto` at boot.

Apply it without rebooting, clearing the old DHCP address first:

![Reference card, clearing the old address and restarting networking](assets/docker-host/note-apply-network-changes.png)

```bash
sudo ip addr flush dev ens18
sudo systemctl restart networking
# or
sudo ifdown ens18 && sudo ifup ens18
```

### 4.4 Why the DNS settings appeared to do nothing

After that change, `/etc/resolv.conf` is still empty:

![Reference card, why the resolver settings are ignored and why writing the file directly is safe](assets/docker-host/note-why-dns-nameservers-ignored.png)

- Debian's `ifupdown` manages **addresses and routes**. It does not write `/etc/resolv.conf`.
- Those `dns-nameservers` lines are only read if the **`resolvconf`** package is installed, which is
  the piece that connects resolver updates to the interface lifecycle.
- This is a minimal install, so `resolvconf` is not present. Nothing on the system was managing the
  file, so the change to a static address had no visible effect on it at all.

There are two reasonable fixes: install `resolvconf` and let `ifupdown` own the file, or write the
file directly. On a minimal Debian install, writing it directly is normal and works well, because
Debian keeps `/etc/resolv.conf` as a plain file that nothing else overwrites. That is different from
Ubuntu, where a resolver daemon manages the file and writes to a local stub address:

![Reference card, populating the resolver file directly and testing it](assets/docker-host/note-write-resolv-conf-manually.png)

```bash
sudo sh -c 'echo "nameserver 192.168.50.11" > /etc/resolv.conf'
sudo sh -c 'echo "search home.arpa"  >> /etc/resolv.conf'

ping -c 2 debian.org
```

> `ifupdown` rather than `systemd-networkd` is a deliberate choice. Debian has used this
> configuration file for two decades, it uses almost no memory, and it is what Proxmox guests
> conventionally use. `systemd-networkd` is the default on other distributions and ships disabled
> on Debian for compatibility, so choosing it here would be opting into something this system was
> not built around.

Check everything at once:

![Debian VM, ip address, route and resolver output all correct](assets/docker-host/verify-ip-route-and-resolv-conf.png)

Turn on autostart:

![VM 102 options, start at boot enabled](assets/docker-host/vm-102-options-start-at-boot.png)

### 4.5 Guest agent and Docker

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

Then Docker from its official Debian repository:

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                   docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER && newgrp docker
```

![Reference card, guest agent and Docker installation](assets/docker-host/note-install-qemu-agent-and-docker.png)

> Being in the `docker` group is effectively root on the host, because control of the Docker socket
> is control of the machine. That is the right trade on a dedicated single-purpose box where there
> is one operator, and the wrong one on a shared machine, where `sudo docker` is the better habit.

Create the stacks directory and install the NFS client:

![Reference card, creating the stacks directory and installing NFS and CIFS utilities](assets/docker-host/note-create-stacks-dir-and-nfs-utils.png)

```bash
mkdir -p ~/stacks
sudo apt install -y nfs-common
```

> **The CIFS utilities get left out.** Most guides say to install them, and here that is wrong.

![Reference card, package audit keeping NFS and dropping CIFS](assets/docker-host/note-package-audit-keep-nfs-drop-cifs.png)

| Package | Decision | Reason |
| :--- | :--- | :--- |
| `nfs-common` | **Keep** | Provides `/sbin/mount.nfs`, which is the program Docker calls to attach an NFS volume. Without it the mount cannot happen at all. |
| `cifs-utils` | **Skip** | The Samba client. This host reaches its storage over NFS; the SMB share is for Windows desktops. Installing it pulls in the Samba libraries, which adds packages and therefore possible problems without adding any capability. |

### 4.6 `showmount` and the `/usr/sbin` path

```bash
showmount -e truenas
```

```text
-bash: showmount: command not found
```

![Debian VM, showmount not found](assets/docker-host/showmount-command-not-found.png)

The package did install the binary. Debian puts system administration tools in `/usr/sbin`, and an
ordinary user's default `$PATH` does not include that directory. Other distributions put it on the
path for normal users, which is why this surprises people specifically on Debian:

![Reference card, the default user path omitting /usr/sbin](assets/docker-host/note-usr-sbin-not-in-path.png)

Use `sudo`, or the full path:

```bash
sudo showmount -e 192.168.50.12
# or
/usr/sbin/showmount -e 192.168.50.12
```

![Reference card, adding /usr/sbin to the interactive path](assets/docker-host/note-add-usr-sbin-to-path.png)

```bash
echo 'export PATH=$PATH:/usr/sbin' >> ~/.bashrc && source ~/.bashrc
```

> On a machine managed over SSH, the annoyance of the narrower path outweighs the benefit of it.
> **Adding the directory grants nothing by itself.** Any tool in it that needs elevated rights still
> asks for `sudo`. What is actually lost by leaving it out is that `showmount`, `fdisk`,
> `tcpdump`, `arp`, and `iptables` stop working without being spelled out.

The hostname `truenas` resolving here is the local DNS record from
[3.5](#35-local-names-and-which-machines-should-ask-pi-hole) doing its job.

The export is confirmed:

![Debian VM, showmount listing the NFS export](assets/docker-host/showmount-lists-nfs-export.png)

```text
Export list for truenas:
/mnt/storage/data *
Export list for 192.168.50.12:
/mnt/storage/data *
```

> The `*` means this export is offered to anything that can reach the NAS. I am comfortable with
> that here because the export exists only on a LAN-only address, the data belongs to a service
> account, and the mount that uses it is read-only. If this host were ever reachable from an
> untrusted network, restricting that `*` would be the first thing to change.

### 4.7 Passing the iGPU through for hardware transcoding

Jellyfin transcodes in software by default, which means the CPU does the work. On a 4-core guest
sharing an 8-thread host, a few simultaneous transcodes will make the whole machine feel stuck. The
i7-7700 has an integrated GPU that can do this in hardware, and Proxmox itself has no use for it.
Handing it to the guest costs nothing and takes transcoding off the CPU entirely:

![VM 102 hardware tab, add menu with PCI device selected](assets/docker-host/vm-102-hardware-add-pci-device.png)

Pick the GPU. `0000:00:02.0` is the integrated graphics, and the list shows vendor and device so it
can be confirmed before committing:

![Add PCI device dialog with the integrated graphics device selected](assets/docker-host/add-pci-device-igpu-hd-graphics-630.png)

**ROM-Bar enabled**, all functions off, IDs left to the device:

![Add PCI device dialog with the option ROM enabled](assets/docker-host/add-pci-device-igpu-rom-bar.png)

| Option | Setting | Why |
| :--- | :--- | :--- |
| Raw device | yes, not mapped | The guest drives the device directly. |
| Option ROM | yes | The graphics device needs its firmware blob to start properly. Without it the driver does not load. |
| All functions | no | Claim only the one function. Taking every function on the device can pull in others that are not wanted. |
| Primary GPU | no | Not relevant on a headless host with no passthrough GPU of its own. |

### 4.8 The Jellyfin stack

One folder per stack, with config and cache bind-mounted alongside, so each application's state
sits next to its own compose file and is easy to back up:

```text
~/stacks/
└── jellyfin/
    ├── compose.yaml
    ├── config/
    └── cache/
```

`compose.yaml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1000:1000
    network_mode: host
    restart: unless-stopped
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    volumes:
      - ./config:/config
      - ./cache:/cache
      - truenas_media:/media:ro

volumes:
  truenas_media:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.50.12,ro,noatime,rsize=8192,wsize=8192,tcp,timeo=14,nolock
      device: ":/mnt/storage/data"
```

```bash
cd ~/stacks/jellyfin
docker compose up -d
```

Each line is there for a reason:

![Reference card, a walk through the compose file and the reason for each setting](assets/docker-host/note-compose-file-review.png)

| Directive | Why |
| :--- | :--- |
| `compose.yaml` | The name Docker Compose now prefers, over the older `docker-compose.yml`. |
| `user: 1000:1000` | Runs the process as user and group 1000, which is the host's `haroldlam` account. Files it writes into `./config` and `./cache` are then owned by that user instead of by root, so they can still be edited from the host. Without this, the first thing that happens is a directory full of files nobody can modify. |
| `network_mode: host` | Uses the VM's own network interface directly, skipping the bridge and the port forwarding Docker would otherwise set up. Less overhead, and local discovery protocols like DLNA and mDNS work across the LAN. On a dedicated host running one known service, the usual argument for keeping containers isolated from the host network does not really apply. |
| `restart: unless-stopped` | Brings the container back after a crash or a reboot, but leaves it stopped if it was stopped deliberately. |
| `card0` **and** `renderD128` | Two different device nodes for the same GPU. `renderD128` is the one encoding and decoding actually happens on. `card0` is the display-oriented node. Passing only the first works until some part of the video pipeline goes looking for the second during setup, and then it does not. |
| `/media:ro` | The most important line in the file. Read-only means that even if the container is completely compromised, nothing reached through this mount can be changed, encrypted, or deleted on the NAS. |
| `driver: local` | Uses the volume driver built into Docker. |
| `type: nfs` | Calls the system's NFS support, which is why `nfs-common` had to be installed first. |
| `addr=192.168.50.12` | Where the NAS is. |
| `ro,noatime` | Read-only at the kernel level as well, and skips updating the last-accessed time on every file read, which on a NAS is a great deal of pointless write traffic. |
| `rsize=8192,wsize=8192` | Larger network buffers, for streaming throughput. |
| `tcp,timeo=14,nolock` | Retries over TCP rather than giving up on a dropped packet, a sensible timeout, and no file locking, which a read-only media mount has no use for. |
| `device: ":/mnt/storage/data"` | The export path, matching what `showmount` reported. |

> On `:latest`: it means a pull can bring back a large release with no changelog having been read.
> For the service people actually watch films on, pinning a version and upgrading deliberately is
> the better habit.

### 4.9 What `1000:1000` actually means

This is the most commonly misread line in a Compose file, and the misunderstanding causes real
problems:

![Reference card, the difference between numeric IDs and user names across the host and container](assets/docker-host/note-user-1000-uid-gid-explainer.png)

- The syntax is a **user ID and a group ID**, not a host user and a container user.
- The kernel decides permissions using numbers. It has no concept of the name `haroldlam`.
- Containers share the host kernel, so a process running as ID 1000 that writes through a bind
  mount is recorded on the host as ID 1000. Since `haroldlam` is ID 1000 on the host, ownership
  looks normal from both sides.
- Container images do have users, but only in their own internal user database, usually `root` or
  an application account with a fixed number. That has no bearing here.
- Writing `user: "haroldlam"` would fail, because the container has never heard of that name and
  would refuse to start. Numbers skip the name lookup entirely.

### 4.10 Jellyfin first run

![Jellyfin, welcome screen setting the server name](assets/docker-host/jellyfin-setup-wizard-server-name.png)
![Jellyfin, admin account creation](assets/docker-host/jellyfin-setup-wizard-admin-account.png)

The media libraries point into the read-only mount, so Jellyfin reads from storage it has no ability
to write to:

![Jellyfin, media library folders pointing into the mount](assets/docker-host/jellyfin-setup-wizard-media-libraries.png)
![Jellyfin, metadata language and region](assets/docker-host/jellyfin-setup-wizard-metadata-language.png)
![Jellyfin, remote access setting](assets/docker-host/jellyfin-setup-wizard-remote-access.png)

### 4.11 Boot ordering

Covered in [Topology](#boot-order) because it is a property of the arrangement rather than of
Docker. The short version: TrueNAS starts first with a 70 second delay, and by the time the Docker
host boots, the NFS export is already there to be mounted.

---

## Design decisions

Ten choices worth explaining, collected in one place. Each one is a trade, and each says what was
given up.

**1. The hypervisor uses a public resolver for DNS, skipping both Pi-hole and the router.**
Pi-hole is a guest on that hypervisor, so the host cannot resolve through it. The router is not
much better, because a router forwarding DNS forwards to whatever its own upstream list contains,
and that list starts with Pi-hole. So the host asks `8.8.8.8` directly, which is the only
configuration where the host genuinely does not depend on the guest it runs. This is the one
decision here that is not a preference.

**2. Stripe on the media pool, not mirror.**
1.76 TiB instead of roughly 0.87 TiB, because the contents can be downloaded again and the backups
already cover the failure that a mirror would have covered. On a pool of anything irreplaceable,
this would be the wrong answer.

**3. `virtio-scsi` for every guest, with a serial set on every disk.**
It is the only bus where device names survive adding disks, changing controllers, and moving a
guest. The cost is that serials and SMART data do not pass through by default, which is exactly
the error in [2.6](#26-building-the-pool-and-the-error-worth-understanding). The result is no
temperature data for those drives in TrueNAS.

**4. VirtIO networking rather than an emulated real NIC.**
Lower CPU cost and more throughput on a virtual switch. Emulating a real card would only buy
driver familiarity, which is worth nothing for a guest sitting behind a bridge.

**5. `host` as the CPU type.**
The guest gets the host's processor features directly, with no compatibility flags to reason
about. In exchange, a guest could in principle use an instruction the host does not have, which
matters only when moving between different machines. There is only one machine here.

**6. A container for Pi-hole, a virtual machine for Docker.**
Pi-hole is one process on one port, so a container is faster to start and smaller to patch. Docker
needs a full init, needs the GPU, and benefits from the extra isolation of a VM.

**7. The media mount is read-only at three separate layers.**
`:ro` on the mount, `ro` in the kernel options, and a dataset the container account does not need
to write. Three layers because the media is the largest thing on the machine, and ransomware
reaching it should be structurally impossible rather than merely unlikely.

**8. Local names live in Pi-hole, not in the router.**
The router only knows that Pi-hole exists. Every other lookup, blocking and forwarding included,
happens in one place, so there is no second opinion to contradict the first.

**9. The Pi-hole web interface has no password.**
A documented trade rather than an oversight. It is LAN-only, and the password the installer
generates cannot be recovered if lost. **Set a real password if it ever becomes reachable from
outside the LAN.**

**10. The container's NTP error is left alone.**
The kernel is correctly refusing to let a container move the host clock. The host is already
synchronised, so the message has nothing to report.

---

## Planned: the *arr stack

**Not built yet.** The plan is to run these on the same Docker host, following the same rules
Jellyfin already follows. Notably, the same NFS mount stays **read-only** at `/media`, so these
applications can read the existing library but have no way to write to it. Downloads land on a
separate dataset, and only finished items are moved across to `storage/data`.

| Service | Port | Job |
| :--- | :--- | :--- |
| Portainer | `9000` | Docker management in the browser |
| qBittorrent | `8080` | Downloads |
| Prowlarr | `9696` | Indexers |
| Sonarr | `8989` | TV series |
| Radarr | `7878` | Films |

Each will get its own folder under `~/stacks/`, its own `compose.yaml`, `user: 1000:1000`, and
`restart: unless-stopped`, matching what is already deployed.
