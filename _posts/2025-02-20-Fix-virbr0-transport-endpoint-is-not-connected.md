---
layout: post
title:  "Fix virbr0 transport endpoint is not connected"
category: ""
date:   2025-02-20
author: "Benjamin Oakes"
---

I use `virt-manager` (Virtual Machine Manager) on Ubuntu to run virtual machines, mostly for development.  Recently, I've been preferring to do all development within a virtual machine that I can easily create.  All too often, something will go wrong with the host OS if I develop directly on it.  Yes, even if I use Docker.  (If you use Docker for Mac, you're running Linux within a VM, which is essentially a similar approach.)  I've chosen to use `virt-manager` over GNOME Boxes and VirtualBox because it's much lighter weight but more feature-rich than GNOME Boxes.  My understanding is that it uses more Linux kernel features to achieve this.  It's nice.  Except for when it stops working.

## The Error

After a hard shutdown of my laptop, I started getting this error when attempting to start guest VMs within `virt-manager`:

```
Unable to complete install: '/usr/lib/qemu/qemu-bridge-helper --use-vnet --br=virbr0 --fd=32: failed to communicate with bridge helper: stderr=failed to create tun device: Operation not permitted
: Transport endpoint is not connected

Traceback (most recent call last):
  File "/usr/share/virt-manager/virtManager/asyncjob.py", line 72, in cb_wrapper
    callback(asyncjob, *args, **kwargs)
  File "/usr/share/virt-manager/virtManager/createvm.py", line 2008, in _do_async_install
    installer.start_install(guest, meter=meter)
  File "/usr/share/virt-manager/virtinst/install/installer.py", line 695, in start_install
    domain = self._create_guest(
             ^^^^^^^^^^^^^^^^^^^
  File "/usr/share/virt-manager/virtinst/install/installer.py", line 637, in _create_guest
    domain = self.conn.createXML(initial_xml or final_xml, 0)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/libvirt.py", line 4529, in createXML
    raise libvirtError('virDomainCreateXML() failed')
libvirt.libvirtError: /usr/lib/qemu/qemu-bridge-helper --use-vnet --br=virbr0 --fd=32: failed to communicate with bridge helper: stderr=failed to create tun device: Operation not permitted
: Transport endpoint is not connected
```

## Analysis

Okay, interesting.  So I guess `virt-manager` is written in Python and must hand off to some Linux internals.  But why would it have stopped working?

It can't connect to the transport endpoint, in this case, a virtual bridged network device (`virbr0`).  I had set up bridged networking so that I could `ssh` into the guest VM.  If there's not a clean shutdown, it seems that something goes wrong with the permissions on the virtual bridge `virbr0`.

Some useful resources about `virbr0`:

  - [https://wiki.libvirt.org/VirtualNetworking.html](https://wiki.libvirt.org/VirtualNetworking.html)
  - [https://askubuntu.com/questions/246343/what-is-the-virbr0-interface-used-for](https://askubuntu.com/questions/246343/what-is-the-virbr0-interface-used-for)

## The Fix

I couldn't find any write-ups on this particular situation online.  And honestly?  I had almost given up until I used an LLM to figure out steps to fix it.  To my surprise, it popped back with a fix!  Not everything I was provided was useful, so here's what helped in my scenario.  Some steps might be unnecessary, but they don't appear to hurt anything.

Destroy the virtual network:

```sh
virsh net-destroy default
virsh net-undefine default
service libvirtd restart
ifconfig
```

Recreate the `virbr0` virtual bridge:

> ### 1. **Install `libvirt` if not already installed**
> `virbr0` is created and managed by `libvirt`. If you don't have `libvirt` installed, you can install it with:
> 
> ```bash
> sudo apt update
> sudo apt install libvirt-daemon libvirt-daemon-system bridge-utils -y
> ```
> 
> ### 2. **Restart the `libvirtd` service**
> The `virbr0` bridge is automatically created by `libvirt` when it starts. Restart the service to regenerate it:
> 
> ```bash
> sudo systemctl restart libvirtd
> ```
> 
> You can verify if `virbr0` is recreated by running:
> 
> ```bash
> ip link show virbr0
> ```
> 
> or
> 
> ```bash
> brctl show
> ```
> 
> ### 3. **Manually Recreate `virbr0` if Needed**
> If `virbr0` is still missing, you can manually define it using `virsh`:
> 
> ```bash
> sudo virsh net-define /usr/share/libvirt/networks/default.xml
> sudo virsh net-autostart default
> sudo virsh net-start default
> ```
> 
> This should recreate `virbr0` with NAT forwarding.
> 
> ### 4. **Verify `virbr0` is Active**
> Check its status:
> 
> ```bash
> ip a show virbr0
> ```
> 
> or
> 
> ```bash
> sudo virsh net-list --all
> ```
> 
> You should see the `default` network active.

I also followed these steps:

> ### **1. Ensure `qemu-bridge-helper` Has Correct Permissions**
> The `qemu-bridge-helper` binary must have the correct permissions to allow non-root users to create bridge connections.
> 
> Run the following command:
> 
> ```bash
> sudo setcap cap_net_admin+ep /usr/lib/qemu/qemu-bridge-helper
> ```
> 
> Then, restart `libvirtd`:
> 
> ```bash
> sudo systemctl restart libvirtd
> ```
> 
> ---
> 
> ### **2. Verify `/etc/qemu/bridge.conf` Configuration**
> Ensure that `qemu-bridge-helper` is allowed to use `virbr0`. Edit (or create) the following file:
> 
> ```bash
> sudo vim /etc/qemu/bridge.conf
> ```
> 
> Add this line:
> 
> ```
> allow virbr0
> ```
> 
> Then, save and exit.
> 
> Make sure it has the correct permissions:
> 
> ```bash
> sudo chmod 644 /etc/qemu/bridge.conf
> ```
> 
> ---
> 
> ### **3. Verify `libvirtd` and `qemu-kvm` Are Running**
> Check the status of `libvirtd`:
> 
> ```bash
> sudo systemctl status libvirtd
> ```
> 
> Restart if needed:
> 
> ```bash
> sudo systemctl restart libvirtd
> ```
> 
> Make sure `qemu-kvm` is installed:
> 
> ```bash
> sudo apt install qemu-kvm -y
> ```

That should take care of it!  I hope this helps you if you landed on this page.
