# NVIDIA Base Command Manager lab on DGX Spark

Step-by-step guides for running an NVIDIA Base Command Manager (BCM) lab cluster as KVM virtual machines on a single DGX Spark, while keeping the GB10 GPU and most of the machine free for LLMs and other tools.

The BCM head node and PXE-provisioned compute nodes run in an isolated CPU and memory partition (a capped systemd slice). BCM manages only its own virtual cluster on a private provisioning network, and the GPU stays on the host for containers such as vLLM, Ollama and NIM.

## Guides

| File | Use it when |
| --- | --- |
| [`bcm-dgx-spark-lab-guide-wifi.html`](bcm-dgx-spark-lab-guide-wifi.html) | The Spark connects to your home LAN over Wi-Fi (`wlP9s9`, 192.168.0.0/24). The head node sits behind libvirt NAT and is reached from the LAN with an SSH jump or port forwards. |
| [`bcm-dgx-spark-lab-guide-wired.html`](bcm-dgx-spark-lab-guide-wired.html) | The Spark connects over its wired 10 GbE port. Adds an optional bridge so the head node can get its own LAN address. |

Both guides cover the architecture and resource budget, KVM/libvirt setup, the isolation boundary, BCM head node installation and licensing, compute node provisioning, optional BCM power control through `virsh`, running LLMs alongside the lab, day-2 operations, full teardown and troubleshooting.

## Reading the guides

GitHub shows `.html` files as source. Download a file and open it in a browser, or enable GitHub Pages for this repository.

## Scope

This is a learning lab, not a supported configuration. Virtualization on DGX Spark isn't a supported NVIDIA use case and GPU passthrough to VMs doesn't work, so the BCM cluster is CPU-only. A BCM product key is needed to download the ISO and license the head node.
