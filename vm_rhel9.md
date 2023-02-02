# RHEL 9 Virtual Machine with Libvirt

Start by downloading the RHEL 9.1 ISO boot image for `x86_64` or `aarch64` architecture from the https://developers.redhat.com/products/rhel/download location.

### Creating VM
Create a RHEL virtual machine with 2 cores, 4096MB of RAM and 50GB of storage. 

Install the `libvirt` packages and reboot your system to start the virtualization environment.
```
sudo dnf install -y libvirt virt-manager virt-install virt-viewer libvirt-client qemu-kvm qemu-img
```

Move the ISO image to `/var/lib/libvirt/images` directory and run the following command to create a virtual machine.

```bash
VMNAME="rhel-dev"
sudo -b bash -c " \
cd /var/lib/libvirt/images/ && \
virt-install \
    --name ${VMNAME} \
    --vcpus 2 \
    --memory 4096 \
    --disk path=./${VMNAME}.qcow2,size=50 \
    --network network=default,model=virtio \
    --os-type generic \
    --events on_reboot=restart \
    --cdrom ./rhel-8.7-$(uname -i)-boot.iso \
"
```

In the OS installation wizard, set the following options:
- Root password and `redhat` administrator user
- Select "Installation Destination"
    - Under "Storage Configuration" sub-section, select "Custom" radial button
    - Select "Done" to open a window for configuring partitions
    - Under "New Red Hat Enterprise Linux 9.x Installation", click "Click here to create them automatically"
    - Select the root partition (`/`)
        - On the right side of the menu, set "Desired Capacity" to `40 GiB`
        - On the right side of the menu, verify the volume group is `rhel`.
        - Click "Update Settings" button
        > This will preserve the rest of the allocatable storage for dynamically provisioned logical volumes.
    - Click "Done" button.
    - At the "Summary of Changes" window, select "Accept Changes"

- Connect network card and set the hostname (i.e. `sigstore-dev.ez`)
- Register the system with Red Hat using your credentials (toggle off Red Hat Insights connection)
- In the Software Selection, select Minimal Install base environment and toggle on Headless Management to enable Cockpit

Click on Begin Installation and wait until the OS installation is complete.

### Configuring VM

Log into the virtual machine using SSH with the `redhat` user credentials.

Run the following commands to configure SUDO and upgrade the system.

```bash
echo -e 'redhat\tALL=(ALL)\tNOPASSWD: ALL' | sudo tee /etc/sudoers.d/microshift
sudo dnf clean all -y
sudo dnf update -y
```
