# NVIDIA Base Command Manager lab on DGX Spark

Step-by-step guides for running an NVIDIA Base Command Manager (BCM) lab cluster as KVM virtual machines on a single DGX Spark, while keeping the GB10 GPU and most of the machine free for LLMs and other tools.

The BCM head node and PXE-provisioned compute nodes run in an isolated CPU and memory partition (a capped systemd slice). BCM manages only its own virtual cluster on a private provisioning network, and the GPU stays on the host for containers such as vLLM, Ollama and NIM.

## Guides

| Guide | Use it when |
| --- | --- |
| [Wi-Fi guide](bcm-dgx-spark-lab-guide-wifi.md) | The Spark connects to your home LAN over Wi-Fi (`wlP9s9`, 192.168.0.0/24). The head node sits behind libvirt NAT and is reached from the LAN with an SSH jump or port forwards. |
| [Wired guide](bcm-dgx-spark-lab-guide-wired.md) | The Spark connects over its wired 10 GbE port. Adds an optional bridge so the head node can get its own LAN address. |

![Lab architecture](assets/architecture-wifi.svg)

Both guides cover the architecture and resource budget, KVM/libvirt setup, the isolation boundary, BCM head node installation and licensing, compute node provisioning, optional BCM power control through `virsh`, running LLMs alongside the lab, day-2 operations, full teardown and troubleshooting.

Every step ends with a **Diagnose this step** panel built from a real install. The issues that most often stop a first install are covered there:

- **Secure Boot firmware.** `virt-install --boot uefi` picks firmware that silently refuses the BCM boot loader. The guides name the plain AAVMF firmware explicitly and show how to fix an existing VM.
- **Wrong-architecture ISO.** The x86 and aarch64 ISOs have near-identical names. Step 5 checks the embedded `efi.img` for `bootaa64.efi` before any VM is created.
- **Console access without virt-manager.** Step 6 reaches the VM's screen from a Mac or Windows PC through an SSH tunnel and any VNC viewer.
- **Verification.** Every step ends with **Verify this step**: the commands to run and the output to expect, including a full ISO check (size, checksum, readability, architecture, and which file the VM actually uses).
- **Head node interfaces.** The installer can map `enp1s0`/`enp2s0` the wrong way round; step 7 shows how to confirm with the PCI bus, and step 8 verifies the result inside the head node.
- **CPU layout.** On the GB10 the efficiency cores are 0–4 and 10–14 and the performance cores are 5–9 and 15–19, not two contiguous blocks.

## Reading the guides

The `.md` guides render directly on GitHub, with the architecture diagram, colour-coded partitions (⬛ host, 🟩 BCM, 🟧 workloads), coloured callouts and a collapsible **🩺 Diagnose this step** panel under every step.

The same guides also exist as styled HTML pages (`.html`), with copy buttons on every command. GitHub shows those files as source code; to read them as web pages, enable GitHub Pages for this repository (Settings → Pages → Deploy from branch `main`, folder `/`). They will then be at:

- https://cloudone365.github.io/nvidia-bcm/bcm-dgx-spark-lab-guide-wifi.html
- https://cloudone365.github.io/nvidia-bcm/bcm-dgx-spark-lab-guide-wired.html

## Scope

This is a learning lab, not a supported configuration. Virtualization on DGX Spark isn't a supported NVIDIA use case and GPU passthrough to VMs doesn't work, so the BCM cluster is CPU-only. A BCM product key is needed to download the ISO and license the head node.
