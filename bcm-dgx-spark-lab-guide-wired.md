# A BCM lab inside one DGX Spark, walled off from your LLM work

> [!NOTE]
> This is the **wired** version of the guide. If your Spark uses its Wi-Fi uplink instead, use [bcm-dgx-spark-lab-guide-wifi.md](bcm-dgx-spark-lab-guide-wifi.md).

Run a complete NVIDIA Base Command Manager cluster — head node plus PXE-provisioned compute nodes — as KVM virtual machines on a fixed slice of CPU cores and memory, while the GB10 GPU and the rest of the machine stay free for models and tools.

⬛ DGX OS host and hypervisor · 🟩 BCM partition (VMs) · 🟧 Workload partition (GPU, LLMs)

*Every step ends with a **Diagnose this step** panel: the checks to run when that step doesn't behave, taken from a real install on a DGX Spark.*

## Contents

- ⬛ [Architecture](#arch)
- ⬛ [Resource budget](#budget)
- ⬛ [1. Survey the host](#s1)
- ⬛ [2. Install KVM and libvirt](#s2)
- ⬛ [3. Build the isolation boundary](#s3)
- 🟩 [4. Create the lab networks](#s4)
- 🟩 [5. Stage storage and the ISO](#s5)
- 🟩 [6. Create the head node VM](#s6)
- 🟩 [7. Run the BCM installer](#s7)
- 🟩 [8. Post-install and licensing](#s8)
- 🟩 [9. Provision compute node VMs](#s9)
- 🟩 [10. Power control from BCM](#s10)
- 🟧 [11. Run LLMs in the workload partition](#s11)
- ⬛ [Day-2 operations](#ops)
- ⬛ [Tear down the lab](#teardown)
- ⬛ [Troubleshooting](#trouble)
- ⬛ [Limits of this design](#caveats)

<a id="arch"></a>

## A. Architecture

> *What you are building before touching anything.*

DGX OS stays the only operating system on the metal. KVM turns it into a hypervisor; libvirt manages the VMs. Two cgroup boundaries split the 20 Arm cores and 128 GB of unified memory: one slice holds every BCM VM, the other is where containers using the GPU run. BCM never sees the Spark itself — it manages only its own virtual cluster, on a private network that has no path to your LAN.

![Architecture diagram: DGX Spark host running DGX OS and KVM, split into a BCM partition with a head node VM and two compute node VMs on an isolated internal network, and a workload partition with the GB10 GPU and LLM containers.](assets/architecture-wired.svg)

*Solid line: NAT network the head node uses for internet (licensing, updates). Dashed line: the private provisioning network BCM owns completely.*

> [!WARNING]
> **Know the support position before you start.**
>
> CPU-only KVM guests run fine on DGX Spark in practice, but NVIDIA has stated in its developer forum that virtualization is not a supported use of the platform, though it is not actively disabled. GPU passthrough to a VM does not work. That is exactly why this design keeps the GPU on the host. Treat this as a lab, not a pattern for anything customer-facing.

<a id="budget"></a>

## B. Resource budget

> *Decide the split once; every later command uses these numbers.*

The GB10's memory is unified: every gigabyte a VM holds is a gigabyte the GPU cannot use for weights or KV cache. BCM's documented floor for an Arm head node is 16 GB RAM and 80 GB of disk, so the head node is the largest single cost. Pick one profile.

| Slice | Lean profile | Comfortable profile (used below) |
|---|---|---|
| ⬛ Host reserve | cores 0-3 · ~8 GB | cores 0-3 · ~8 GB |
| 🟩 bcm-head | 4 vCPU · 16 GB · 100 GB disk | 4 vCPU · 20 GB · 120 GB disk |
| 🟩 Compute nodes | 2 × (1 vCPU · 3 GB · 40 GB) | 2 × (2 vCPU · 4 GB · 60 GB) |
| 🟩 BCM slice cap | cores 4,10-12 · MemoryMax=24G | cores 4,10-14 · MemoryMax=34G |
| 🟧 Left for LLMs | cores 5-9,13-19 · ~96 GB | cores 5-9,15-19 · ~86 GB |

The comfortable profile puts 8 vCPUs on 6 cores. That is optional: it gives the head node 4 vCPUs so the installer and image builds run faster, and a management cluster idles most of the time, so the overlap rarely matters. To avoid it, give the head node 2 vCPUs (2 + 2 + 2 = 6), or widen the slice to 8 cores at the cost of host or workload cores.

> [!NOTE]
> **The partition is elastic.**
>
> Shutting the BCM VMs down returns their memory to the pool immediately. When you want to load a model that needs everything, run `virsh shutdown` on the lab and start it again afterwards.

<a id="s1"></a>

## ⬛ 1. Survey the host

> *Touches: DGX OS host only. Nothing is changed in this step.*

### Confirm hardware virtualization is exposed

```bash
# Should list /dev/kvm. If missing, stop here: the kernel module is not loaded.
ls -l /dev/kvm
lsmod | grep kvm
uname -r          # note the kernel; DGX OS updates can change it
free -g           # expect ~119 GB usable of 128 GB
df -h /var/lib    # VM disks land here; need ~250 GB free
```

### Map the two kinds of core

The GB10 CPU has ten performance cores (Cortex-X925) and ten efficiency cores (Cortex-A725), arranged in two clusters of five of each. Put the host and BCM on efficiency cores and give every performance core to the workload side. Confirm the numbering on your unit:

```bash
lscpu -e=CPU,CORE,MAXMHZ
# Cores with the higher MAXMHZ are the X925 performance cores.
# Cross-check by part number: 0xd85 = X925, 0xd87 = A725
grep -E 'processor|CPU part' /proc/cpuinfo | paste - -
```

On the Spark this guide was built on, the output is:

| Core type | CPU part | CPUs |
|---|---|---|
| 🟩 Efficiency (Cortex-A725) | `0xd87` | 0-4, 10-14 |
| 🟧 Performance (Cortex-X925) | `0xd85` | 5-9, 15-19 |

So the split used throughout this guide is:

| Slice | CPUs | Core type |
|---|---|---|
| ⬛ Host reserve | `0-3` | 4 efficiency |
| 🟩 BCM (machine.slice) | `4,10-14` | 6 efficiency |
| 🟧 Workloads (containers) | `5-9,15-19` | 10 performance |

> [!NOTE]
> **Different unit, different numbers.**
>
> If your output doesn't match the table above, keep the same logic — host on four efficiency cores, BCM on the remaining six, workloads on all ten performance cores — and replace `0-3`, `4,10-14` and `5-9,15-19` everywhere below.

### Identify the network interfaces

```bash
ip -br link
nmcli device status
```

Note the name of the wired 10 GbE port (something like `enP7s7`). Only a wired port can be bridged; Wi-Fi cannot. The two ConnectX-7 QSFP ports are not needed for this lab.

### Verify this step

KVM is available, and the CPU and memory are what the budget assumes:

```bash
ls -l /dev/kvm
nproc
free -g | awk '/Mem:/{print $2" GB total"}'
```

**Expected output**

```text
crw-rw---- 1 root kvm 10, 232 Sep 29 08:00 /dev/kvm
20
119 GB total
```

Core types match the split used in this guide:

```bash
grep 'CPU part' /proc/cpuinfo | awk '{print $4}' | paste -sd' '
```

**Expected output**

```text
0xd87 0xd87 0xd87 0xd87 0xd87 0xd85 0xd85 0xd85 0xd85 0xd85 0xd87 0xd87 0xd87 0xd87 0xd87 0xd85 0xd85 0xd85 0xd85 0xd85
```

Positions 0–4 and 10–14 are `0xd87` (efficiency); 5–9 and 15–19 are `0xd85` (performance).

The wired port is up with a LAN address:

```bash
ip -4 -br addr
ip route | grep default
```

**Expected output**

```text
enP7s7   UP   <your LAN address>/24
default via <your router> dev enP7s7 proto dhcp metric 100
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`/dev/kvm` doesn't exist**

The KVM module isn't loaded. Check the kernel log, then try loading it; if it still fails, virtualization isn't available on this kernel and nothing later will work.

```bash
dmesg | grep -i kvm
sudo modprobe kvm
ls -l /dev/kvm
```

**`lscpu` shows no MAXMHZ column or identical values**

Use the part-number method instead: `0xd87` is an efficiency core (A725), `0xd85` a performance core (X925).

```bash
grep -E 'processor|CPU part' /proc/cpuinfo | paste - -
```

**You can't tell which interface is the wired port**

```bash
ip -br link
ethtool <iface> | grep -E 'Speed|Link detected'
```

The wired port reports `Link detected: yes` and a speed such as 10000Mb/s when a cable is connected.

</details>

<a id="s2"></a>

## ⬛ 2. Install KVM and libvirt

> *Touches: DGX OS host. Adds packages, groups and services.*

```bash
sudo apt update
sudo apt install -y qemu-system-arm qemu-efi-aarch64 qemu-utils \
  libvirt-daemon-system libvirt-clients virtinst virt-manager virt-viewer \
  ovmf cpu-checker

sudo usermod -aG libvirt,kvm $USER
sudo systemctl enable --now libvirtd
# Log out and back in (or reboot) so the group membership applies.
```

### Verify

```bash
virsh -c qemu:///system version
virt-host-validate qemu
# WARN lines about IOMMU, secure guest or the cgroup 'devices' controller are fine.
# FAIL on "hardware virtualization" or /dev/kvm is not.

ls /usr/share/AAVMF/          # Arm UEFI firmware the VMs will boot from
```

The listing includes two kinds of firmware. Files with `.ms` or `secboot` in the name enforce Secure Boot with Microsoft keys; the plain `AAVMF_CODE.fd` and `AAVMF_VARS.fd` do not. The BCM boot loader isn't signed for Secure Boot, so every VM in this guide is created with the plain pair (step 6).

### Bring up the default NAT network

Installing libvirt creates a NAT network called `default`; the head node's external NIC will use it. On a fresh Spark it is often defined but inactive.

```bash
sudo virsh net-list --all
sudo virsh net-start default
sudo virsh net-autostart default
```

If `default` is missing from the list entirely, recreate it from the copy libvirt ships, then run the two commands above:

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
```

### Verify this step

libvirt and QEMU are installed and answering:

```bash
virsh -c qemu:///system version
```

**Expected output**

```text
Compiled against library: libvirt 10.0.0
Using library: libvirt 10.0.0
Using API: QEMU 10.0.0
Running hypervisor: QEMU 8.2.2
```

Version numbers may be newer after updates; the point is that all four lines appear.

Your user is in the right groups (after logging out and back in):

```bash
id -nG | tr ' ' '\n' | grep -E '^(libvirt|kvm)$'
```

**Expected output**

```text
kvm
libvirt
```

The plain (non-Secure-Boot) firmware exists:

```bash
ls /usr/share/AAVMF/AAVMF_CODE.fd /usr/share/AAVMF/AAVMF_VARS.fd
```

**Expected output**

```text
/usr/share/AAVMF/AAVMF_CODE.fd  /usr/share/AAVMF/AAVMF_VARS.fd
```

The NAT network is running and starts at boot:

```bash
sudo virsh net-list --all
```

**Expected output**

```text
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`virt-host-validate`: WARN “Checking for cgroup 'devices' controller support”**

Harmless. DGX OS uses cgroup v2, which has no `devices` controller; libvirt uses eBPF for device control instead. Don't switch the kernel back to cgroup v1 — it would break the slice limits in step 3. Confirm the controllers step 3 needs are present:

```bash
stat -fc %T /sys/fs/cgroup/            # cgroup2fs
cat /sys/fs/cgroup/cgroup.controllers   # must include cpuset and memory
```

**`virt-host-validate`: WARN about IOMMU or secure guest**

Also harmless for this lab: nothing is passed through to the VMs. Only a **FAIL** on hardware virtualization or `/dev/kvm` needs fixing.

**`virsh` says permission denied without `sudo`**

The new `libvirt` group membership only applies to new logins. Log out and back in, or keep using `sudo`. Check with `id | grep libvirt`.

**`default` is missing from `virsh net-list --all`**

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

**`AAVMF_CODE.fd` or `AAVMF_VARS.fd` is missing**

```bash
sudo apt install --reinstall -y qemu-efi-aarch64
ls /usr/share/AAVMF/
```

</details>

<a id="s3"></a>

## ⬛ 3. Build the isolation boundary

> *Touches: systemd cgroups and libvirt hooks on the host. This is what makes the BCM side a partition rather than just some VMs.*

libvirt starts every VM inside `machine.slice`. Capping that slice caps the whole BCM lab at once, regardless of how many VMs you add later. Keeping host and container processes off the BCM cores makes the fence work in both directions.

### Cap the BCM slice

```bash
# Persistent: systemd writes a drop-in under /etc/systemd/system.control/
sudo systemctl set-property machine.slice AllowedCPUs=4,10-14 MemoryMax=34G

# Keep everything else off the BCM cores
sudo systemctl set-property system.slice AllowedCPUs=0-3,5-9,15-19
sudo systemctl set-property user.slice   AllowedCPUs=0-3,5-9,15-19

systemctl show machine.slice -p AllowedCPUs -p MemoryMax
```

Docker containers run under `system.slice`, so they automatically stay on cores 0-3, 5-9 and 15-19. Step 11 narrows them further to the performance cores, 5-9 and 15-19.

### Protect the VMs from the out-of-memory killer

If a model loads more than expected, the kernel will pick a victim. Make sure it is never the head node. A libvirt hook lowers each QEMU process's OOM score as it starts.

```bash
sudo tee /etc/libvirt/hooks/qemu >/dev/null <<'EOF'
#!/bin/bash
# $1 = domain name, $2 = operation
if [ "$2" = "started" ]; then
  pid=$(cat "/run/libvirt/qemu/$1.pid" 2>/dev/null)
  [ -n "$pid" ] && echo -800 > "/proc/$pid/oom_score_adj"
fi
exit 0
EOF
sudo chmod 755 /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd   # libvirt only discovers new hooks on restart
```

Each VM will also be created with locked memory (step 6), so its RAM is committed up front and never swapped or reclaimed by a hungry model.

> [!NOTE]
> **Optional: turn off page merging.**
>
> If `cat /sys/kernel/mm/ksm/run` prints 1, set it to 0. Merging pages across VMs saves little in a two-node lab and adds CPU churn on the host cores.

### Verify this step

Each slice has the limits you set:

```bash
systemctl show machine.slice -p AllowedCPUs -p MemoryMax
systemctl show system.slice  -p AllowedCPUs
systemctl show user.slice    -p AllowedCPUs
```

**Expected output**

```text
AllowedCPUs=4,10-14
MemoryMax=36507222016
AllowedCPUs=0-3,5-9,15-19
AllowedCPUs=0-3,5-9,15-19
```

`MemoryMax` is shown in bytes: 36507222016 = 34 GiB.

The OOM hook is installed and executable:

```bash
ls -l /etc/libvirt/hooks/qemu
```

**Expected output**

```text
-rwxr-xr-x 1 root root 213 Sep 29 08:10 /etc/libvirt/hooks/qemu
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**Check that the fence is really in place**

```bash
systemctl show machine.slice -p AllowedCPUs -p MemoryMax
cat /sys/fs/cgroup/machine.slice/cpuset.cpus.effective   # 4,10-14 once a VM runs
cat /sys/fs/cgroup/system.slice/cpuset.cpus.effective    # 0-3,5-9,15-19
```

**`AllowedCPUs` shows the value but `cpuset.cpus.effective` doesn't**

The cpuset controller isn't enabled for that part of the tree. Check it's listed in the root's subtree control; it normally is on DGX OS once a slice sets `AllowedCPUs`.

```bash
cat /sys/fs/cgroup/cgroup.subtree_control   # should include cpuset
```

**You set the wrong ranges**

```bash
sudo systemctl revert machine.slice system.slice user.slice
sudo systemctl daemon-reload
```

Then run the `set-property` commands again with the right values.

**Check the OOM protection hook works (once a VM is running)**

```bash
cat /proc/$(pgrep -f 'guest=bcm-head')/oom_score_adj   # -800
```

If it shows 0, confirm the hook is executable (`ls -l /etc/libvirt/hooks/qemu`) and restart libvirtd; it only picks up new hooks on restart.

</details>

<a id="s4"></a>

## 🟩 4. Create the lab networks

> *Touches: libvirt networks on the host. Your LAN is untouched unless you choose the optional bridge.*

### External side: reuse the NAT network

The head node's external NIC goes on libvirt's `default` network (192.168.122.0/24, gateway 192.168.122.1). Unlike the internal network, it needs no XML of its own: libvirt created it at install time and step 2 started it. Its outbound traffic is NATed out through the Spark's wired port, which gives it internet access for licensing and repositories, and you reach it from the Spark directly or from your laptop through an SSH jump.

Check that it matches what the head node will expect:

```bash
sudo virsh net-dumpxml default
```

```xml
<!-- expected, roughly -->
<network>
  <name>default</name>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

#### Reserve the head node's address

The head node uses a static 192.168.122.10, which sits inside libvirt's DHCP range. Reserve it for the head node's MAC so no other VM is ever handed the same address. (Skip this if you bridge the head node onto your LAN below.)

```bash
sudo virsh net-update default add ip-dhcp-host \
  "<host mac='52:54:00:bc:00:01' name='bcm-head' ip='192.168.122.10'/>" \
  --live --config

sudo virsh net-dumpxml default | grep bcm-head   # reservation is present
```

### Internal side: an isolated provisioning network

BCM runs its own DHCP and PXE on the internal network. This network must have no libvirt DHCP, no forwarding and no host IP, so BCM owns it completely and nothing leaks out.

```bash
cat > /tmp/bcm-internal.xml <<'EOF'
<network>
  <name>bcm-internal</name>
  <bridge name="virbr-bcmi" stp="off" delay="0"/>
  <mtu size="1500"/>
</network>
EOF
sudo virsh net-define /tmp/bcm-internal.xml
sudo virsh net-start bcm-internal
sudo virsh net-autostart bcm-internal
sudo virsh net-dumpxml bcm-internal   # confirm: no <ip>, no <forward>
```

### Optional: bridge the head node onto your LAN

Only do this if you want the head node to have its own LAN address instead of sitting behind NAT. Run it from the Spark's local console, not over SSH, because the wired connection drops while it moves onto the bridge.

```bash
NIC=enP7s7   # your wired port from step 1
sudo nmcli con add type bridge ifname br0 con-name br0 stp no ipv4.method auto ipv6.method auto
sudo nmcli con add type bridge-slave ifname $NIC master br0 con-name br0-port
sudo nmcli con down "$(nmcli -g GENERAL.CONNECTION device show $NIC)"
sudo nmcli con up br0
ip -br addr show br0
```

If you use the bridge, replace `network=default` with `bridge=br0` in step 6 and give the head node a LAN address in step 7.

### Verify this step

Both lab networks are active and autostart:

```bash
sudo virsh net-list --all
```

**Expected output**

```text
 Name           State    Autostart   Persistent
--------------------------------------------------
 bcm-internal   active   yes         yes
 default        active   yes         yes
```

The internal network has no IP, DHCP or forwarding, and the host has no address on it:

```bash
sudo virsh net-dumpxml bcm-internal | grep -cE '<ip|<forward|<dhcp'
ip -4 addr show virbr-bcmi | grep -c inet
```

**Expected output**

```text
0
0
```

The head node's address is reserved on the NAT network:

```bash
sudo virsh net-dumpxml default | grep bcm-head
```

**Expected output**

```text
      <host mac='52:54:00:bc:00:01' name='bcm-head' ip='192.168.122.10'/>
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`net-start default` fails with “network is already in use by interface”**

Your LAN or another network already uses 192.168.122.0/24. Edit the network to a free range (for example 172.30.122.0/24) with `sudo virsh net-edit default`, and use that range for the head node's external address in step 7.

**`net-update … add ip-dhcp-host` fails because an entry exists**

```bash
sudo virsh net-dumpxml default | grep host
sudo virsh net-update default delete ip-dhcp-host \
  "<host mac='52:54:00:bc:00:01' name='bcm-head' ip='192.168.122.10'/>" --live --config
```

Then add it again.

**`net-dumpxml bcm-internal` shows an `<ip>` or `<forward>` element**

libvirt would run its own DHCP there and fight BCM's. Destroy, undefine and redefine it from the XML above.

```bash
sudo virsh net-destroy bcm-internal
sudo virsh net-undefine bcm-internal
```

**The Spark lost its network after creating `br0`**

From the local console:

```bash
sudo nmcli con delete br0-port br0
sudo nmcli device connect <wired-port>
```

</details>

<a id="s5"></a>

## 🟩 5. Stage storage and the ISO

> *Touches: host filesystem.*

### Download the right ISO

From the BCM ISO download portal, use your product key and choose:

- The current BCM 11 release.
- Architecture **aarch64 / arm64** — the VMs are Arm, like the Spark.
- Base distribution **Ubuntu 24.04**.
- A generic hardware vendor. The VMs are not DGX systems, so skip the DGX software stack option.

> [!WARNING]
> **The x86 and Arm ISOs look almost identical.**
>
> Both editions can arrive with names like `bcm-11.0-ubuntu2404.iso`. An x86 ISO attached to an Arm VM fails silently: the CD-ROM appears in the boot menu but selecting it does nothing. Rename every ISO to include its architecture as soon as it arrives, and verify it below before creating any VM.

### Get the ISO onto the Spark

Pick whichever is easiest. Downloading directly on the Spark avoids a second multi-gigabyte copy over Wi-Fi or the LAN.

#### Option A: download directly on the Spark

Use the browser on the Spark's desktop, or enter your product key in a browser on another machine, right-click the download button, choose **Copy link address**, and fetch it over SSH straight away (the links expire):

```bash
cd ~
wget -c -O bcm11-arm64.iso "PASTE-THE-DOWNLOAD-LINK-HERE"   # -c resumes if it drops
```

If the result is a small HTML file rather than several gigabytes, the link needed a browser session or had expired; copy a fresh one, or use the Spark's own browser.

#### Option B: copy from a Windows PC (PowerShell)

```powershell
# Find the ISO in Downloads and note its checksum
$iso = Get-ChildItem "$env:USERPROFILE\Downloads\bcm-11*.iso"
$iso.FullName
Get-FileHash $iso.FullName -Algorithm SHA256

# Copy it to your home folder on the Spark, renamed on the way
scp $iso.FullName <spark-user>@<spark-ip>:~/bcm11-arm64.iso
```

PowerShell doesn't expand `*` for `scp`, which is why the path is looked up first. If `$iso.FullName` prints more than one file, pass the exact filename instead. WinSCP (SFTP to the Spark) does the same by drag and drop.

#### Option C: copy from a Mac (Terminal)

```bash
shasum -a 256 ~/Downloads/bcm-11*.iso
scp ~/Downloads/bcm-11*.iso <spark-user>@<spark-ip>:~/bcm11-arm64.iso
```

### Put it in place

Your Spark user can't write into libvirt's image folder, so every option lands in your home folder first and is then moved with `sudo`. VMs run as `libvirt-qemu`, so hand the file to that user.

```bash
sudo mkdir -p /var/lib/libvirt/images/iso
sudo mv ~/bcm11-arm64.iso /var/lib/libvirt/images/iso/bcm11-arm64.iso
sudo chown libvirt-qemu:kvm /var/lib/libvirt/images/iso/bcm11-arm64.iso
ls -lh /var/lib/libvirt/images/iso/
```

### Verify the checksum and the architecture

The checksum proves the file arrived intact; the second check proves it is the Arm build. The ISO's own `EFI` folder isn't the one the firmware boots from — installer ISOs boot from the small `efi.img` embedded in them, so look inside that.

```bash
ISO=/var/lib/libvirt/images/iso/bcm11-arm64.iso
sha256sum "$ISO"                     # compare with the download portal

sudo umount /mnt 2>/dev/null         # clear any stale mount first
sudo mount -o loop,ro "$ISO" /mnt
sudo mkdir -p /tmp/efiimg
sudo mount -o loop,ro /mnt/efi.img /tmp/efiimg
find /tmp/efiimg -type f -iname "*.efi" | xargs -r file
sudo umount /tmp/efiimg; sudo umount /mnt
```

**Expected output** (checksum and size will differ by release)

```text
<64-character hex checksum>  /var/lib/libvirt/images/iso/bcm11-arm64.iso
/tmp/efiimg/EFI/BOOT/bootaa64.efi: PE32+ executable (EFI application) Aarch64 (stripped to external PDB), for MS Windows, 4 sections
```

For comparison, the x86 edition shows `bootx64.efi: PE32+ executable (EFI application) x86-64`. While the ISO is mounted, `ls /mnt` on a BCM 11 Arm ISO looks like this:

```text
3rd-party-licenses.pdf  boot  boot.catalog  BRIGHTDATA.md5  data  EFI  efi.img  grub  OSS-Written-Offer.pdf  README.ADDON  README.BRIGHTUSB  README.MODIFY.BRIGHTISO.md
```

Two things in that listing trip people up: the ISO's own `EFI` folder has no `.efi` file near the top, and the kernel isn't called `vmlinuz`. Both are normal — the firmware boots from `efi.img`, which is why the check looks inside it.

You want `BOOTAA64.EFI` or `bootaa64.efi` described as `Aarch64`. `bootx64.efi` or `x86-64` means the x86 edition: download the aarch64 one before going further.

### ISO verification at a glance

| Check | Command | Pass | Fail means |
|---|---|---|---|
| Complete download | `ls -lh "$ISO"` | Same size as listed on the portal (several GB) | Truncated transfer: fetch again (`wget -c` resumes) |
| Intact file | `sha256sum "$ISO"` | Matches the portal's checksum exactly | Corrupted download or copy |
| Readable ISO | `sudo mount -o loop,ro "$ISO" /mnt` | Mounts without error; `ls /mnt` shows `EFI`, `efi.img`, `boot`, `data`, `grub` | “wrong fs type” / “can't read superblock”: damaged file |
| Arm build | `find /tmp/efiimg … \| xargs file` | `bootaa64.efi … Aarch64` | `bootx64.efi … x86-64`: x86 edition, won't boot |
| Right file in the VM | `sudo virsh domblklist bcm-head --details` | The `cdrom` line shows the file you just verified | A different ISO is attached (step 6 diagnostics) |

> [!NOTE]
> **Licensing.**
>
> You need a BCM product key both to download and to activate the cluster in step 8. NVIDIA has offered no-cost BCM licensing in the past; check the current terms on the download portal or with your NVIDIA contact.

### Verify this step

The ISO is in place, owned by the QEMU user:

```bash
ls -lh /var/lib/libvirt/images/iso/
```

**Expected output**

```text
-rw-r--r-- 1 libvirt-qemu kvm 5.1G Sep 29 07:40 bcm11-arm64.iso
```

The size is illustrative; it must match the portal. There should be no other BCM ISO here unless its name says which architecture it is.

The checksum and architecture checks above, with the table, complete this step.

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`mount` says “already mounted”**

An earlier mount is still active, and it may be of an older file with the same name — its contents can mislead you. Unmount until it says “not mounted”, then mount again.

```bash
mount | grep /mnt
sudo umount /mnt; sudo umount /mnt
```

**`ls /mnt/EFI/BOOT` fails or `find` shows no `.efi` files in the ISO**

Expected on BCM ISOs: folder names may be lowercase (`EFI/boot`), and the boot loader lives inside `/mnt/efi.img`. Use the `efi.img` check above.

**`file /mnt/boot/vmlinuz*` finds nothing**

The installer kernel has a different name. Let `file` identify everything instead:

```bash
find /mnt/boot /mnt/EFI -type f | head -40 | xargs -r file
```

**You're not sure which ISO file is which**

```bash
sudo find / -xdev -iname "*bcm*.iso" -exec ls -lh {} \; 2>/dev/null
```

Check each one's `efi.img`, then rename them to include the architecture, for example `…-x86_64.iso` and `…-arm64.iso`.

**Checksum doesn't match the portal**

Compare with the checksum from the machine you downloaded on (`Get-FileHash` or `shasum`). If that one matches the portal, the copy to the Spark was damaged — copy again. If neither matches, download again. If the portal lists MD5, use `md5sum`.

**`mount` fails with “wrong fs type” or “can't read superblock”**

The file is incomplete. `ls -lh` it and compare with the size on the portal, then fetch it again (`wget -c` resumes).

</details>

<a id="s6"></a>

## 🟩 6. Create the head node VM

> *Touches: a new VM inside machine.slice.*

Three choices in this command matter:

- **Firmware without Secure Boot.** Plain `--boot uefi` makes libvirt pick the Secure Boot firmware (`AAVMF_CODE.ms.fd`), which silently refuses to run the BCM boot loader: selecting the CD-ROM just returns to the menu. The `loader=` and `nvram.template=` options name the plain firmware instead.
- **CD-ROM first, then disk.** The VM boots straight into the installer; once BCM is installed and the ISO is ejected (step 7), it falls through to the disk.
- **Fixed MAC addresses and SCSI disks.** The MACs make NICs easy to match in the installer (explained below), and virtio-SCSI disks appear as `/dev/sda`, which BCM's default disk layouts expect.

### Why the MAC addresses start with 52:54:00:bc

Every NIC in the lab gets a MAC address chosen by hand instead of a random one. That makes each NIC recognisable at a glance in the BCM installer, in `cmsh`, in DHCP logs and in `virsh` output, and lets the same address appear in four places without looking it up: the `virt-install` command, the libvirt DHCP reservation (step 4), the installer's interface mapping (step 7) and `cmsh set mac` for the nodes (step 9).

| Bytes | Value | Why |
|---|---|---|
| 1–3 | `52:54:00` | The prefix QEMU/KVM uses for virtual NICs; libvirt generates `52:54:00:xx:xx:xx` itself when no MAC is given. In the first byte, `0x52` = `0101 0010`: bit 1 set means *locally administered* (assigned by software, never burned into real hardware), and bit 0 clear means *unicast*. So these addresses can never collide with a physical NIC on your network. |
| 4 | `bc` | “BCM”. A lab-wide tag, so every lab NIC matches one search: `virsh dumpxml … \| grep 52:54:00:bc`. Other VMs you create later keep libvirt's random addresses and never clash. |
| 5 | `00` / `01` | Which network: `00` = external (libvirt NAT), `01` = internal (BCM provisioning). |
| 6 | `01`, `11`, `12`… | Which machine: `01` = head node, `11` onwards = node001, node002…, leaving room for a second head node (`02`) if you try HA later. |

| MAC | Machine | Network | Linux name in the VM |
|---|---|---|---|
| 🟩 `52:54:00:bc:00:01` | bcm-head | externalnet (libvirt `default`) | `enp1s0` |
| 🟩 `52:54:00:bc:01:01` | bcm-head | internalnet (`bcm-internal`) | `enp2s0` |
| 🟩 `52:54:00:bc:01:11` | node001 | internalnet | — |
| 🟩 `52:54:00:bc:01:12` | node002 | internalnet | — |

Any values work as long as each MAC is unique and used consistently. If you change the scheme, change it everywhere it appears.

```bash
sudo virt-install \
  --name bcm-head \
  --osinfo ubuntu24.04 \
  --arch aarch64 --machine virt \
  --boot cdrom,hd,loader=/usr/share/AAVMF/AAVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/AAVMF/AAVMF_VARS.fd \
  --cpu host-passthrough \
  --vcpus 4,cpuset=4,10-14 \
  --memory 20480 \
  --memorybacking locked=on \
  --controller type=scsi,model=virtio-scsi \
  --disk path=/var/lib/libvirt/images/bcm-head.qcow2,size=120,format=qcow2,bus=scsi,cache=none,discard=unmap \
  --cdrom /var/lib/libvirt/images/iso/bcm11-arm64.iso \
  --network network=default,model=virtio,mac=52:54:00:bc:00:01 \
  --network network=bcm-internal,model=virtio,mac=52:54:00:bc:01:01 \
  --graphics vnc,listen=127.0.0.1 --video virtio \
  --input tablet,bus=usb --input keyboard,bus=usb \
  --console pty,target_type=serial \
  --noautoconsole
```

### Open the console

The VM has a virtual graphics card. QEMU draws that card's screen into memory and publishes it as a **VNC server**; `--graphics vnc,listen=127.0.0.1` makes that server listen only on the Spark itself, on port 5900 for the first VM (display `:0`), 5901 for the next (`:1`) and so on. Anything that shows you the VM's screen is a VNC client of that server. The VM also has a **serial port** wired to a text console, which needs no graphics at all.

```text
VM's virtual GPU (virtio / ramfb) ──► QEMU VNC server 127.0.0.1:5900 on the Spark
                                            ▲
            virt-viewer / virt-manager ─────┤  (libvirt clients, find the port for you)
            TigerVNC via SSH tunnel ────────┘  (plain VNC client, any OS)

VM's serial port ──► virsh console bcm-head   (text only, over any SSH session)
```

| Tool | Runs on | How it reaches the VM | Use it when |
|---|---|---|---|
| **virt-viewer** | The Spark's own desktop | Asks libvirt for the VNC port and connects locally | A monitor is attached to the Spark |
| **virt-manager** | A Linux desktop (the Spark, or a Linux PC) | Full libvirt GUI; connects over `qemu+ssh://` and opens the VNC console for you | You want a GUI to manage VMs, not just see one. Not practical on macOS or Windows. |
| **TigerVNC** (or RealVNC) | Mac, Windows or Linux | Plain VNC client; reaches `127.0.0.1:5900` through an SSH tunnel | You work from a Mac or Windows PC — the method used for this lab |
| **virsh console** | Any SSH session to the Spark | Attaches to the VM's serial port | The screen is blank, you only have a terminal, or a boot menu prints to serial |

#### The VM's display device

`--video virtio` gives the VM a paravirtual GPU that the UEFI firmware and Linux both drive. On some Arm firmware builds the firmware can't draw on it, so the screen stays black until Linux starts; the `ramfb` device works from power-on. If you see a black screen at boot, switch to `ramfb` (see *Diagnose this step*).

#### On the Spark's own desktop: virt-viewer

```bash
virt-viewer --connect qemu:///system bcm-head
```

#### From a Mac or Windows PC: TigerVNC through an SSH tunnel

virt-manager isn't needed. The tunnel forwards a port on your computer to the VM's VNC port on the Spark, so nothing is exposed on your network. Local port 5901 avoids clashing with the Mac's own Screen Sharing on 5900.

```bash
# 1. On the Spark: which VNC display is the VM on?
sudo virsh vncdisplay bcm-head
```

**Expected output**

```text
127.0.0.1:0
```

`:0` means port 5900, `:1` port 5901, and so on. Use that port on the right-hand side of the tunnel.

```bash
# 2. On your Mac/PC, in its own Terminal window (leave it open; it prints nothing)
ssh -N -o ServerAliveInterval=30 -L 5901:127.0.0.1:5900 <spark-user>@<spark-ip>
```

In a second Terminal tab (<kbd>Cmd</kbd>+<kbd>T</kbd> on a Mac), install a viewer once, then open it and connect to `localhost:5901`:

```bash
brew install --cask tigervnc-viewer     # Mac, one time; or download TigerVNC / RealVNC Viewer
lsof -nP -iTCP:5901 -sTCP:LISTEN        # Mac: confirms the tunnel is listening
```

**Expected output** of the `lsof` check

```text
COMMAND  PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
ssh     4242  you    5u  IPv4  0x...      0t0  TCP 127.0.0.1:5901 (LISTEN)
```

macOS's built-in Screen Sharing often refuses QEMU's password-less VNC, so use a standalone viewer. On Windows, run the same `ssh -N -L …` command in PowerShell and use TigerVNC or RealVNC Viewer. Click inside the viewer window before typing so it has keyboard focus. If the viewer says *Connection refused*, the tunnel has closed; start it again.

#### Serial console: connect and exit

The VM was created with `--console pty,target_type=serial`, so its first serial port is a text console you can open from any SSH session on the Spark:

```bash
sudo virsh console bcm-head --escape '^Q'
```

**Expected output**

```text
Connected to domain 'bcm-head'
Escape character is ^Q (Ctrl + Q)
```

- **Nothing appears after connecting:** press <kbd>Enter</kbd> once. The console only shows new output; a boot menu or login prompt that was drawn earlier isn't repeated.
- **Exit:** <kbd>Control</kbd>+<kbd>Q</kbd> with the `--escape '^Q'` option above. Without it, the default is <kbd>Control</kbd>+<kbd>]</kbd>; on some non-US keyboards that key is <kbd>Control</kbd>+<kbd>5</kbd>. Exiting only disconnects you — the VM keeps running.
- **“Active console session exists for this domain”:** another terminal is still attached. Close it, or take over with `sudo virsh console bcm-head --force`.
- **Last resort:** close the Terminal tab. The VM is unaffected, and you can reconnect at any time.

Once BCM is installed, the serial console shows the head node's login prompt, which is handy if a network change locks you out of SSH.

### Confirm it is fenced

```bash
systemd-cgls -u machine.slice | head
virsh vcpupin bcm-head         # every vCPU should show 4,10-14
systemd-cgtop -1 machine.slice
```

### Verify this step

The VM is running:

```bash
sudo virsh list --all
```

**Expected output**

```text
 Id   Name       State
--------------------------
 1    bcm-head   running
```

The right ISO and disk are attached:

```bash
sudo virsh domblklist bcm-head --details
```

**Expected output**

```text
 Type   Device   Target   Source
-------------------------------------------------------------------
 file   disk     sda      /var/lib/libvirt/images/bcm-head.qcow2
 file   cdrom    sdb      /var/lib/libvirt/images/iso/bcm11-arm64.iso
```

Plain firmware, no Secure Boot, CD-ROM first:

```bash
sudo virsh dumpxml bcm-head | grep -iE "loader|nvram|secure|<boot"
```

**Expected output**

```text
    <loader readonly='yes' type='pflash'>/usr/share/AAVMF/AAVMF_CODE.fd</loader>
    <nvram template='/usr/share/AAVMF/AAVMF_VARS.fd'>/var/lib/libvirt/qemu/nvram/bcm-head_VARS.fd</nvram>
    <boot dev='cdrom'/>
    <boot dev='hd'/>
```

No `secure-boot` line and no `.ms.fd` files. The two `<boot>` lines only appear if the VM was created with `--boot cdrom,hd` as above.

Both NICs carry the planned MACs:

```bash
sudo virsh domiflist bcm-head
```

**Expected output**

```text
 Interface   Type      Source         Model    MAC
------------------------------------------------------------------
 vnet0       network   default        virtio   52:54:00:bc:00:01
 vnet1       network   bcm-internal   virtio   52:54:00:bc:01:01
```

vCPUs are pinned to the BCM cores, and the console port is known:

```bash
sudo virsh vcpupin bcm-head
sudo virsh vncdisplay bcm-head
```

**Expected output**

```text
 VCPU   CPU Affinity
----------------------
 0      4,10-14
 1      4,10-14
 2      4,10-14
 3      4,10-14

127.0.0.1:0
```

On screen, the VM should show the BCM boot menu, with **Start Base Command Manager Graphical Installer** first.

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**VNC viewer: “Connection refused” on `localhost:5901`**

Nothing is listening on your machine: the SSH tunnel has closed (sleep, Wi-Fi blip, or the window was closed). Start the tunnel again; the VM is unaffected. If the tunnel is up, check the VM is running and the display number matches the port in the tunnel.

```bash
sudo virsh list --all
sudo virsh vncdisplay bcm-head
```

**You type but nothing appears on the VM's screen**

The viewer doesn't have keyboard focus. Click inside it, or close and reopen the viewer; the tunnel can stay up.

**Black screen**

Press a key first — the VM may be idle at a menu. If it stays black, switch to the `ramfb` display, which works from power-on on Arm VMs:

```bash
sudo virsh destroy bcm-head
sudo virt-xml bcm-head --edit --video model.type=ramfb
sudo virsh start bcm-head
```

**Firmware Boot Manager lists no CD-ROM, only HARDDISK, EFI Internal Shell and PXE/HTTP entries**

No ISO is attached. Check, then insert or add one:

```bash
sudo virsh domblklist bcm-head --details

# cdrom line with '-' as source: insert the ISO
sudo virsh change-media bcm-head sdb /var/lib/libvirt/images/iso/bcm11-arm64.iso --insert --config

# no cdrom line at all: add a drive
sudo virt-xml bcm-head --add-device \
  --disk /var/lib/libvirt/images/iso/bcm11-arm64.iso,device=cdrom,bus=scsi,readonly=on
```

**Selecting the CD-ROM does nothing and returns to the menu**

Either the ISO is the x86 edition or Secure Boot is on. Check which file is attached, then check the firmware:

```bash
sudo virsh domblklist bcm-head --details          # is it the file you verified in step 5?
sudo virsh dumpxml bcm-head | grep -iE "loader|nvram|secure"
```

Wrong file: swap it with `sudo virsh change-media bcm-head sdb <arm64.iso> --update --config`. Secure Boot on (`secure-boot` enabled, or `.ms.fd` files): see the next item.

**`dumpxml` shows `name='secure-boot'` enabled or `AAVMF_CODE.ms.fd`**

The VM was created with plain `--boot uefi`. Switch the existing VM to the plain firmware and put the CD-ROM first. Removing the `firmware='efi'` attribute and `<firmware>` block matters: without that, `virsh define` fails with “Unable to find 'efi' firmware that is compatible with the current configuration”.

```bash
sudo virsh destroy bcm-head
sudo virsh dumpxml --inactive bcm-head > ~/bcm-head.xml
sed -i \
  -e "s/ firmware='efi'//" \
  -e "/<firmware>/,/<\/firmware>/d" \
  -e "s#AAVMF_CODE.ms.fd#AAVMF_CODE.fd#" \
  -e "s#AAVMF_VARS.ms.fd#AAVMF_VARS.fd#" \
  -e "s#<boot dev='hd'/>#<boot dev='cdrom'/>\n    <boot dev='hd'/>#" \
  ~/bcm-head.xml
grep -A6 "<os" ~/bcm-head.xml        # plain AAVMF files, no <firmware> block
sudo rm -f /var/lib/libvirt/qemu/nvram/bcm-head_VARS.fd
sudo virsh define ~/bcm-head.xml
sudo virsh start bcm-head
```

**VM lands at the UEFI `Shell>` prompt**

The firmware didn't boot the CD automatically. `FS0` on a `CDROM` path is the ISO's boot image. Start the boot loader by hand:

```text
fs0:
ls \EFI\BOOT
\EFI\BOOT\bootaa64.efi
```

If `ls` shows only `bootx64.efi` and running it gives “Command Error Status: Unsupported”, the attached ISO is the x86 edition — swap it as above.

**`virt-install` fails: “domain 'bcm-head' already exists”**

Change the existing VM (as above) rather than creating it again. To start over completely, remove it first; this also deletes an attached ISO unless you eject it.

```bash
sudo virsh change-media bcm-head sdb --eject --config 2>/dev/null
sudo virsh destroy bcm-head 2>/dev/null
sudo virsh undefine bcm-head --nvram --remove-all-storage
```

**Leaving the serial console**

<kbd>Control</kbd>+<kbd>]</kbd>; on some non-US layouts <kbd>Control</kbd>+<kbd>5</kbd>. Or close the tab — the VM keeps running.

</details>

<a id="s7"></a>

## 🟩 7. Run the BCM installer

> *Touches: inside bcm-head only.*

At the ISO boot menu choose **Start Base Command Manager Graphical Installer**. If the VM lands at a `Shell>` prompt instead, start the boot loader by hand with `fs0:` and then `\EFI\BOOT\bootaa64.efi`. Screen order varies slightly between releases; these are the values that matter, as seen in BCM 11.34.0 on Ubuntu 24.04.

| Installer screen | Value for this lab |
|---|---|
| License / EULA | Accept. The product key is applied after installation. |
| Kernel modules, hardware detection | Defaults. Confirm two virtio NICs and one ~120 GB disk are listed. |
| Installation source | The DVD/ISO. |
| Cluster settings | Cluster name `sparklab`; your time zone; time server `pool.ntp.org`; nameserver `192.168.122.1`. |
| Workload manager | None for now. You add Slurm once nodes are up (step 9). |
| Network topology | **Type 1**: nodes on a private internal network, head node routes to the outside. |
| Head node | Hostname `bcm-head`; set the root password; hardware vendor Other/Generic. |
| Compute nodes | 2 nodes, base name `node`, 3 digits (node001, node002); vendor Other/Generic. |
| BMC configuration | No BMC / IPMI. VMs don't have one; see step 10 for power control. |
| Networks: externalnet | DHCP **off**; base `192.168.122.0`, netmask `/24`, gateway `192.168.122.1`, MTU 1500. Change **Domain name** from the prefilled `nvidia.com` to a private name such as `sparklab.home.arpa`: BCM answers DNS for this domain itself, so a real domain there can break lookups of NVIDIA's servers during licensing and updates. Avoid `.local` (used by macOS Bonjour). |
| Networks: internalnet | Keep the defaults: base `10.141.0.0`, netmask `/16`, dynamic range `10.141.160.0`–`10.141.167.255` (temporary addresses for nodes while they PXE-boot, before they are identified), domain `eth.cluster`, gateway **blank** (the head node becomes the nodes' gateway), MTU 1500. |
| Head node network interfaces | **Check this screen — the installer's guess can be swapped.** It assigns networks by name order, not by what each NIC is plugged into. On this VM the correct mapping is:<br>`enp1s0` → **externalnet** → `192.168.122.10` (or your LAN address with br0)<br>`enp2s0` → **internalnet** → `10.141.255.254`<br>Confirm with the PCI bus check below; change the Network dropdowns and IPs if the screen shows it the other way round. |
| Disk layout | The installer warns that the disk is smaller than the recommended 500 GB and defaults to **One big partition** for both the head node and compute nodes. Keep it: that's fine for a lab. Use the edit (pencil) button on the compute nodes layout to confirm it targets `/dev/sda`. |
| Additional software | Leave **CUDA unchecked**. Nothing in this cluster has a GPU, and CUDA would add several GB to the head node and every software image. It can be added later with `apt` if you want to practise with it. |

### Check which interface is which

Before accepting the head node network interfaces screen, look up each NIC's PCI bus on the Spark. Linux names a NIC on bus `0x01` `enp1s0`, on bus `0x02` `enp2s0`, and so on.

```bash
sudo virsh dumpxml bcm-head | grep -A4 "<interface" | grep -E "mac address|source network|bus="
```

Expected output on this VM:

```xml
<mac address='52:54:00:bc:00:01'/>
<source network='default' … bridge='virbr0'/>
<address type='pci' … bus='0x01' …/>        ← enp1s0 = externalnet
<mac address='52:54:00:bc:01:01'/>
<source network='bcm-internal' … bridge='virbr-bcmi'/>
<address type='pci' … bus='0x02' …/>        ← enp2s0 = internalnet
```

A swapped mapping puts BCM's DHCP and PXE service on the NAT network: the compute nodes never get addresses, and the head node has no internet access for licensing.

Review the summary, start the install, and let it reboot. Installation on the efficiency cores takes roughly 20-40 minutes.

### Detach the ISO after the first boot

```bash
sudo virsh domblklist bcm-head --details   # find the CD-ROM target, e.g. sdb
sudo virsh change-media bcm-head sdb --eject --config
```

With the ISO ejected, the CD-first boot order falls through to the disk, so the head node boots its installed system from now on.

### Verify this step

Before accepting the network screens: the PCI bus check above shows `bus='0x01'` on `default`, and the installer shows `enp1s0` → externalnet → `192.168.122.10`, `enp2s0` → internalnet → `10.141.255.254`.

After the install has rebooted and you've ejected the ISO, the CD-ROM is empty:

```bash
sudo virsh domblklist bcm-head --details
```

**Expected output**

```text
 Type   Device   Target   Source
-------------------------------------------------------------------
 file   disk     sda      /var/lib/libvirt/images/bcm-head.qcow2
 file   cdrom    sdb      -
```

The console shows an Ubuntu login prompt for `bcm-head` instead of the installer.

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**The installer lists only one NIC, or the disk is missing**

```bash
sudo virsh domiflist bcm-head     # two interfaces: default and bcm-internal
sudo virsh domblklist bcm-head --details
```

**The head node network interfaces screen shows enp1s0 = internalnet**

That's the installer's guess, and on this VM it's backwards. Run the PCI bus check above; `bus='0x01'` on the `default` network means `enp1s0` is external. Swap the Network dropdowns and set `enp1s0` to `192.168.122.10`, `enp2s0` to `10.141.255.254`.

**You already installed with the mapping swapped**

Nodes won't get DHCP and `request-license` can't reach the internet. Reinstalling is quickest in a lab. To fix it in place instead, swap the two interfaces' networks and IPs on the head node in cmsh, then reboot it:

```bash
cmsh
device use master
interfaces
list                       # shows enp1s0 / enp2s0 with their networks and IPs
```

Use `use enp1s0`, `set network externalnet`, `set ip 192.168.122.10`, then the same for `enp2s0` with internalnet and 10.141.255.254, and `commit`. Work from the VNC or serial console, since SSH over the wrong interface will drop.

**externalnet's domain was left as nvidia.com**

Lookups of NVIDIA hosts from the cluster may be answered locally and fail. Change it after installation:

```bash
cmsh -c "network use externalnet; set domainname sparklab.home.arpa; commit"
```

**After the install reboots, the installer starts again**

The ISO is still first in the boot order. Eject it (above) and reset the VM with `sudo virsh reset bcm-head`.

**Installation stalls or fails partway**

Watch the serial console (`sudo virsh console bcm-head --escape '^Q'`) for errors, and check the Spark isn't short on memory or disk: `free -g`, `df -h /var/lib`. A truncated ISO often fails here too — re-check its checksum.

</details>

<a id="s8"></a>

## 🟩 8. Post-install and licensing

> *Touches: inside bcm-head.*

### Log in

```bash
# From the Spark
ssh root@192.168.122.10

# From your laptop, jumping through the Spark
ssh -J <spark-user>@<spark-ip> root@192.168.122.10
```

### Activate the licence

```bash
request-license
# Enter the product key; accept the defaults for the certificate details.
# The head node must reach the internet through the NAT network for this.

cmsh -c "main licenseinfo"
```

### Update and sanity-check

```bash
apt update && apt -y upgrade

cmsh -c "network list"      # externalnet and internalnet
cmsh -c "device list"       # bcm-head UP; node001/node002 DOWN (not created yet)
cmsh -c "category list"     # default category, default software image
cmsh -c "softwareimage list"

# DHCP should serve the internal network only
ss -ulpn | grep -E ':67|:69'
```

### Verify the head node's network interfaces

Confirm the installer applied the mapping from step 7: `enp1s0` on externalnet, `enp2s0` on internalnet. Run these on bcm-head (over SSH, or on the serial console if SSH doesn't answer):

```bash
ip -br addr show enp1s0; ip -br addr show enp2s0
```

**Expected output**

```text
enp1s0   UP   192.168.122.10/24 fe80::5054:ff:febc:1/64
enp2s0   UP   10.141.255.254/16 fe80::5054:ff:febc:101/64
```

The link-local IPv6 addresses are usually derived from the MACs (`…:bc:00:01` and `…:bc:01:01`), which is a quick second confirmation that each name is on the right NIC.

```bash
ip -br link | grep 52:54:00:bc
ip route | grep default
cmsh -c "device use master; interfaces; list"
```

**Expected output** (cmsh columns vary slightly by release)

```text
enp1s0   UP   52:54:00:bc:00:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
enp2s0   UP   52:54:00:bc:01:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
default via 192.168.122.1 dev enp1s0 proto static

Type     Network device name   IP               Network
-------- --------------------- ---------------- ------------
physical enp1s0                192.168.122.10   externalnet
physical enp2s0 [prov]         10.141.255.254   internalnet
```

From the Spark, the two VM-side NICs also appear as `vnet` devices. Match them to their networks with:

```bash
sudo virsh domiflist bcm-head
```

**Expected output**

```text
 Interface   Type      Source         Model    MAC
------------------------------------------------------------------
 vnet0       network   default        virtio   52:54:00:bc:00:01
 vnet1       network   bcm-internal   virtio   52:54:00:bc:01:01
```

If the default route points at `enp2s0`, or `enp1s0` holds 10.141.255.254, the mapping is swapped: see *Diagnose this step* in step 7.

### Reach Base View (the web GUI)

```bash
# On your laptop: forward a local port through the Spark
ssh -L 8081:192.168.122.10:8081 <spark-user>@<spark-ip>
# Then browse to https://localhost:8081/base-view and log in as root
```

### Verify this step

The interface checks above, plus:

The licence is active:

```bash
cmsh -c "main licenseinfo"
```

**Expected output**

```text
License version        7.0
Licensee               /C=US/ST=.../O=.../CN=sparklab
Start time             ...
End time               ...
Licensed nodes         ...
```

Exact fields vary; what matters is that it returns details rather than an error.

BCM knows both networks and the head node is up:

```bash
cmsh -c "network list"
cmsh -c "device list"
```

**Expected output**

```text
Name (key)        Type       Netmask bits   Base address     Domain name
----------------- ---------- -------------- ---------------- --------------------
externalnet       External   24             192.168.122.0    sparklab.home.arpa
internalnet       Internal   16             10.141.0.0       eth.cluster

Type         Hostname (key)  MAC                 Category   IP               Network       Status
------------ --------------- ------------------- ---------- ---------------- ------------- --------
HeadNode     bcm-head        52:54:00:BC:01:01                  10.141.255.254   internalnet   [   UP   ]
PhysicalNode node001         00:00:00:00:00:00   default    10.141.0.1       internalnet   [  DOWN  ]
PhysicalNode node002         00:00:00:00:00:00   default    10.141.0.2       internalnet   [  DOWN  ]
```

Nodes show DOWN with an empty MAC until step 9.

BCM's DHCP and TFTP listen only on the internal network:

```bash
ss -ulpn | grep -E ':67 |:69 '
```

**Expected output**

```text
UNCONN 0 0   0.0.0.0%enp2s0:67   0.0.0.0:*  users:(("dhcpd",...))
UNCONN 0 0          0.0.0.0:69   0.0.0.0:*  users:(("in.tftpd",...))
```

The DHCP line should name `enp2s0`. If it names `enp1s0`, the interfaces are swapped.

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`request-license` can't connect**

```bash
ping -c2 192.168.122.1        # from bcm-head: gateway
ping -c2 8.8.8.8
getent hosts www.nvidia.com
cat /etc/resolv.conf           # nameserver 192.168.122.1
```

**Base View doesn't load**

Work outward from the head node:

```bash
# on bcm-head
ss -tlnp | grep 8081
# on the Spark
curl -k https://192.168.122.10:8081/base-view | head -5
```

If the Spark gets HTML, the problem is the tunnel or forward; if it doesn't, check BCM's firewall (`/etc/shorewall/rules`) allows 8081. Use `https://`, and accept the self-signed certificate.

**SSH warns “REMOTE HOST IDENTIFICATION HAS CHANGED” after a reinstall**

```bash
ssh-keygen -R 192.168.122.10
ssh-keygen -R bcm-head
```

</details>

<a id="s9"></a>

## 🟩 9. Provision compute node VMs

> *Touches: new VMs in machine.slice; BCM device records.*

### Tell BCM each node's MAC address

Pre-registering the MAC means each VM is identified as the right node the first time it PXE-boots, with no manual identification prompt.

```bash
# On bcm-head
cmsh
device
use node001
set mac 52:54:00:bc:01:11
commit
use node002
set mac 52:54:00:bc:01:12
commit
quit
```

### Create the VMs on the Spark

Nodes attach only to `bcm-internal` and boot from the network first. The disk is second in the boot order, so a reinstall is always one reboot away. They use the same non-Secure-Boot firmware as the head node; with Secure Boot on, the firmware would refuse BCM's network boot loader in exactly the same silent way.

Unlike the head node, a node has no ISO to install from: BCM installs it over the network. `virt-install` still insists on an install method, so each node's empty disk is created first with `qemu-img` and the VM is created with `--import` (“boot what's there”). The `network,hd` order in `--boot` then makes it PXE-boot from the head node.

```bash
for i in 1 2; do
  n=$(printf "%03d" $i)
  sudo qemu-img create -f qcow2 /var/lib/libvirt/images/bcm-node$n.qcow2 60G
  sudo virt-install \
    --name bcm-node$n \
    --osinfo ubuntu24.04 \
    --arch aarch64 --machine virt \
    --import \
    --boot network,hd,loader=/usr/share/AAVMF/AAVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/AAVMF/AAVMF_VARS.fd \
    --cpu host-passthrough \
    --vcpus 2,cpuset=4,10-14 \
    --memory 4096 \
    --memorybacking locked=on \
    --controller type=scsi,model=virtio-scsi \
    --disk path=/var/lib/libvirt/images/bcm-node$n.qcow2,format=qcow2,bus=scsi,cache=none,discard=unmap \
    --network network=bcm-internal,model=virtio,mac=52:54:00:bc:01:1$i \
    --graphics vnc,listen=127.0.0.1 --video virtio \
    --console pty,target_type=serial \
    --noautoconsole
done
```

Three details matter here:

- **`qemu-img create` + `--import`**: the install method. Without it, `virt-install` stops with “An install method must be specified”.
- **`--boot network,hd,…`**: network first, disk second, set on the VM rather than per device. Don't add `boot.order=` to `--disk` or `--network` as well; libvirt refuses to combine the two styles.
- **No `size=` on `--disk`**: the disk already exists.

Check that each node got the right firmware, boot order and MAC:

```bash
sudo virsh dumpxml bcm-node001 | grep -iE "loader|<boot|mac address"
```

**Expected output**

```text
    <loader readonly='yes' type='pflash'>/usr/share/AAVMF/AAVMF_CODE.fd</loader>
    <boot dev='network'/>
    <boot dev='hd'/>
      <mac address='52:54:00:bc:01:11'/>
```

### Watch them provision

```bash
# On bcm-head
tail -f /var/log/node-installer
cmsh -c "device status"
# Expect: installing → installer_callinginit → UP. First install takes 10-20 min.

ssh node001 hostname
```

If a node sits at the UEFI screen without getting an address, check [troubleshooting](#trouble).

### Add a workload manager

```bash
cm-wlm-setup
# Choose Slurm; server role on bcm-head; client role on the "default" category.

sinfo
srun -N2 hostname
```

### Verify this step

Both nodes are provisioned and up:

```bash
cmsh -c "device status"
```

**Expected output**

```text
bcm-head ................. [   UP   ]
node001 .................. [   UP   ]
node002 .................. [   UP   ]
```

They answer by name and run jobs:

```bash
ssh node001 hostname
sinfo
srun -N2 hostname
```

**Expected output**

```text
node001
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
defq*        up   infinite      2   idle node[001-002]
node001
node002
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`virt-install`: “An install method must be specified (--location URL, --cdrom CD/ISO, --pxe, --import, --boot hd|cdrom|...)”**

The command has no install method. Nodes have no ISO, and per-device `boot.order=` settings, or a `--boot` with only firmware options, don't count. Use the command above: create the disk with `qemu-img`, then `--import` with `--boot network,hd,…`. Nothing is defined when this error appears, so there's nothing to clean up; if a disk file was already created, `qemu-img` reports it exists and `virt-install` reuses it.

**A node boots into the UEFI shell or “No bootable device” instead of PXE**

The boot order puts the disk first, or the NIC isn't on `bcm-internal`. Check `sudo virsh dumpxml bcm-node001 | grep -E "<boot|source network"`: you want `network` before `hd`, and `bcm-internal`.

**A node's screen shows PXE attempts but no DHCP offer**

```bash
# on bcm-head, while the node boots
journalctl -u dhcpd -f
cmsh -c "device use node001; get mac"
```

The MAC must match the node's NIC exactly (`sudo virsh domiflist bcm-node001`), and the node's NIC must be on `bcm-internal`.

**The node gets an address but never loads the installer**

Check its firmware: if it was created with plain `--boot uefi`, Secure Boot blocks BCM's network boot loader just as it blocked the ISO.

```bash
sudo virsh dumpxml bcm-node001 | grep -iE "loader|secure"
```

Apply the step 6 Secure Boot fix to the node (skip the CD-ROM boot-order line; nodes boot from the network).

**The node-installer fails on the disk**

The disk isn't at `/dev/sda`. Keep node disks on the SCSI bus, or change the category's disk setup to use `/dev/vda`.

**`sinfo` shows nodes `down` or `drain`**

```bash
scontrol show node node001 | grep -i reason
scontrol update nodename=node001 state=resume
```

</details>

<a id="s10"></a>

## 🟩 10. Power control from BCM (optional)

> *Touches: a host service account and a script on bcm-head.*

Real clusters power nodes through IPMI or Redfish. VMs have neither, so a custom power script lets BCM call `virsh` on the Spark instead. Without this, power nodes on and off from the host yourself; everything else works the same.

### On the Spark: a dedicated account

```bash
sudo useradd -m -s /bin/bash -G libvirt bcmpower
sudo -u bcmpower mkdir -p ~bcmpower/.ssh
# Paste bcm-head's root public key (next block) into:
sudo -u bcmpower nano ~bcmpower/.ssh/authorized_keys
```

> [!WARNING]
> **Scope this account tightly.**
>
> Membership in the libvirt group can control every VM on the host. Use a key that exists only on bcm-head, and consider adding `from="192.168.122.10"` in front of the key in authorized_keys.

### On bcm-head: key and script

```bash
ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519   # skip if it exists
cat /root/.ssh/id_ed25519.pub                          # → bcmpower's authorized_keys
ssh bcmpower@192.168.122.1 virsh -c qemu:///system list --all   # test

mkdir -p /cm/local/apps/cmd/scripts/powerscripts
cat > /cm/local/apps/cmd/scripts/powerscripts/virsh-power <<'EOF'
#!/bin/bash
# CMDaemon passes the action as $1 and the node name in CMD_HOSTNAME.
dom="bcm-${CMD_HOSTNAME}"
v="ssh -o BatchMode=yes bcmpower@192.168.122.1 virsh -c qemu:///system"
case "${1,,}" in
  on)     $v start "$dom" ;;
  off)    $v destroy "$dom" ;;
  reset)  $v reset "$dom" ;;
  status) state=$($v domstate "$dom")
          [ "$state" = "running" ] && echo ON || echo OFF ;;
esac
exit 0
EOF
chmod 755 /cm/local/apps/cmd/scripts/powerscripts/virsh-power
```

### Attach it to the nodes

```bash
cmsh
category use default
set powercontrol custom
set custompowerscript /cm/local/apps/cmd/scripts/powerscripts/virsh-power
commit
quit

cmsh -c "device power status -n node001..node002"
cmsh -c "device power reset -n node002"
```

Check the argument and status conventions against the "custom power management" section of the BCM Administrator Manual for your release; if they differ, only the `case` block needs to change.

### Verify this step

BCM reads each node's power state through the script:

```bash
cmsh -c "device power status -n node001..node002"
```

**Expected output**

```text
custom      node001 ............. [   ON   ]
custom      node002 ............. [   ON   ]
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**`device power status` shows UNKNOWN or fails**

Run the same SSH call the script makes, by hand, from bcm-head:

```bash
ssh -o BatchMode=yes bcmpower@192.168.122.1 virsh -c qemu:///system list --all
```

A password prompt or “Permission denied” means the key isn't in `~bcmpower/.ssh/authorized_keys` or that file's permissions are too open (`chmod 700 ~/.ssh; chmod 600 authorized_keys`).

**The script runs but acts on the wrong VM**

The script builds the VM name as `bcm-` plus the BCM node name, so `node001` must map to a VM called `bcm-node001`. Check with `sudo virsh list --all`.

</details>

<a id="s11"></a>

## 🟧 11. Run LLMs in the workload partition

> *Touches: Docker on the host. The GPU stays here and never enters a VM.*

DGX OS ships Docker with the NVIDIA Container Toolkit. Two habits keep workloads on their side of the fence: pin containers to the performance cores, and cap GPU memory in the framework itself.

### Pin to the performance cores

```bash
# Ollama
docker run -d --name ollama --gpus all --cpuset-cpus 5-9,15-19 \
  -v ollama:/root/.ollama -p 11434:11434 --restart unless-stopped \
  ollama/ollama

# vLLM (use the arm64 image NVIDIA publishes for DGX Spark)
docker run -d --name vllm --gpus all --cpuset-cpus 5-9,15-19 --ipc=host \
  -p 8000:8000 -v ~/.cache/huggingface:/root/.cache/huggingface \
  <vllm-image-for-spark> \
  vllm serve <model> --gpu-memory-utilization 0.60 --max-model-len 32768
```

### Cap memory where the model runs

On unified memory, a model's weights and KV cache come from the same pool the VMs use. Docker's `--memory` flag is not a reliable limit for GPU allocations here, so set the limit inside the framework:

- **vLLM**: `--gpu-memory-utilization` is a fraction of total memory. With about 34 GB held by the BCM lab and 8 GB by the host, stay at 0.60-0.65.
- **Ollama and llama.cpp**: bound the context length and the number of models kept loaded (`OLLAMA_MAX_LOADED_MODELS=1`).
- **NIM containers**: follow the model card's memory profile and pick one that fits in ~80 GB.

### Watch both sides at once

```bash
nvidia-smi                          # GPU processes and memory
free -g                             # unified pool as the OS sees it
systemd-cgtop -d 2                  # machine.slice vs system.slice in real time
docker stats --no-stream
```

> [!TIP]
> **If you need the whole machine.**
>
> Stop the lab (`virsh shutdown` the nodes, then the head), raise the framework cap, and run the job. Start the lab again when you're done; BCM resumes where it left off.

### Verify this step

Containers see the GPU and run on the performance cores:

```bash
docker exec ollama nvidia-smi --query-gpu=name --format=csv,noheader
docker inspect --format '{{.HostConfig.CpusetCpus}}' ollama
```

**Expected output**

```text
NVIDIA GB10
5-9,15-19
```

Memory is shared the way you planned:

```bash
free -g
systemd-cgtop -1 -n1 --order=memory | head -6
```

**Expected output**

```text
Mem:   total ~119, used includes ~34 GB for the lab VMs plus the loaded model
machine.slice appears with close to its locked VM memory
```

<details>
<summary><b>🩺 Diagnose this step</b></summary>

**A container can't see the GPU**

```bash
docker run --rm --gpus all nvcr.io/nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
nvidia-ctk --version
```

**Check a container really is on the performance cores**

```bash
docker inspect --format '{{.HostConfig.CpusetCpus}}' ollama   # 5-9,15-19
```

**A model load gets killed, or a VM dies**

```bash
journalctl -k | grep -iE 'oom|killed process'
free -g
```

Lower the framework's memory fraction or context length, or shut the lab down while that model runs.

</details>

<a id="ops"></a>

## ⬛ D. Day-2 operations

> *Keeping the lab tidy once it works.*

### Start and stop in order

```bash
# Start: head first, nodes after DHCP/TFTP are up
virsh -c qemu:///system start bcm-head
sleep 120
for n in bcm-node001 bcm-node002; do virsh -c qemu:///system start $n; done

# Stop: nodes first
for n in bcm-node001 bcm-node002; do virsh -c qemu:///system shutdown $n; done
sleep 60
virsh -c qemu:///system shutdown bcm-head
```

To bring only the head node up at boot, run `virsh autostart bcm-head` and leave the nodes manual. libvirt has no ordering between autostarted VMs.

### Back up

Internal snapshots are unreliable for UEFI Arm guests, so take cold copies of the disk, the UEFI variable store and the definition:

```bash
virsh -c qemu:///system shutdown bcm-head   # wait until "shut off"
B=/srv/backup/bcm-$(date +%F); sudo mkdir -p $B
sudo cp --sparse=always /var/lib/libvirt/images/bcm-head.qcow2 $B/
sudo cp /var/lib/libvirt/qemu/nvram/bcm-head_VARS.fd $B/
virsh -c qemu:///system dumpxml bcm-head | sudo tee $B/bcm-head.xml >/dev/null
```

Compute nodes don't need backing up; BCM reprovisions them from the software image.

### Things worth practising once it runs

- Clone the default software image, change a package, assign it to a new category and watch nodes pick it up with `imageupdate`.
- Add a third node VM and register it with the `foreach` and `clone` commands in cmsh.
- Build an x86_64 image with `cm-image` for mixed-architecture practice. It runs under QEMU emulation, so expect hours rather than minutes.
- Set up head node HA with a second head VM on the same two networks.

<a id="teardown"></a>

## ⬛ X. Tear down the lab

> *Touches: everything this guide created on the host. Your LLM containers, models and DGX OS configuration are not affected.*

Undo the build in reverse order: VMs first, then the networks, the isolation boundary, and finally the leftover accounts and files. Each step is independent, so you can stop after removing the VMs if you only want the memory and disk back.

> [!WARNING]
> **This is permanent.**
>
> Deleting the head node disk deletes the cluster configuration, software images and licence state. If you might rebuild, take the cold backup from [Day-2 operations](#ops) first.

### 1. Delete the VMs and their disks

```bash
# Eject the ISO first so --remove-all-storage doesn't delete it
sudo virsh change-media bcm-head sdb --eject --config 2>/dev/null

for d in bcm-node002 bcm-node001 bcm-head; do
  sudo virsh destroy "$d" 2>/dev/null      # force off if running
  sudo virsh undefine "$d" --nvram --remove-all-storage
done

sudo virsh list --all                      # no bcm-* entries left
ls /var/lib/libvirt/images/                # no bcm-*.qcow2 left
ls /var/lib/libvirt/qemu/nvram/ | grep bcm # no UEFI variable stores left
```

`--remove-all-storage` deletes every disk still attached to the VM, including an ISO that was never ejected, which is why the loop starts with an eject. Delete the ISO separately in step 6 below if you don't want it.

### 2. Remove the lab networks

```bash
sudo virsh net-destroy bcm-internal
sudo virsh net-undefine bcm-internal

# Only if nothing else on the Spark uses libvirt's NAT network:
sudo virsh net-destroy default
sudo virsh net-autostart default --disable

sudo virsh net-list --all
```

### 3. Remove the LAN bridge (only if you created br0)

Do this from the Spark's local console, because the wired connection drops briefly.

```bash
NIC=enP7s7   # your wired port
sudo nmcli con delete br0-port br0
sudo nmcli device connect $NIC
ip -br addr show $NIC                      # the port has its LAN address again
```

### 4. Remove the isolation boundary

`systemctl revert` deletes the drop-ins that `set-property` wrote, returning all three slices to their defaults so LLM workloads can use every core again.

```bash
sudo systemctl revert machine.slice system.slice user.slice
sudo systemctl daemon-reload

systemctl show machine.slice system.slice user.slice -p AllowedCPUs -p MemoryMax
# Expect AllowedCPUs empty and MemoryMax=infinity
```

If you had narrowed Docker containers to `--cpuset-cpus 5-9,15-19`, recreate them without that flag to let them use all 20 cores.

### 5. Remove the libvirt hook and power account

```bash
sudo rm -f /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd

# Only if you set up BCM power control in step 10
sudo userdel -r bcmpower
```

### 6. Delete the ISO and backups

```bash
sudo rm -f /var/lib/libvirt/images/iso/bcm11-arm64.iso
sudo rmdir /var/lib/libvirt/images/iso 2>/dev/null
sudo rm -rf /srv/backup/bcm-*              # only if you no longer want them
```

### 7. Remove the virtualization stack (optional)

Skip this if you run any other VMs on the Spark. Otherwise, it returns the host to its original package set.

```bash
sudo systemctl disable --now libvirtd
sudo apt purge -y qemu-system-arm qemu-efi-aarch64 libvirt-daemon-system \
  libvirt-clients virtinst virt-manager virt-viewer cpu-checker
sudo apt autoremove -y
sudo gpasswd -d $USER libvirt
sudo gpasswd -d $USER kvm
```

Don't purge `qemu-utils` or `ovmf` without checking `apt`'s removal list first; other DGX OS tooling may depend on them.

### 8. Confirm the Spark is back to normal

```bash
free -g                                    # ~119 GB available again
nvidia-smi                                 # GPU unaffected
ip -br link | grep -E 'virbr|br0'          # no lab bridges remain
df -h /var/lib                             # disk space recovered
```

> [!NOTE]
> **Licence reuse.**
>
> BCM licences are issued to a specific head node. If you plan to rebuild the lab later with the same product key, check with NVIDIA whether it needs to be reissued before you run `request-license` on the new head node.

<details>
<summary><b>🩺 Diagnose teardown</b></summary>

**`undefine` fails: “cannot undefine domain with nvram”**

Include `--nvram`, as in the loop above.

**`net-destroy default` fails because it's in use**

Another VM still uses it. Check `sudo virsh list --all`; keep `default` if you run other VMs.

**You deleted the ISO by accident**

Download it again on the Spark with `wget` (step 5) — the portal lets you re-download with the same product key.

</details>

<a id="trouble"></a>

## ⬛ T. Troubleshooting

> *The failures this setup actually produces.*

| Symptom | Likely cause and fix |
|---|---|
| VM starts but the screen is black or it never reaches the ISO menu | Firmware or display mismatch. Confirm the loader is the plain `AAVMF_CODE.fd` (`virsh dumpxml bcm-head \| grep -i loader`). Try `--video ramfb`, or use the text installer over `virsh console`. |
| Mouse cursor doesn't track in the console | Add a USB tablet input device (`--input tablet,bus=usb`, already in the commands above). |
| Node stays at UEFI with no DHCP offer | Check the node NIC is on `bcm-internal`, its MAC matches the value set in cmsh, and the head's internal NIC is mapped to internalnet. On the head: `journalctl -u dhcpd -f` while the node boots. |
| Node gets an address but the node-installer fails on disk | The disk isn't where the layout expects it. Confirm the node disk is on the SCSI bus (`/dev/sda`), or edit the category's disk setup to include `/dev/vda`. |
| `request-license` can't connect | No outbound path. From the head: `ping 192.168.122.1`, then `curl -I https://www.nvidia.com`. Check the nameserver is `192.168.122.1` and `virsh net-list` shows default active. |
| VM fails to start with "cannot lock memory" | Locked memory exceeds the slice cap or memlock limit. Raise `MemoryMax` on machine.slice or reduce the VM's RAM. |
| A VM was killed while a model loaded | Check `journalctl -k \| grep -i oom`. Confirm the hook ran (`cat /proc/$(pgrep -f 'guest=bcm-head')/oom_score_adj` shows -800) and lower the framework's memory fraction. |
| Spark lost its wired connection after bridging | From the local console: `nmcli con delete br0 br0-port` and bring the original connection back up. |
| Boot Manager shows no CD-ROM entry | No ISO attached. `sudo virsh domblklist bcm-head --details`, then insert or add the ISO (step 6 diagnostics). |
| Selecting the CD-ROM does nothing | Wrong-architecture ISO or Secure Boot firmware. Check the attached file and `virsh dumpxml bcm-head \| grep -iE "loader\|secure"`. |
| UEFI shell: “Command Error Status: Unsupported” | The boot file is `bootx64.efi`: the attached ISO is x86. Attach the aarch64 ISO with `virsh change-media … --update --config`. |
| `virsh define`: “Unable to find 'efi' firmware that is compatible” | Remove `firmware='efi'` and the `<firmware>` block when naming the plain AAVMF files explicitly. |
| VNC viewer: “Connection refused” on localhost:5901 | The SSH tunnel closed. Start it again; check `virsh vncdisplay bcm-head` matches the tunnel's port. |
| ISO mount: “already mounted” | A stale mount, possibly of an older file. `sudo umount /mnt` until “not mounted”, then remount. |
| `virt-host-validate`: WARN on cgroup 'devices' controller | Harmless on cgroup v2. Make sure `cpuset` and `memory` are in `/sys/fs/cgroup/cgroup.controllers`. |
| Installer maps enp1s0 to internalnet | The guess is swapped on this VM. Check PCI buses with `virsh dumpxml bcm-head`: bus `0x01` (`default` network) is `enp1s0` = externalnet (step 7). |
| `virt-install` for nodes: “An install method must be specified” | Create the node disk with `qemu-img create` and add `--import` with `--boot network,hd,…` (step 9). |
| Things broke after a DGX OS update | A new kernel may reset kvm or cgroup behaviour. Recheck `/dev/kvm`, `systemctl show machine.slice` and `virt-host-validate`. |

<a id="caveats"></a>

## ⬛ L. Limits of this design

> *What the lab can and can't teach you.*

- No GPU inside the cluster. You can practise provisioning, images, categories, Slurm, monitoring and HA, but not GPU scheduling, DCGM health checks or MIG.
- No real BMCs. Power control is simulated through virsh, and BMC network configuration can't be exercised.
- Arm-only by default. The VMs match the Spark's architecture; x86 nodes would need full emulation and would be very slow.
- Unsupported host configuration. Virtualization on DGX Spark works but isn't a supported NVIDIA use, and future DGX OS updates could change that.
- Shared memory pool. The cgroup cap and locked memory protect the VMs, but the GPU side still has to be configured to leave room; nothing enforces that from the hardware.
