# proxmox-nvidia-passthrough

Passing an NVIDIA RTX 50-series GPU through Proxmox VE 9 to an Ubuntu 24.04 VM for local AI, with Ollama and Open WebUI on top. These are the steps I followed on my home lab, including the problems I hit along the way.

Tested in November 2026 with:

| Piece | Version |
| --- | --- |
| Host | Proxmox VE 9.2, kernel 7.0.14, AMD Ryzen 7 8700G (its Radeon 780M stays with the host) |
| GPU | NVIDIA GeForce RTX 5070 Ti, 16 GB |
| VM | Ubuntu Server 24.04.5, kernel 6.8 |
| Driver | `nvidia-driver-595-open` 595.99.02 (CUDA 13.2) |
| Software | Ollama 0.40.15, Open WebUI (`main`), Docker 29.1 from Ubuntu's repo |

Results: `qwen3:14b` runs fully on the GPU at about 80 tokens/s, and `llama3.2:3b` at 150–190 tokens/s.

## Disclaimer

> [!WARNING]
> **Use at your own risk.** These steps and files are provided as-is, with no warranty or guarantee of any kind. The author accepts no liability for any damage, data loss, downtime or other problems that result from using them.
>
> Read and understand anything before you run it, do your own research, and test in a lab before using it in production. See the [MIT License](LICENSE) for the full terms.

## What's here

| Path | Goes to | Purpose |
| --- | --- | --- |
| `host/etc/modules-load.d/vfio.conf` | Proxmox host | Loads the vfio modules at boot |
| `host/etc/modprobe.d/vfio.conf` | Proxmox host | Gives the GPU to vfio-pci before the host drivers get it |
| `host/etc/modprobe.d/blacklist-nvidia.conf` | Proxmox host | Keeps nouveau and nvidia off the host |
| `vm/etc/systemd/system/ollama.service.d/wait-for-gpu.conf` | VM | Stops Ollama starting before the GPU is ready (see [Problems I hit](#problems-i-hit)) |
| `vm/etc/systemd/system/ollama.service.d/network.conf` | VM | Ollama listens on the LAN, not just localhost |
| `vm/opt/open-webui/compose.yaml` | VM | Open WebUI in Docker |
| `caddy/Caddyfile.example` | Reverse-proxy box | Optional HTTPS in front of Open WebUI |

Placeholders used below: `192.0.2.50` is the VM, `192.0.2.10` is the reverse proxy, and `ai.example.lan` is the HTTPS name. Swap in your own.

## 1. Check the host

Run everything in this section as root on the Proxmox node that has the card.

Find the card and its PCI IDs:

```bash
lspci -nn | grep -i nvidia
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GB203 [GeForce RTX 5070 Ti] [10de:2c05]
# 01:00.1 Audio device [0403]: NVIDIA Corporation GB203 High Definition Audio Controller [10de:22e9]
```

Check that the IOMMU is on. AMD turns it on by default. On Intel, add `intel_iommu=on` to the kernel command line in step 2.

```bash
dmesg | grep -iE "AMD-Vi|DMAR|iommu" | head
```

Check that the GPU and its audio function are alone in their IOMMU group. If other devices share the group, they all go to the VM together, so move the card to another slot first.

```bash
G=$(basename $(readlink /sys/bus/pci/devices/0000:01:00.0/iommu_group))
ls /sys/kernel/iommu_groups/$G/devices/
# 0000:01:00.0  0000:01:00.1
```

Check how the card is actually connected. This doesn't affect passthrough, but it's worth knowing (see [Problems I hit](#problems-i-hit)):

```bash
lspci -vv -s 01:00.0 | grep -E "LnkCap:|LnkSta:"
```

## 2. Prepare the host

Back up what you're about to change:

```bash
mkdir -p /root/pre-gpu-passthrough
cp -a /etc/default/grub /etc/modprobe.d /etc/modules-load.d /root/pre-gpu-passthrough/
```

Add `iommu=pt` to the kernel command line. It makes devices the host keeps a little faster, and it's safe to leave on. This host boots with GRUB:

```bash
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt"/' /etc/default/grub
update-grub
```

If your host boots ZFS with systemd-boot instead (if `proxmox-boot-tool status` lists ESPs, it does), add it to the end of `/etc/kernel/cmdline` and run `proxmox-boot-tool refresh`.

Copy the three files from `host/etc/` into place. Then edit `/etc/modprobe.d/vfio.conf` to use **your** card's two IDs from step 1, and rebuild the initramfs:

```bash
update-initramfs -u -k all
reboot
```

After the reboot, both functions should show `vfio-pci`:

```bash
lspci -nnk -s 01:00
#   Kernel driver in use: vfio-pci   (for both 01:00.0 and 01:00.1)
cat /proc/cmdline   # ... quiet iommu=pt
```

## 3. Map the card in the cluster (optional)

A resource mapping gives the card a name. VMs then refer to it by that name, not by one node's PCI address, which is useful in a cluster. In the GUI, it's under Datacenter → Resource Mappings → Add. Or from the shell:

```bash
pvesh create /cluster/mapping/pci --id rtx5070ti \
  --map node=pve03,path=0000:01:00,id=10de:2c05,subsystem-id=1043:89f7,iommugroup=15
```

`subsystem-id` is in `lspci -nnk -s 01:00.0`. Without a mapping, use `--hostpci0 0000:01:00,pcie=1` in the next step instead.

## 4. Create the VM

Download the Ubuntu Server ISO to the node's `local` storage, then create the VM. It needs the q35 machine type, UEFI (OVMF), CPU type `host`, and memory ballooning off, which passthrough requires. Leaving out `,function` (`path=0000:01:00`) passes both the GPU and its audio.

```bash
cd /var/lib/vz/template/iso
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.5-live-server-amd64.iso

qm create 115 --name ai-vm --ostype l26 --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0 \
  --cpu host --sockets 1 --cores 12 --memory 49152 --balloon 0 \
  --scsihw virtio-scsi-single --scsi0 local-lvm:300,discard=on,iothread=1,ssd=1 \
  --ide2 local:iso/ubuntu-24.04.5-live-server-amd64.iso,media=cdrom \
  --net0 virtio,bridge=vmbr0 --agent 1 \
  --hostpci0 mapping=rtx5070ti,pcie=1 --boot "order=scsi0;ide2" --vga std
qm start 115
```

- `pre-enrolled-keys=0` leaves Secure Boot off. Otherwise, after installing the NVIDIA driver you have to enrol its signing key (MOK) at the VM console.
- `--vga std` keeps the Proxmox console working for the install. The NVIDIA card isn't the primary display (`x-vga` is off), which is what you want for compute.
- Sizing is up to you. This host has 16 threads and 60 GB of RAM. Models live on disk, so give it room.

On the host, `dmesg | grep vfio` should show the card being reset cleanly when the VM starts.

## 5. Install Ubuntu

Run the installer from the Proxmox console. Choices that matter:

- **Ubuntu Server**, not minimized. The minimized install leaves out tools you'll want.
- **Leave "third-party drivers" unticked.** RTX 50-series cards need NVIDIA's *open* kernel modules, version 570 or later, and it's better to pick the driver yourself in step 6.
- **Install OpenSSH server.**
- Skip the featured snaps.

The installer only gives the root LVM volume about 100 GB, whatever the disk size. Grow it:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Then update and add the guest agent, so Proxmox can see the VM's IP and shut it down cleanly:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
```

## 6. NVIDIA driver

List the drivers Ubuntu offers and install a recent `-open` one:

```bash
ubuntu-drivers list
sudo apt install -y nvidia-driver-595-open nvidia-utils-595
dkms status          # nvidia/595.x, <kernel>, x86_64: installed
sudo reboot
```

```bash
nvidia-smi           # should list the card and its full memory
```

If `nvidia-smi` says it can't talk to the driver in the first few seconds after boot, wait a moment and try again. The driver takes about 7 seconds to start inside the VM.

## 7. Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Install the two drop-ins from `vm/etc/systemd/system/ollama.service.d/`:

- `wait-for-gpu.conf` is **important**. Without it, Ollama can start before the GPU is ready after a reboot. It then runs on the CPU until it's restarted, and nothing reports an error.
- `network.conf` makes Ollama listen on the LAN. Leave it out to keep Ollama local to the VM.

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo cp wait-for-gpu.conf network.conf /etc/systemd/system/ollama.service.d/
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

Test it, and check that the model is on the GPU:

```bash
ollama pull qwen3:14b
ollama run qwen3:14b "In one sentence, what is a sandpiper?"
ollama ps            # PROCESSOR should say 100% GPU
journalctl -b -u ollama | grep "inference compute"   # library=CUDA, not cpu
```

A 16 GB card fits models up to about 14B parameters comfortably at the default 4-bit quantization (`qwen3:14b`, `gemma3:12b`).

## 8. Open WebUI

Install Docker from Ubuntu's own repository. That's all that's needed here:

```bash
sudo apt install -y docker.io docker-compose-v2
```

Copy `vm/opt/open-webui/compose.yaml` to `/opt/open-webui/`, set `WEBUI_URL`, and start it:

```bash
cd /opt/open-webui
sudo docker compose up -d
curl -s http://localhost:8080/health   # {"status":true}
```

It uses host networking, so it reaches Ollama at `127.0.0.1:11434` and listens on port 8080. Chats and settings are kept in the `open-webui` volume, which survives updates:

```bash
cd /opt/open-webui && sudo docker compose pull && sudo docker compose up -d
```

> [!IMPORTANT]
> The **first account created becomes the admin**. Open the page and sign up straight away.

## 9. HTTPS and firewall (optional)

I put Caddy on another always-on box in front of Open WebUI, with a certificate from my own CA. `caddy/Caddyfile.example` shows the site block. With a public domain, drop the `tls` block and Caddy gets a Let's Encrypt certificate on its own. Point the DNS name at the Caddy box, not at the VM.

Then let only the proxy reach port 8080 on the VM. Because the container uses host networking, ufw rules apply to it normally:

```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 11434/tcp                                      # Ollama API on the LAN (no auth)
sudo ufw allow from 192.0.2.10 to any port 8080 proto tcp     # the reverse proxy only
sudo ufw enable
```

The Ollama API has no authentication. Anyone who can reach port 11434 can run or pull models. If you don't need it on the LAN, leave out `network.conf` and the 11434 rule.

## Problems I hit

**Ollama on the CPU after a reboot.** `ollama ps` showed `100% CPU`, and the log said `inference compute ... library=cpu`. The GPU driver finishes starting about 2 seconds after Ollama starts, and Ollama only looks for GPUs once, at startup. `wait-for-gpu.conf` fixes it. I tested it with three full stop/start cycles from Proxmox.

**The reset problem on RTX 50-series cards.** Some Blackwell cards don't reset cleanly when the VM stops, and the next start hangs, sometimes taking the host with it. My earlier card on another host had issues, so I tested this one with several full stop/start cycles (`qm shutdown` then `qm start`, not just a reboot inside the guest). The host log showed `reset done` each time, and the GPU worked after every start. Run the same test before you rely on it.

**PCIe link narrower than the slot.** `LnkSta: Speed 16GT/s (downgraded), Width x4 (downgraded)` on a card that's capable of x16. On my board, the x16 slot is wired for fewer lanes. Once a model is in GPU memory this hardly matters for chat speed. It slows model loading, and it really hurts models that don't fit in VRAM and spill over to system RAM. Check your board's manual for lane sharing with the M.2 slots, or a bifurcation setting.

**First generation is slow.** The first run after loading a model was 8 tokens/s, and the runs after it were about 80. Measure from the second run.

**`VF BAR ... failed to assign` in the host's dmesg.** These are about SR-IOV virtual functions the consumer card doesn't use, and they're harmless.

**SSH "Too many authentication failures".** Not a passthrough problem, but I hit it logging in to the fresh VM with a password. The SSH agent offers every key it holds, and the server gives up before it asks for the password. Use `ssh -o PubkeyAuthentication=no user@host`, or `ssh-copy-id -o PubkeyAuthentication=no -i key.pub user@host` to install a key.

## Undo

On the host: copy the files back from `/root/pre-gpu-passthrough/`, delete `/etc/modprobe.d/vfio.conf`, `/etc/modprobe.d/blacklist-nvidia.conf` and `/etc/modules-load.d/vfio.conf`, run `update-grub` and `update-initramfs -u -k all`, and reboot.
