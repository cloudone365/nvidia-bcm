# NVIDIA Base Command Manager lab on DGX Spark

Step-by-step guides for running an NVIDIA Base Command Manager (BCM) lab cluster as KVM virtual machines on a single DGX Spark, while keeping the GB10 GPU and most of the machine free for LLMs and other tools.

The BCM head node and PXE-provisioned compute nodes run in an isolated CPU and memory partition (a capped systemd slice). BCM manages only its own virtual cluster on a private provisioning network, and the GPU stays on the host for containers such as vLLM, Ollama and NIM.

## Guides

| File | Use it when |
| --- | --- |
| [`bcm-dgx-spark-lab-guide-wifi.html`](bcm-dgx-spark-lab-guide-wifi.html) | The Spark connects to your home LAN over Wi-Fi (`wlP9s9`, 192.168.0.0/24). The head node sits behind libvirt NAT and is reached from the LAN with an SSH jump or port forwards. |
| [`bcm-dgx-spark-lab-guide-wired.html`](bcm-dgx-spark-lab-guide-wired.html) | The Spark connects over its wired 10 GbE port. Adds an optional bridge so the head node can get its own LAN address. |

Both guides cover the architecture and resource budget, KVM/libvirt setup, the isolation boundary, BCM head node installation and licensing, compute node provisioning, optional BCM power control through `virsh`, running LLMs alongside the lab, day-2 operations, full teardown and troubleshooting.

Every step ends with a **Diagnose this step** panel built from a real install. The issues that most often stop a first install are covered there:

- **Secure Boot firmware.** `virt-install --boot uefi` picks firmware that silently refuses the BCM boot loader. The guides name the plain AAVMF firmware explicitly and show how to fix an existing VM.
- **Wrong-architecture ISO.** The x86 and aarch64 ISOs have near-identical names. Step 5 checks the embedded `efi.img` for `bootaa64.efi` before any VM is created.
- **Console access without virt-manager.** Step 6 reaches the VM's screen from a Mac or Windows PC through an SSH tunnel and any VNC viewer.
- **CPU layout.** On the GB10 the efficiency cores are 0–4 and 10–14 and the performance cores are 5–9 and 15–19, not two contiguous blocks.

## Reading the guides

Read them as web pages on GitHub Pages:

- **Wi-Fi guide:** https://cloudone365.github.io/nvidia-bcm/bcm-dgx-spark-lab-guide-wifi.html
- **Wired guide:** https://cloudone365.github.io/nvidia-bcm/bcm-dgx-spark-lab-guide-wired.html

Opening the `.html` files in the repository file list shows their source code, because GitHub doesn't render HTML there. You can also download a file and open it in any browser.

## Scope

This is a learning lab, not a supported configuration. Virtualization on DGX Spark isn't a supported NVIDIA use case and GPU passthrough to VMs doesn't work, so the BCM cluster is CPU-only. A BCM product key is needed to download the ISO and license the head node.
