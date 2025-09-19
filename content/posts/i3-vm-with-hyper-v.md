---
title: Fedora i3 VM setup on Windows 11 using Hyper-V
date: 2025-09-17
---

Recently a thought came to my mind. We live in the year 2025, virtualization has come
very far thanks to the cloud and WSL2 works pretty good now.
Surely it must have become easy and reasonably performant to create a Linux VM on
Windows and use that for development work. Let's even try to use Hyper-V since
that is something the Microsoft Azure Platform is build on.

<!--more-->

## Creating a VM in Hyper-V

This step is actually pretty easy. [Microsoft has a very minimal and good documentation
on how to do it](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-machine-in-hyper-v).
You just need to follow the steps in the Hyper-V VM creation wizard.

After that is done, open the VM settings, search for "Integration Services" and
then make sure to check the "Guest services" checkbox.

## Intializing the Linux VM

After you install the guest OS with the ISO, make sure to not forget to
eject the ISO drive or else Hyper-V will load it again.

If you only want to run a Linux VM that has 0 contact with the host OS,
then you done. You don't need to read further.
However, usually you want to share files, devices and other resources with the host OS.
And that is exactly why I made this blog post.
It is not really straight forward to share these resources on a newly created VM.
I am not sure why though since Hyper-V is a rather advanced Hypervisor.
Well, it is Windows in the end so lets just dive into what we need,
to make sure we have a somewhat pleasant desktop VM experience.

From what I could gather Hyper-V does not provide a native way to share the resources.
Hyper-V has something called "enhanced session mode".
This mode enables the VM to use / share some host OS resources like network connections,
drives and audio devices. But where is the catch? Well, for all these features we
are required to use the RDP protocol. Yes, you read it correctly.
For some reason Microsoft did not think it would be good to provide these features
out-of-the-box but rely on the RDP protocol. At least for me, that is a very weird decision.

This brings us to the next step. To be able to even activate the "enhanced session mode"
for a VM, you need to install and configure a RDP server on your Linux VM.
Otherwise it will simply not work.

### RDP and Xorg

This blog post is about setting up the [i3 window manager](https://i3wm.org/) in a VM on Windows.
i3 is built on top of Xorg but Xorg itself does not implement the RDP protocol
nor does it understand it. So we need to install something called [xrdp](https://www.xrdp.org/).
You don't need to know much about it except it enables Xorg to understand the RDP protocol.
Or to be more precise, xrdp acts as a frontend and Xorg will act as the backend.

### Actual setup

1. The first thing to do after the first boot is to install the following packages:

    ```bash
    sudo dnf install xrdp xrdp-selinux xorgxrdp hyperv-daemons
    ```

    This will install the required services needed to run a RDP server with Xorg as the backend.
    The `hyperv-daemons` package is something very essential here. It installs a couple
    of daemons that enable clipboard integration and other important features.

2. Before starting the xrdp server, you need to optimize some of its configuration.

    The following `sed` commands will handle the optimizations for you:

    ```bash
    sudo sed -i_orig 's/port=3389/port=vsock:\/\/-1:3389/g' /etc/xrdp/xrdp.ini
    sudo sed -i_orig 's/security_layer=negotiate/security_layer=rdp/g' /etc/xrdp/xrdp.ini
    sudo sed -i_orig 's/crypt_level=high/crypt_level=none/g' /etc/xrdp/xrdp.ini
    sudo sed -i_orig 's/bitmap_compression=true/bitmap_compression=false/g' /etc/xrdp/xrdp.ini
    ```

3. One of they daemons installed by `hyperv-daemons` will automatically mount

    all drives that are shared with the VM into your home directory. The default name
    for the mount directory is not very *nice*. The next `sed` command renames it
    to `shared-drives`.

    ```bash
    sudo sed -i 's/FuseMountName=thinclient_drives/FuseMountName=shared-drives/g' /etc/xrdp/sesman.ini
    ```

4. Allow everyone to create X server sessions (required because of RDP):

    ```bash
    sudo tee /etc/X11/Xwrapper.config >/dev/null <<EOL
    needs_root_rights=no
    allowed_users=anybody
    EOL
    ```

5. Make sure the Hyper-V specific kernel module is enabled:

    ```bash
    echo "hv_sock" | sudo tee /etc/modules-load.d/hv_sock.conf
    # Make sure VMware kernel module is blacklisted
    echo "blacklist vmw_vsock_vmci_transport" | sudo tee /etc/modprobe.d/blacklist-vmw_vsock_vmci_transport.conf
    ```

6. Enable the xrdp service and shutdown the VM:

    ```bash
    sudo systemctl enable xrdp
    sudo systemctl enable xrdp-sesman
    poweroff
    ```

7. Enable enhanced session mode for the VM (run this as Administrator in powershell):

    ```powershell
    Set-VM "VM NAME" -EnhancedSessionTransportType HVSocket
    ```

Now you can start the VM again.
A new dialog should now pop up asking you about resolution. Congrats! You are done!
Now you should be able to run the VM in enhanced session mode and share OS resources
with your VM.

There is one remaining issue which I could not solve yet.
When using Xorg as the backend for xrdp, the screen is blurry and pixilated on heavy load.
This is very weird especially because a friend of mine, who uses Arch Linux, can't reproduce this behaviour.
I will update this blog once I find a solution to this issue or if I understand why this even happens.
