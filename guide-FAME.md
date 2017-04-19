# Guide to FAME

Author: Yupu Zhang (yupu.zhang@hpe.com)

FAME (Fabric-Attached Memory Emulation) provides an emulated environment where multiple virtual
machines/processes (nodes) share the same global memory pool (fabric-attached memory) backed by a
file (persistent) or memory (volatile) on the host. It does not emulate the non-cache-coherent
aspect of FAM, but it offers a great platform for software development on FAM.

FAME consists of a set of [open-sourced tools](https://github.com/FabricAttachedMemory). The
following guide outlines the steps necessary to set up FAME on a Ubuntu host.

## Setup the host

- Install host OS (Ubuntu 16.04 LTS)  
  ISO image: http://releases.ubuntu.com/16.04/ubuntu-16.04.2-server-amd64.iso

- Update and upgrade
  ```
  $ sudo apt-get update
  $ sudo apt-get upgrade
  $ sudo reboot
  ```

- Add a sudo user: l4fame
  ```
  $ sudo adduser l4fame
  $ sudo usermod -aG sudo username
  ```

- Setup qemu
  - Log into host as user l4fame
  - Install qemu and libvirt
    ```
    $ sudo apt-get install qemu-kvm libvirt-bin
    ```
    Note, Ubuntu 16.04 comes with qemu 2.5

  - Configure and check bridged network
    ```
    $ sudo virsh net-autostart default
    $ sudo virsh net-list --all
      Name                 State      Autostart     Persistent
     ----------------------------------------------------------
      default              active     yes           yes

    $ sudo brctl show
      bridge name     bridge id               STP enabled     interfaces
      virbr0          8000.525400e6e03d       yes             virbr0-nic
    ```

  - Setup l4fame user
    ```
    $ echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> /home/l4fame/.bashrc
    $ sudo adduser l4fame kvm
    $ sudo adduser l4fame libvirtd
    ```
    For qemu version 2.6+, run:
    ```
    $ sudo adduser l4fame libvirt
    ```

  - Create a global FAM backend file (/dev/shm/GlobalNVM)
    ```
    $ touch /dev/shm/GlobalNVM
    $ chmod 666 /dev/shm/GlobalNVM
    $ fallocate -l 16G /dev/shm/GlobalNVM
    ```
    The file here must match the file specified in the VM configuration file (e.g., node01.xml).  
    If you want the backend file to persist across host reboots, place it in your local file system.

  - Disable AppArmor (https://help.ubuntu.com/lts/serverguide/apparmor.html)  
    AppArmor would prevent virtual machines from accessing the FAM backend file.
    ```
    $ sudo systemctl stop apparmor.service
    $ sudo update-rc.d -f apparmor remove
    ```

  - Reboot
    ```
    $ sudo reboot
    ```

- Make the host FAME ready
  - Install tm-librarian

    ```
    $ cd ~
    $ git clone https://github.com/FabricAttachedMemory/tm-librarian.git
    ```

## Setup guest VMs

- Build l4fame kernel for guests
  - Log into host as user l4fame
  - Build linux-l4fame

    ```
    $ cd ~
    $ sudo apt-get install build-essential bc libssl-dev
    $ git clone https://github.com/FabricAttachedMemory/linux-l4fame.git
    $ cd linux-l4fame
    $ ./build-l4fame.sh
    ```
    Note: to utilize multiple cores (e.g., 4), edit build-l4fame.sh and replace:
    ```
    fakeroot make deb-pkg || exit 1
    ```
    with
    ```
    fakeroot make -j5 deb-pkg || exit 1
    ```

  - The new kernel deb files are for the guests, NOT for the host

- Setup one guest VM (node01)
  - Log into host as user l4fame
  - Create the virtual disk image

    ```
    $ cd ~
    $ mkdir fame
    $ cd fame
    $ qemu-img create -f qcow2 ./node01.qcow2 10G
    ```

  - Download the OS image

    ```
    $ wget http://releases.ubuntu.com/16.04/ubuntu-16.04.2-server-amd64.iso
    ```

  - Config the VM (in this example, node01, one cpu, 768MB mem, 10GB disk)  
    Create a file "node01.xml" and copy the following lines

    ```
    <domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
      <name>node01</name>
      <memory unit='KiB'>786432</memory>
      <currentMemory unit='KiB'>786432</currentMemory>
      <vcpu placement='static'>1</vcpu>
      <os>
        <type arch='x86_64' machine='pc-i440fx-2.1'>hvm</type>
        <boot dev='cdrom'/>
        <boot dev='hd'/>
      </os>
      <features>
        <acpi/>
        <apic/>
        <pae/>
      </features>
      <cpu mode='host-model' match='exact'>
        #<model fallback='allow'>SandyBridge</model>
      </cpu>
      <clock offset='utc'>
        <timer name='rtc' tickpolicy='catchup'/>
        <timer name='pit' tickpolicy='delay'/>
        <timer name='hpet' present='no'/>
      </clock>
      <on_poweroff>destroy</on_poweroff>
      <on_reboot>restart</on_reboot>
      <on_crash>restart</on_crash>
      <pm>
        <suspend-to-mem enabled='no'/>
        <suspend-to-disk enabled='no'/>
      </pm>
      <devices>
        <emulator>/usr/bin/kvm</emulator>
        <disk type='file' device='cdrom'>
          <driver name='qemu' type='raw'/>
          <source file='/home/l4fame/fame/ubuntu-16.04.2-server-amd64.iso'/>
          <target dev='hdc' bus='ide'/>
          <readonly/>
          <address type='drive' controller='0' bus='1' unit='0'/>
        </disk>
        <disk type='file' device='disk'>
          <driver name='qemu' type='qcow2'/>
          <source file='/home/l4fame/fame/node01.qcow2'/>
          <target dev='vda' bus='virtio'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
        </disk>
        <controller type='usb' index='0' model='ich9-ehci1'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x7'/>
        </controller>
        <controller type='usb' index='0' model='ich9-uhci1'>
          <master startport='0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0' multifunction='on'/>
        </controller>
        <controller type='usb' index='0' model='ich9-uhci2'>
          <master startport='2'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x1'/>
        </controller>
        <controller type='usb' index='0' model='ich9-uhci3'>
          <master startport='4'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x2'/>
        </controller>
        <controller type='pci' index='0' model='pci-root'/>
        <controller type='ide' index='0'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
        </controller>
        <controller type='virtio-serial' index='0'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
        </controller>
        <interface type='network'>
          <mac address='52:54:00:01:01:01'/>
          <source network='default'/>
          <model type='virtio'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
        </interface>
        <serial type='pty'>
          <target port='0'/>
        </serial>
        <console type='pty'>
          <target type='serial' port='0'/>
        </console>
        <channel type='spicevmc'>
          <target type='virtio' name='com.redhat.spice.0'/>
          <address type='virtio-serial' controller='0' bus='0' port='1'/>
        </channel>
        <input type='tablet' bus='usb'/>
        <input type='mouse' bus='ps2'/>
        <input type='keyboard' bus='ps2'/>
        <graphics type='spice' autoport='yes'/>
        <sound model='ich6'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
        </sound>
        <video>
          <model type='qxl' ram='65536' vram='65536' heads='1'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
        </video>
        <redirdev bus='usb' type='spicevmc'>
        </redirdev>
        <redirdev bus='usb' type='spicevmc'>
        </redirdev>
        <redirdev bus='usb' type='spicevmc'>
        </redirdev>
        <redirdev bus='usb' type='spicevmc'>
        </redirdev>
        <memballoon model='virtio'>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
        </memballoon>
        <graphics type='vnc' port='-1' autoport='yes' keymap='en-us'/>
      </devices>
      <qemu:commandline>
        <qemu:arg value='-object'/>
        <qemu:arg value='memory-backend-file,size=16G,mem-path=/dev/shm/GlobalNVM,id=GlobalNVM,share=on'/>
        <qemu:arg value='-device'/>
        <qemu:arg value='ivshmem,x-memdev=GlobalNVM'/>
      </qemu:commandline>
    </domain>
    ```

    Note: for qemu version 2.6+, replace

    ```
      <qemu:arg value='ivshmem,x-memdev=GlobalNVM'/>
    ```

    with

    ```
      <qemu:arg value='ivshmem-plain,memdev=GlobalNVM'/>
    ```

  - Add the VM
    ```
    $ virsh define node01.xml
    ```

  - Start the VM
    ```
    $ virsh start node01
    ```

  - Install VNC
    ```
    $ sudo apt-get install vnc4server xvnc4viewer
    ```

  - Connect to the VM through VNC  
    First find out the address and port:
    ```
    $ virsh vncdisplay node01
    ```
    Then run VNC viewer
    ```
    $ xvnc4viewer <host:port> &
    ```

  - Install the OS and create a new sudo user: l4fame

  - After OS is finished installing, but before rebooting the VM, eject the OS image:
    ```
    $ virsh change-media node01 hdc --eject
    ```
  - Now reboot the VM.

  - Repeat these steps for every guest VM, but change the xml file name, the VM name and mac address in the xml file

- Make the guest FAME ready  
  In the host:
  - Log into host as user l4fame
  - Start a node VM (e.g., node01)

    ```
    $ virsh start node01
    ```

  - Get the VM's IP address (guest_ip_addr)

    ```
    $ virsh net-dhcp-leases default
    ```

  - Copy the l4fame deb files to the VM

    ```
    $ scp ~/*.deb guest_ip_addr:~/
    ```

  In the guest:
  - Log into the VM as user l4fame
  - Install l4fame kernel

    ```
    $ cd ~
    $ sudo dpkg -i *.deb
    $ sudo reboot
    ```

  - Load zbridge module

    ```
    $ sudo modprobe zbridge
    ```

  - Install tm-libfuse

    ```
    $ sudo apt-get install automake libtool gettext make
    $ git clone https://github.com/FabricAttachedMemory/tm-libfuse.git
    $ cd tm-libfuse
    $ git checkout hp_l4tm
    $ touch config.rpath
    $ ./makeconf.sh
    $ ./configure --libdir="/lib/x86_64-linux-gnu"
    $ make
    $ sudo make install
    ```

  - Install libpmem

    ```
    $ cd ~
    $ sudo apt-get install autoconf pkg-config doxygen graphviz
    $ git clone https://github.com/FabricAttachedMemory/nvml.git
    $ cd nvml
    $ make
    $ sudo make install
    ```

  - Install libfam-atomic

    ```
    $ cd ~
    $ sudo apt-get install autoconf autoconf-archive libtool
    $ sudo apt-get --no-install-recommends install asciidoc xsltproc xmlto
    $ git clone https://github.com/FabricAttachedMemory/libfam-atomic.git
    $ cd libfam-atomic
    $ bash autogen.sh
    $ ./configure
    $ make
    $ sudo make install
    ```

  - Install tm-librarian

    ```
    $ cd ~
    $ git clone https://github.com/FabricAttachedMemory/tm-librarian.git
    ```

## Configure FAME
- Log into host as user l4fame

- Make sure the FAM backend file still exists. If not, re-create the file (e.g., /dev/shm/GlobalNVM)

  ```
  $ touch /dev/shm/GlobalNVM
  $ chmod 666 /dev/shm/GlobalNVM
  $ fallocate -l 16G /dev/shm/GlobalNVM
  ```

  The file here must match the file specified in the VM configuration file (e.g., node01.xml).  
  If you want the backend file to persist across host reboots, place it in your local file system.

- Configure librarian (e.g., 4 nodes, 4GB per node)
  ```
  $ cd tm-librarian
  ```

  Create a config file (e.g., l4fame_book_data.ini) and add the following lines:

  ```
  [global]
  node_count = 4
  book_size_bytes = 8M
  nvm_size_per_node = 512B
  ```

  Create initial librarian database (e.g., l4fame.db) from the config file:

  ```
  $ ./book_register.py -d l4fame.db l4fame_book_data.ini
  ```

## Start FAME

- Run the librarian in the host
  - Log into host as user l4fame
  - Start librarian using the previously created librarian database (e.g., l4fame.db)

    ```
    $ cd tm-librarian
    $ ./librarian.py --db_file l4fame.db --verbose 3
    ```

- Run the librarian file system in each guest  
  In the host:
  - Log into host as user l4fame
  - Start guest VM (e.g., node01)

    ```
    $ virsh start node01
    ```

  - Get the host's IP address (host_ip_addr)

    ```
    $ ifconfig virbr0 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}'
    ```

  - Get the guest's IP address (guest_ip_addr)

    ```
    $ virsh net-dhcp-leases default
    ```

  In the guest:
  - Log into each guest (e.g., node01) as user l4fame
  - Start the librarian file system (lfs)

    ```
    $ cd tm-librarian
    $ sudo ./lfs_fuse.py --hostname host_ip_addr --physloc 1:1:1
    ```

    Note, use 1:1:2, 1:1:3, and 1:1:4 for node 2, 3, and 4.

- Test if FAME is running correctly  
  In one guest (e.g., node01):
  - Log in as user l4fame
  - Write a message to a file/shelf in /lfs (e.g., /lfs/test)

    ```
    $ echo "hello from node01" > /lfs/test
    ```

  In another guest (e.g., node02):
  - Log in as user l4fame
  - Read the message in /lfs/test

    ```
    $ cat /lfs/test
    ```

    You should see "hello from node01".

  - Reply back using the same file/shelf:

    ```
    $ echo "hello from node02" > /lfs/test
    ```

  Back to the former guest (node01):
  - Check the message in /lfs/test

    ```
    $ cat /lfs/test
    ```

    You should now see "hello from node02"

- All set! Try out the FAM version of ALPS, NVMM, or RadixTree.

## References
https://hlinux-web.us.rdlabs.hpecorp.net/dokuwiki/doku.php/l4tm:qemu_fabric_experience (internal)
