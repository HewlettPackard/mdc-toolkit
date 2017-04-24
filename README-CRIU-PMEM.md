# CRIU-PMEM: Application-transparent Checkpoint with Persistent Memory

**Authors**: Yuan Chen (yuan.chen@hpe.com), Oliver Moreno (oliver.moreno@hpe.com)

## Description

**CRIU-PMEM** is system level, an application-transparent tool for suspending and
resuming Linux applications and Docker containers with persistent
memory.  It optimizes **CRIU** (**C**heckpoint/**R**estore **I**n
**U**serspace) software tool to support fast checkpoint and restore
with persistent memory, such as HPE NVDIMMs.  

### Use Scenarios
- Fast checkpoint and recovery from crash and power loss
- "Save" ability (application snapshot creation) in applications and containers
- Application startup boost
- Desktop environment snapshots
- Application/container duplication for debugging and analysis for load balancing
- Instant OS and kernel update

## Master Source
- HPE Internal : https://github.hpe.com/labs/criu-pmem
- External:      https://github.com/HewlettPackard/criu-pmem

## Maturity
Basic CRIU functionality is working properly using NVDIMMs as fast storage devices with some performance optimizations. Full PMEM-awareness and optimization is work in progress. 

## Dependencies
- CRIU (Checkpoint/Restore In Userspace): https://criu.org/Main_Page
- Google Protocol Buffer https://criu.org/Installation#Protocol_Buffers
- NVDIMM and DAX support https://nvdimm.wiki.kernel.org/
- NVM library: pmem.io

## Usage

#### Configue Linux Kernel

Linux kernel v3.11 or newer is required, with some specific options set. If your distribution does not provide the needed kernel, you might want to compile one yourself. Please see CRIU web page for the detailed kenerl options that must be enabled.

https://criu.org/Installation#Linux_Kernel

Here is an example how to check the kernel option CONFIG_CHECKPOINT_RESTORE. 
```
grep CONFIG_CHECKPOINT_RESTORE= /boot/config-`uname -r`
CONFIG_CHECKPOINT_RESTORE=y
```
Please refer to https://linuxconfig.org/in-depth-howto-on-linux-kernel-configuration for how to configure linux kernel.

#### Install Google Protocol Buffer 

Follow the instructinos at https://criu.org/Installation#Protocol_Buffers

On Ubuntu, you can just run the following command to install Google Protocol Buffer.
```
 sudo apt-get install --no-install-recommends git build-essential libprotobuf-dev libprotobuf-c0-dev protobuf-c-compiler protobuf-compiler python-protobuf libnl-3-dev libpth-dev pkg-config libcap-dev asciidoc libnet-dev
```
<!---
####  Install Other Packages (optional)

If you would like to use make test you should install libaio-devel (RPM) or libaio-dev (DEB).
For test launcher zdtm.py you need PyYAML (RPM) or python-yaml (DEB).
-->

#### Get CRIU-PMEM
HPE Internal: 
```
git clone https://github.hpe.com/labs/criu-pmem
```
External
```
git clone https://github.com/HewlettPackard/criu-pmem
```
<!---
The following commands show how to create an ext4 filesystem in the pmem0 device:

```
mkfs -t ext4 /dev/pmem0
mkdir /mnt/pmem0
mount -o dax /dev/pmem0 /mnt/pmem0
```
-->

#### Build CRIU-PMEM

```
cd criu-pmem/criu
make 
sudo ./criu/criu check
```

#### Test CRIU-PMEM

Here is an example that suspends and restores a simple Linux application using CRIU-PMEM.

Build and run a simple program such as 'top' or the following program.

```
cd criu-pmem/criu/examples
gcc -o count count.c
./count
```

Open another console and run CRIU-PMEM to suspend and restore the count program.
```
pgrep count  
sudo criu-pmem/criu/criu/criu dump -t <process-id> -D <directory> -j
sudo criu-pmem/criu/criu/criu restore -D <directory> -j
```
"process-id" is the process id of the application that will be checkpointed.

"directory" is to store the checkpoints and can be any directory on HDD, SSD or NVM (real or emulated one).

You may get the following warnings, which can be ignored. 
```
Warn  (criu/autofs.c:77): Failed to find pipe_ino option (old kernel?)
Warn  (criu/arch/x86/crtools.c:133): Will restore 15138 with interrupted system call
```

####  Another implemenation

This is another implementation of CRIU-PMEM using Non-Volatile-Memory Library (nvml). It's in branch 'libpmem'. Here are the steps to build and test this version.

Download and install nvml (follow the instrutions at https://github.com/pmem/nvml/tree/master

switch to branch 'libpmem'
```
git checkout libpmem
```

Follow the instructions in the master branch to build and test CRIU-PMEM. 

## Interfaces

In addtion to commonad line interface, CRIU also provides RPC and C API.  
- Command line interface: https://criu.org/CLI
- C API: https://criu.org/C_API
- RPC: https://criu.org/RPC

## Demos

Here are two demos that checkpoint and restart MySQL server and docker container on a HPE ProLiant G9 server with NVDIMMS. 

https://asciinema.org/a/4au6y1zasz9p1ej7lwljmfjjo

https://asciinema.org/a/ezkky8ai2ifoyqlufs2iovx2m

<!---
#### Checkpoint and restore a docker container
##### Install experimental Docker binary (Docker Engine 1.10.0)
###### Ubuntu
```
wget https://github.com/boucher/docker/releases/download/v1.10_2-16-16-experimental/docker-1.10.0-dev
service docker stop
mv /usr/bin/docker /usr/bin/docker.Orig
cp docker-1.10_2-16-16-experimental/docker-1.10.0-dev /usr/bin/docker
service docker start
```
###### Fedora
```
sudo dnf update

open /etc/yum.repos.d/docker.repo for edit and add
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/fedora/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg

dnf install docker-2:1.10.3-50.gita612434.fc24.x86_64
service docker stop
wget https://github.com/boucher/docker/releases/download/v1.10_2-16-16-experimental/docker-1.10.0-dev
mv /usr/bin/docker/docker-1.10.0-dev /usr/bin/docker.Orig
cp docker-1.10_2-16-16-experimental /usr/bin/docker
service docker start
```
<!---
##### Checkpoint a docker container
```
docker ps
docker checkpoint --image-dir <pathToDumpFiles> <cid>
```
##### Restore a docker container
```
docker restore --image-dir <pathToDumpFiles> <cid>
```
-->
## Notes

#### Configure NVM device (optional)

You can test CRIU-PMEM with any storage devices such as HDD, SSD or NVM. 

You can emulate a NVM device using DRAM. The detail instructions are avaiable at https://nvdimm.wiki.kernel.org/. Here is an example showing how to reserve 16GB DRAM to emualte a NVDIMM device.

- Install 4.x kernel.
- Include this kernel parameter "memmap=16G!16G".
- Check that the pmem device is ready to use: lsblk pmem0m
- mkfs.ext4 /dev/pmem0m
- Check the new partition: lsblk pmem0m
- Format the partition: mkfs.ext4 -F /dev/pmem0m
- Mount the partition:  mount -o dax /dev/pmem0m /mnt/pmem/

#### How to Configure Logical Volumes with NVDIMM Devices (optional)

Linux 4.8 and newer kernels are required to support DAX with LVM.

Logical Volume Manager (LVM) can aggregate physical disks and/or disk partitions as a Physical Volume (PV) in a Volume Group (VG); The Volume Groups can be partitioned in Logical Volumes (LV) and over them a filesystem can be created. This means, we can define a Logical Volume with a higher capacity than a single physical disk or disk partition. The NVDIMM devices can be treated as other physical disks and therefore we can define Logical Volumes over them.

To configure a Logical Volume, these are the steps involved:
- From NVDIMM device(s), create a Physical Volume(s): pvcreate /dev/pmem0 /dev/pmem1 /dev/pmem2
- Create a Volume Group from existing Physical Volume(s): vgcreate vg_nvdimm /dev/pmem0 /dev/pmem1 /dev/pmem2, to display the existing Volume Groups use the vgdisplay command.
- Create a Logical Volume from an existing Volume Group: lvcreate -–name lv_ndimm --size 12G vg_nvdimm, to display the existing Logical Volumes use the lvdisplay command.
- Assign a filesystem to the Logical Volume: mkfs.ext4 /dev/vg_nvdimm/lv_nvdimm
- mount –o dax /dev/vg_nvdimm/lv_nvdimm /pmem

## See Also

- **Checkpoint-Restore in User Space**: http://criu.org
- **Improving Preemptive Scheduling with Application-Transparent Checkpointing in Shared Clusters**.  Jack Li, Yuan Chen, Vanish
    Talwar, Calton Pu, and Dejan Milojicic.  *Proceedings of 2015 ACM/IFIP/USENIX Middleware Conference (Middleware 2015)*, Dec. 2015.
- **Persient Memory wiki**: https://nvdimm.wiki.kernel.org/
- **NVM Programming Library**: http://pmem.io/nvml/
- **HPE NVDIMM**: http://linux.hpe.com/nvdimm/
