
# OpenWrt in a Firecracker Micro VM

Boots in five seconds. There are a number of large sleeps in the OpenWrt boot process that can probably be avoided in a VM (no actual hardware to wait for).

- [Basic usage](#basic-usage)
- [DHCP for eth1 / wan](#dhcp-for-eth1-wan)
- [VLANs](#vlans)
- [Multiple VMs](#multiple-vms)

### Requirements

- Host architecture x86_64 or aarch64
- Firecracker 1.1.0 or later -- https://github.com/firecracker-microvm/firecracker/releases
- OpenWrt 22.03 or later -- https://downloads.openwrt.org

### Known issues

- Power off and reboot: to power off, use `reboot`. Firecracker hasn't implemented ACPI controls yet, so `poweroff` halts the guest but keeps the VM open, and `reboot` exits the VM as you'd expect from `poweroff`. You can also always do `sudo killall firecracker` for a non-graceful shutdown.
- Squashfs: Firecracker supports it, but the only other supported fstype is ext4 (not jffs2), so the proper overlay filesystem setup would need a lot of work in OpenWrt.

## Basic usage

Get the kernel and rootfs image.
```sh
wget -O extract-vmlinux.sh https://github.com/torvalds/linux/raw/master/scripts/extract-vmlinux
chmod +x extract-vmlinux.sh

wget -O bzImage https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-generic-kernel.bin
./extract-vmlinux.sh bzImage > vmlinux

wget -O rootfs.img.gz https://downloads.openwrt.org/snapshots/targets/x86/64/openwrt-x86-64-generic-ext4-rootfs.img.gz
gunzip rootfs.img.gz
```

On simple x86-64 systems such as VMs, OpenWrt automatically uses the `eth0` interface for the `lan` bridge and, if present, `eth1` for `wan`/`wan6`.

Create host-side interfaces for the VM's eth0/lan and eth1/wan interfaces. The interface names must match what's specified in `vmconfig.json` - `tap0` will be used as `eth0` and `tap1` as `eth1`. (The address on `tap0` can alternatively also be obtained via DHCP from OpenWrt, for example using `dhclient` or NetworkManager.)
```sh
sudo ip tuntap add dev tap0 mode tap
sudo ip tuntap add dev tap1 mode tap
sudo ip addr add 10.0.1.2/24 dev tap0
sudo ip addr add 10.0.42.1/24 dev tap1
sudo ip link set dev tap0 up
sudo ip link set dev tap1 up
```

Create the VM configuration.
```sh
cat vmconfig.json
{
  "boot-source": {
    "kernel_image_path": "./vmlinux",
    "boot_args": "ro console=ttyS0 noapic reboot=k panic=1 pci=off nomodules random.trust_cpu=on i8042.noaux"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "./rootfs.img",
      "is_root_device": true,
      "is_read_only": false
    }
  ],
  "network-interfaces": [
    {
      "iface_id": "eth0",
      "guest_mac": "02:fc:00:00:00:05",
      "host_dev_name": "tap0"
    },
    {
      "iface_id": "eth1",
      "guest_mac": "02:fc:00:00:00:06",
      "host_dev_name": "tap1"
    }
  ],
  "machine-config": {
    "vcpu_count": 1,
    "mem_size_mib": 128,
    "smt": false
  }
}
```

In a separate shell, start the VM. Note that changes to the ext4 rootfs partition are persistent. Firecracker can also be run as a manager daemon with an HTTP API.
```sh
sudo firecracker --no-api --config-file ./vmconfig.json
```

Now the host can already ping the guest's `lan` bridge, and vice versa.
```sh
ping 10.0.1.1
ssh root@10.0.1.1 ping 10.0.1.2
```

To stop the VM, you use `reboot` inside (see known issues) or `kill` outside.
```sh
sudo killall firecracker
```

## DHCP for eth1 / wan

To get the VM's `wan` interface up, again in a separate shell, start the host's DHCP server for the guest. The `-d` option enables debug mode, so dnsmasq prints the incoming DHCP requests. If you want the VM to have Internet access, you'd also need to enable forwarding and possibly NAT/masquerading separately.
```sh
sudo dnsmasq -d --bind-interfaces --listen-address=10.0.123.1 --dhcp-range=10.0.123.10,10.0.123.100
```

## VLANs

Because TAP interfaces carry Ethernet frames (while TUN interfaces carry IP packets), many convenient things just work, including VLANs.

On the host:
```sh
sudo ip link add link tap0 name tap0.foo type vlan id 42
sudo ip addr add 10.42.0.2/24 dev tap0.foo
sudo ip link set tap0.foo up
```

In the VM:
```sh
ip link add link eth0 name eth0.foo type vlan id 42
ip addr add 10.42.0.1/24 dev eth0.foo
ip link set eth0.foo up
```

Et voila:
```sh
ping 10.42.0.1
ssh root@10.42.0.1 ping 10.42.0.2
```

## Multiple VMs

Each VM needs its own TAP interfaces and `vmconfig.json` file.

If you want to run multiple VMs with a bridge connecting their TAP interfaces, you'd need to enable ARP proxying so the VMs can see each other via the host.
```sh
sudo sysctl net.ipv4.conf.tap0.proxy_arp=1 # optional
```

## Without root

It might be possible to run Firecracker without root, by utilizing rootless Podman with KVM in the container: https://stackoverflow.com/questions/48422001/how-to-launch-qemu-kvm-from-inside-a-docker-container && https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md && https://github.com/containers/podman/blob/main/rootless.md

---

License: Creative Commons Attribution-ShareAlike 4.0 International
