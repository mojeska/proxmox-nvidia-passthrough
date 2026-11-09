# proxmox-nvidia-passthrough

How I passed an NVIDIA RTX 50-series card through Proxmox VE 9 to an Ubuntu 24.04 VM, so I could run local AI (Ollama, with Open WebUI on top). I had done this once before on another host. However, that card has been giving me trouble, so this time I wrote everything down. That includes the parts that didn't go to plan, which I imagine are the parts most people will actually want.

Tested in November 2026 with:

| Piece | Version |
| --- | --- |
| Host | Proxmox VE 9.2, kernel 7.0.14, AMD Ryzen 7 8700G (its Radeon 780M stays with the host) |
| GPU | NVIDIA GeForce RTX 5070 Ti, 16 GB, in an OCuLink dock (PCIe 4.0 x4) |
| VM | Ubuntu Server 24.04.5, kernel 6.8 |
| Driver | `nvidia-driver-595-open` 595.99.02 (CUDA 13.2) |
| Software | Ollama 0.40.15, Open WebUI (`main`), Docker 29.1 from Ubuntu's repo |

The end result: `qwen3:14b` runs entirely on the GPU at about **80 tokens/s**, and `llama3.2:3b` at **150-190 tokens/s**. Not bad for a card hanging off a cable.

## Disclaimer

> [!WARNING]
> **Use at your own risk.** These steps and files are provided as-is, with no warranty or guarantee of any kind. The author accepts no liability for any damage, data loss, downtime or other problems that result from using them.
>
> Read and understand anything before you run it, do your own research, and test in a lab before using it in production. See the [MIT License](LICENSE) for the full terms.

## What's here

The README is the guide. The other files are the exact configs I'm running, with my addresses swapped out.

| Path | Goes to | Purpose |
| --- | --- | --- |
| `host/etc/modules-load.d/vfio.conf` | Proxmox host | Loads the vfio modules at boot |
| `host/etc/modprobe.d/vfio.conf` | Proxmox host | Hands the GPU to vfio-pci before the host drivers can grab it |
| `host/etc/modprobe.d/blacklist-nvidia.conf` | Proxmox host | Keeps nouveau, and nvidia off the host |
| `vm/etc/systemd/system/ollama.service.d/wait-for-gpu.conf` | VM | Stops Ollama starting before the GPU is ready (see [Things that bit me](#things-that-bit-me)) |
| `vm/etc/systemd/system/ollama.service.d/network.conf` | VM | Ollama listens on the LAN, not just localhost |
| `vm/opt/open-webui/compose.yaml` | VM | Open WebUI in Docker |
| `caddy/Caddyfile.example` | Reverse proxy | Optional HTTPS in front of Open WebUI |

Placeholders: `192.0.2.50` is the VM, `192.0.2.10` is the reverse proxy, and `ai.example.lan` is the HTTPS name. Swap in your own.

## 1. Check the host first

Everything in this section runs as root on the Proxmox node with the card. None of it changes anything, so it's worth doing before you start editing files.

Find the card and its two PCI IDs (the GPU, and its HDMI audio). You'll need both later.

```bash
lspci -nn | grep -i nvidia
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GB203 [GeForce RTX 5070 Ti] [10de:2c05]
# 01:00.1 Audio device [0403]: NVIDIA Corporation GB203 High Definition Audio Controller [10de:22e9]
```

Make sure the IOMMU is on. AMD turns it on by default. On Intel you'll need to add `intel_iommu=on` to the kernel command line in step 2.

```bash
dmesg | grep -iE "AMD-Vi|DMAR|iommu" | head
```

Check that the GPU, and its audio function are alone in their IOMMU group. Everything in a group goes to the VM together, so if the card shares its group with something the host needs (a NIC, the boot drive) you'll want to try another slot before going any further.

```bash
G=$(basename $(readlink /sys/bus/pci/devices/0000:01:00.0/iommu_group))
ls /sys/kernel/iommu_groups/$G/devices/
# 0000:01:00.0  0000:01:00.1
```

Lastly, check how the card is actually connected. It doesn't affect passthrough, but it does affect what to expect later (more on that below).

```bash
lspci -vv -s 01:00.0 | grep -E "LnkCap:|LnkSta:"
```

## 2. Prepare the host

Back up what you're about to change. If something goes sideways, this is your way back.

```bash
mkdir -p /root/pre-gpu-passthrough
cp -a /etc/default/grub /etc/modprobe.d /etc/modules-load.d /root/pre-gpu-passthrough/
```

Add `iommu=pt` to the kernel command line. It's essentially a small speed-up for the devices the host keeps, and it's safe to leave on. My host boots with GRUB:

```bash
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt"/' /etc/default/grub
update-grub
```

If your host boots ZFS with systemd-boot (`proxmox-boot-tool status` will list ESPs if it does), add it to the end of `/etc/kernel/cmdline` instead, and run `proxmox-boot-tool refresh`.

Copy the three files from `host/etc/` into place. Then edit `/etc/modprobe.d/vfio.conf` so it has **your** two IDs from step 1, not mine (it's an easy one to miss). Rebuild the initramfs, and reboot:

```bash
update-initramfs -u -k all
reboot
```

After the reboot, both functions should show `vfio-pci`. If either still shows `nouveau` or `snd_hda_intel`, the IDs in `vfio.conf` are the first place I'd look.

```bash
lspci -nnk -s 01:00
#   Kernel driver in use: vfio-pci   (for both 01:00.0 and 01:00.1)
cat /proc/cmdline   # ... quiet iommu=pt
```

## 3. Map the card in the cluster (optional)

My nodes are in a cluster, so I gave the card a resource mapping. The VM then refers to the card by name instead of one node's PCI address, which makes it a bit more obvious which node the VM belongs on. In the GUI it's under Datacenter → Resource Mappings → Add, or from the shell:

```bash
pvesh create /cluster/mapping/pci --id rtx5070ti \
  --map node=pve03,path=0000:01:00,id=10de:2c05,subsystem-id=1043:89f7,iommugroup=15
```

`subsystem-id` comes from `lspci -nnk -s 01:00.0`. On a single host you can skip this, and use `--hostpci0 0000:01:00,pcie=1` in the next step instead.

## 4. Create the VM

Download the Ubuntu Server ISO to the node's `local` storage, then create the VM. The settings that matter are q35, UEFI (OVMF), CPU type `host`, and ballooning off (passthrough pins the VM's memory, so ballooning has nothing to work with). Leaving the function off the address (`path=0000:01:00`) passes the GPU, and its audio together.

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

A few notes on those choices:

- `pre-enrolled-keys=0` leaves Secure Boot off. With it on, you have to enrol the NVIDIA driver's signing key (MOK) at the console after installing it. It's doable, but it's one more thing to go wrong, and this VM's only job is running models.
- `--vga std` keeps the Proxmox console working for the install. The NVIDIA card isn't the primary display (`x-vga` is off), which is what you want for compute.
- Size it to taste. My host has 16 threads and 60 GB of RAM. Models live on disk, and they add up quickly, so don't be stingy with storage.

On the host, `dmesg | grep vfio` should show the card being reset cleanly when the VM starts.

## 5. Install Ubuntu

Run the installer from the Proxmox console. Most of it is pretty self-explanatory; these are the choices that matter:

- **Ubuntu Server**, not minimized. Minimized saves almost nothing on a 300 GB disk, and you'll spend the time you saved reinstalling things like an editor.
- **Leave "third-party drivers" unticked.** RTX 50-series cards need NVIDIA's *open* kernel modules (570 or later), and I'd rather pick the driver myself than hope the installer picks the right one.
- **Install OpenSSH server.**
- Skip the featured snaps.

For whatever reason, the installer only gives the root LVM volume about 100 GB, no matter how big the disk is. Grow it:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Then update, and add the guest agent so Proxmox can see the VM's IP, and shut it down cleanly:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
```

## 6. NVIDIA driver

See what Ubuntu offers, and install a recent `-open` driver:

```bash
ubuntu-drivers list
sudo apt install -y nvidia-driver-595-open nvidia-utils-595
dkms status          # nvidia/595.x, <kernel>, x86_64: installed
sudo reboot
```

```bash
nvidia-smi           # should list the card, and its full memory
```

If `nvidia-smi` complains that it can't talk to the driver right after boot, don't panic. The driver takes about 7 seconds to come up inside the VM. Give it a moment, and try again.

## 7. Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Then install the two drop-ins from `vm/etc/systemd/system/ollama.service.d/`:

- `wait-for-gpu.conf` is the important one. Without it, Ollama can start before the GPU is ready, run on the CPU until it's restarted, and never mention it. See [Things that bit me](#things-that-bit-me).
- `network.conf` makes Ollama listen on the LAN. Leave it out if you only want Open WebUI talking to it.

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo cp wait-for-gpu.conf network.conf /etc/systemd/system/ollama.service.d/
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

Test it, and make sure the model is actually on the GPU (don't trust the speed alone; see below):

```bash
ollama pull qwen3:14b
ollama run qwen3:14b "In one sentence, what is a sandpiper?"
ollama ps            # PROCESSOR should say 100% GPU
journalctl -b -u ollama | grep "inference compute"   # library=CUDA, not cpu
```

With 16 GB, models up to roughly 14B parameters fit comfortably at the default 4-bit quantization (`qwen3:14b`, `gemma3:12b`). Bigger than that, and you're into spilling over to system RAM, which on my setup really isn't worth it.

## 8. Open WebUI

Open WebUI runs in Docker. I'm not going to cover Docker itself here, but Ubuntu's own packages are all you need:

```bash
sudo apt install -y docker.io docker-compose-v2
```

Copy `vm/opt/open-webui/compose.yaml` to `/opt/open-webui/`, set `WEBUI_URL` to whatever address people will use, and start it:

```bash
cd /opt/open-webui
sudo docker compose up -d
curl -s http://localhost:8080/health   # {"status":true}
```

It uses host networking, so it reaches Ollama on `127.0.0.1:11434`, and listens on port 8080. Chats and settings live in the `open-webui` volume, so updating doesn't lose anything:

```bash
cd /opt/open-webui && sudo docker compose pull && sudo docker compose up -d
```

> [!IMPORTANT]
> The **first account created becomes the admin**. Open the page and sign up straight away, before someone else on your network does it for you.

## 9. HTTPS and firewall (optional)

I put Caddy on another always-on box in front of Open WebUI, with a certificate from my own internal CA. `caddy/Caddyfile.example` has the site block. With a public domain you can drop the `tls` block, and Caddy will get a Let's Encrypt certificate on its own. Point the DNS name at the Caddy box, not the VM (I got that backwards the first time).

Once HTTPS works, there's no reason for anything but the proxy to reach port 8080. Since the container uses host networking, plain ufw rules apply to it:

```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 11434/tcp                                      # Ollama API on the LAN (no auth)
sudo ufw allow from 192.0.2.10 to any port 8080 proto tcp     # the reverse proxy only
sudo ufw enable
```

Keep in mind the Ollama API has no authentication at all. Anyone who can reach port 11434 can run models, or pull new ones. On a home network that's probably fine; if it isn't for you, leave out `network.conf`, and the 11434 rule.

## Things that bit me

**Ollama quietly running on the CPU.** After a reboot, `ollama ps` showed `100% CPU`, and the log said `inference compute ... library=cpu`. The cause is a race: the GPU driver finishes starting about 2 seconds after Ollama does, and Ollama only looks for GPUs once, at startup. It doesn't error, it just gets slow. `wait-for-gpu.conf` fixes it, and I checked it over three full stop/start cycles from Proxmox.

**The RTX 50-series reset problem.** Some Blackwell cards don't reset cleanly when the VM stops, and the next start hangs (sometimes taking the host with it). Given my other card's history, I tested this one with several full stop/start cycles (`qm shutdown` then `qm start`, not just a reboot inside the guest). The host log showed `reset done` every time, and the GPU worked after each start. I'm not sure if that's this card, the newer kernel or luck, so run the same test before you rely on it.

**PCIe link shows x4.** `LnkSta: Speed 16GT/s (downgraded), Width x4 (downgraded)` on a card that can do x16. My card is in an external dock connected by OCuLink, which tops out at PCIe 4.0 x4, so this is expected and passthrough doesn't care. Once a model is in GPU memory it hardly affects chat speed. Where you'll feel it is model loading, and models too big for VRAM that spill over to system RAM. If your card is in a motherboard slot and shows x4, check the manual for lanes shared with the M.2 slots, or a bifurcation setting.

**The first run is slow.** The first generation after loading a model came in at 8 tokens/s, and every run after that was about 80. Measure from the second run, or you'll go chasing a problem that isn't there.

**`VF BAR ... failed to assign` in the host's dmesg.** These are about SR-IOV virtual functions, which a consumer card doesn't use. As far as I can tell they're harmless.

**SSH "Too many authentication failures".** Not a passthrough problem, but it got me while logging in to the fresh VM. The SSH agent offers every key it holds, and the server gives up before it ever asks for a password. `ssh -o PubkeyAuthentication=no user@host` gets you in, and `ssh-copy-id -o PubkeyAuthentication=no -i key.pub user@host` installs a key.

## Undo

On the host, copy the files back from `/root/pre-gpu-passthrough/`, delete `/etc/modprobe.d/vfio.conf`, `/etc/modprobe.d/blacklist-nvidia.conf`, and `/etc/modules-load.d/vfio.conf`, then run `update-grub`, `update-initramfs -u -k all`, and reboot.

## What's next

If you're starting out, I'd begin with a 12-14B model, make sure `ollama ps` says 100% GPU, and only then go looking for bigger ones. And if you've done this with other RTX 50-series cards, especially if you hit the reset problem, I'd be interested to hear what worked.
