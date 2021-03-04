# QEMU (Alternative 1)

You keep running your current kernel, and instead run a newer kernel inside QEMU.
You will be able to SSH into your QEMU machine, where your QEMU machine will
have a ZNS drive (/dev/nvme0n1) which is backed by a file in the host's filesystem.
Therefore the data on your ZNS drive can be persistent across reboots.
(However, right now, the persistence support has been dropped in QEMU, in order
to get the basic ZNS support merged first.)

Create a new directory were you will keep the new code, e.g.:
```
mkdir ~/src
```

Clone and build QEMU:
```
cd ~/src
git clone https://github.com/qemu/qemu.git
cd qemu
./configure --target-list=x86_64-softmmu
make -j$(nproc --all)
```

Clone and build the kernel that you will run inside QEMU:
```
cd ~/src
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
make defconfig
./scripts/config --enable CONFIG_BLK_DEV_ZONED
./scripts/config --enable CONFIG_BLK_DEV_NVME
./scripts/config --enable CONFIG_HYPERVISOR_GUEST
./scripts/config --enable CONFIG_PARAVIRT
./scripts/config --enable CONFIG_KVM_GUEST
./scripts/config --enable CONFIG_VIRTIO_PCI
./scripts/config --enable CONFIG_VIRTIO_NET
./scripts/config --enable CONFIG_VIRTIO_BLK
./scripts/config --enable CONFIG_SQUASHFS
./scripts/config --enable CONFIG_SQUASHFS_XZ
./scripts/config --enable CONFIG_DM_MULTIPATH
make olddefconfig
make -j$(nproc --all)
```

Install the kernel uapi headers to ~/src. (We do not want to replace the kernel uapi headers
on the running system.) We will later copy these headers to our QEMU machine.
```
make headers_install INSTALL_HDR_PATH=~/src/kernel_uapi_headers
```

Get a QEMU rootfs image (this guide uses Ubuntu) using:
```
wget https://cloud-images.ubuntu.com/releases/focal/release/ubuntu-20.04-server-cloudimg-amd64.img
```

Create a new image that will be used as our rootfs, based on the Ubuntu image that we just downloaded.
It will not modify the Ubuntu image we just downloaded. Differences will be saved in my-ubuntu-rootfs.img.
It will not take up 128 GB, it will simply allow it to grow up to that size, since this new file will automatically
increase in size when we install new packages.
```
~/src/qemu/qemu-img create -f qcow2 -b ubuntu-20.04-server-cloudimg-amd64.img my-ubuntu-rootfs.img 128G
```

Install cloud-localds on Ubuntu/Debian:
```
sudo apt install cloud-image-utils
```

or on Fedora/RHEL:
```
sudo dnf install cloud-utils
```

We need to tell Ubuntu to create a user that you can use inside your new image. This is done using cloud-config.
The username will be the same as your current user ($USER). There will be no password, the only way to login
will be via SSH. Start by saving the value of your public SSH key into a variable:
```
export PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
```

Now create the user-data file by pasting the following into a terminal and press enter:
```
cat >user-data <<EOF
#cloud-config
users:
  - name: $USER
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh-authorized-keys:
      - $PUB_KEY
EOF
```

Generate the binary file user-data.img using:
```
cloud-localds user-data.img user-data
```

Create a file named launch_qemu.sh containing the following:
Modify the variables to match your setup.
```
#!/bin/sh

QEMU_BINARY=~/src/qemu/build/x86_64-softmmu/qemu-system-x86_64
KERNEL_IMAGE=~/src/linux/arch/x86/boot/bzImage
ROOTFS_DATA=~/src/my-ubuntu-rootfs.img
USER_DATA=~/src/user-data.img
# ZNS_DATA is the file in the host that will reflect the /dev/nvme0n1 in the guest
ZNS_DATA=./zns-data
ZONE_SIZE=128M
ZONE_CAP=124M
# MAX_ACTIVE_ZONES has to be >= MAX_OPEN_ZONES
MAX_ACTIVE_ZONES=12
MAX_OPEN_ZONES=12
QEMU_GUEST_SSH_FWD_PORT=10222
CPU_CORES=4
RAM=8G

if [ ! -f $ZNS_DATA ]; then
  dd if=/dev/zero of=$ZNS_DATA bs=4M count=1K
fi

$QEMU_BINARY -d guest_errors -m $RAM -smp $CPU_CORES -s -drive file=$ROOTFS_DATA,format=qcow2 -drive file=$USER_DATA,format=raw \
             -device nvme,id=nvme0,serial=deadbeef,logical_block_size=4096,physical_block_size=4096 \
             -drive file=$ZNS_DATA,id=mynvme,format=raw,if=none \
             -device nvme-ns,drive=mynvme,bus=nvme0,nsid=1,zoned=true,zoned.zone_size=$ZONE_SIZE,zoned.zone_capacity=$ZONE_CAP,zoned.max_open=$MAX_OPEN_ZONES,zoned.max_active=$MAX_ACTIVE_ZONES \
             -serial stdio -cpu host -vga none -serial pty -nographic -enable-kvm \
             -chardev socket,id=qmp,path=/tmp/test.qmp,server,nowait -mon chardev=qmp,mode=control \
             -netdev user,id=n1,ipv6=off,hostfwd=tcp::$QEMU_GUEST_SSH_FWD_PORT-:22 \
             -device virtio-net-pci,netdev=n1 -kernel $KERNEL_IMAGE \
             -append "console=ttyS0 root=/dev/sda1 net.ifnames=0 biosdevname=0"
```

Make the script executable and run it:
```
chmod +x launch_qemu.sh
./launch_qemu.sh
```

Keep the QEMU terminal running as long as you want your QEMU machine to be running.
When you want to kill your kill your QEMU machine, simply ctrl + c inside the QEMU terminal.
Wait until your QEMU machine has finished booting (remember, you can only login via SSH).
When you want to login to the QEMU machine, simply open a new terminal and SSH into it.
We will login to the QEMU machine in just a moment.

Open a new terminal, and copy the kernel uapi headers that we installed in a previous step
to the QEMU machine:
```
scp -r -P 10222 ~/src/kernel_uapi_headers localhost:
```

From now on, all commands will be performed inside your QEMU machine, so start off by logging on
to the QEMU machine:
```
ssh -X -p 10222 localhost
```

The first thing you should do after logging on to the QEMU machine is to update the package lists
and make sure that we are running the latest security and bug fixes:
```
sudo apt update
sudo apt upgrade
```

Install the kernel uapi headers that comes with Ubuntu, and put them on hold, so that the package
manager won't update them later:
```
sudo apt install linux-libc-dev
sudo apt-mark hold linux-libc-dev
```

The kernel uapi headers that we just installed matches the kernel that came with Ubuntu (which is old).
Since our QEMU is not running the kernel that came with Ubuntu, but rather a newer kernel
that we build ourself, we need to replace the old headers with the headers that actually matches
the kernel version that our QEMU machine is currently running:
```
sudo rsync -vmrl --include='*/' --include='*\.h' --exclude='*' ~/kernel_uapi_headers/include /usr/
```

Create a new ~/src directory where we will keep the code:<br/>(This time on the QEMU machine rather than your host.)
```
mkdir ~/src
```

Now follow the steps in section: Building the packages

All the instructions in the link above should be performed after logging on to the QEMU machine.

# null_blk (Alternative 2)

You build and boot a new kernel on your bare-metal.
You will load a kernel module that will create a zoned block device (/dev/nullb0)
which is backed by RAM. Therefore the data on your zoned block device will not
be persistent across reboots.

Create a new directory were you will keep the new code, e.g.:
```
mkdir ~/src
```

Clone and build the kernel that you will run on your bare-metal:
(Even to use null_blk, in order to use zone capacity in the null_blk driver, you need a recent kernel.)
```
cd ~/src
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

Try to copy the running kernel's config:
```
zcat /proc/config.gz > .config
```

If that didn't work, try copying the config file from /boot:
```
cp /boot/config-$(uname -r) .config
```

Enable Zoned block device support and the null_blk driver as a module before you build:
```
./scripts/config --enable CONFIG_BLK_DEV_ZONED
./scripts/config --enable CONFIG_CONFIGFS_FS
./scripts/config --module CONFIG_BLK_DEV_NULL_BLK
make olddefconfig
make -j$(nproc --all)
```

Since there are changes in the kernel uapi headers, we need to hold back further updates to the kernel uapi headers
provided by your distro (since, unless your distro is shipping with kernel v5.9+, the headers provided by your distro
are too old to build the rest of the stack). How you do this depends on your distro, but on Ubuntu/Debian you do:
```
sudo apt install linux-libc-dev
sudo apt-mark hold linux-libc-dev
```

Install your new kernel uapi headers, kernel modules, and kernel:
```
sudo make headers_install INSTALL_HDR_PATH=/usr
sudo make modules_install
sudo make install
```

Reboot and boot your new kernel.

After booting your new kernel, create a file named nullblk-zoned.sh containing the following:
```
#!/bin/bash

if [ $# != 7 ]; then
        echo "Usage: $0 <sect size (B)> <zone size (MB)> <zone capacity (MB)> <nr conv zones> <nr seq zones> <max active zones> <max open zones>"
        exit 1
fi

scriptdir="$(cd "$(dirname "$0")" && pwd)"

modprobe null_blk nr_devices=0 || return $?

function create_zoned_nullb()
{
        local nid=0
        local bs=$1
        local zs=$2
        local zc=$3
        local nr_conv=$4
        local nr_seq=$5
        local max_active_zones=$6
        local max_open_zones=$7

        cap=$(( zs * (nr_conv + nr_seq) ))

        while [ 1 ]; do
                if [ ! -b "/dev/nullb$nid" ]; then
                        break
                fi
                nid=$(( nid + 1 ))
        done

        dev="/sys/kernel/config/nullb/nullb$nid"
        mkdir "$dev"

        echo $bs > "$dev"/blocksize
        echo 0 > "$dev"/completion_nsec
        echo 0 > "$dev"/irqmode
        echo 2 > "$dev"/queue_mode
        echo 1024 > "$dev"/hw_queue_depth
        echo 1 > "$dev"/memory_backed
        echo 1 > "$dev"/zoned

        echo $cap > "$dev"/size
        echo $zs > "$dev"/zone_size
        echo $zc > "$dev"/zone_capacity
        echo $nr_conv > "$dev"/zone_nr_conv
        echo $max_active_zones > "$dev"/zone_max_active
        echo $max_open_zones > "$dev"/zone_max_open

        echo 1 > "$dev"/power

        echo mq-deadline > /sys/block/nullb$nid/queue/scheduler

        echo "$nid"
}

nulldev=$(create_zoned_nullb $1 $2 $3 $4 $5 $6 $7)
echo "Created /dev/nullb$nulldev"
```

Make the script executable and run it like this to create a 4 GB zoned block device with 512 B sector size,
128 MB zone size, 124 MB zone capacity, 32 sequential zones, 12 max active zones, and 12 max open zones:
```
chmod +x nullblk-zoned.sh
sudo ./nullblk-zoned.sh 512 128 124 0 32 12 12
```

You should now have a /dev/nullb0 device.

Now follow the steps in section: Building the packages

# Building the packages

Clone and build nvme-cli:
```
sudo apt install build-essential pkg-config
cd ~/src
git clone https://github.com/linux-nvme/nvme-cli.git
cd nvme-cli
make -j$(nproc --all)
sudo make install
```

You can now run e.g.: (If running on null_blk, you cannot run nvme-cli against /dev/nullb0,
since null_blk does not implement the NVMe protocol. It instead targets the Linux
zoned block device (zbd) layer, so use either blkzone or zbd instead.)
```
sudo nvme zns report-zones /dev/nvme0n1
```

Clone and build blkzone:
```
sudo apt install autopoint autoconf automake libtool bison
cd ~/src
git clone https://github.com/karelzak/util-linux.git
cd util-linux
./autogen.sh
./configure
make -j$(nproc --all) blkzone
sudo cp blkzone /usr/sbin/blkzone
```

You can now run e.g.: (If running on null_blk use: /dev/nullb0 instead of /dev/nvme0n1)
```
sudo blkzone report /dev/nvme0n1
```

Clone and build libzbd:
```
sudo apt install autoconf-archive libgtk-3-dev
cd ~/src
git clone https://github.com/westerndigitalcorporation/libzbd.git
cd libzbd
./autogen.sh
./configure
make -j$(nproc --all)
sudo make install
```

You can now run e.g.: (If running on null_blk use: /dev/nullb0 instead of /dev/nvme0n1)
```
sudo zbd report /dev/nvme0n1
sudo -E gzbd &
sudo -E gzbd-viewer /dev/nvme0n1 &
```

Clone and build fio:
```
cd ~/src
git clone https://github.com/axboe/fio.git
cd fio
./configure
make -j$(nproc --all)
sudo make install
```

You can now run e.g.: (If running on null_blk use: /dev/nullb0 instead of /dev/nvme0n1)
```
sudo fio --name=test --filename=/dev/nvme0n1 --zonemode=zbd --direct=1 --runtime=5 --ioengine=io_uring --hipri --rw=randwrite --iodepth=4 --bs=16K
```

You can use gzbd-viewer to see how fio will actually write the data.

Clone and build RocksDB:
```
sudo apt install libgflags-dev libsnappy-dev
cd ~/src
git clone https://github.com/westerndigitalcorporation/rocksdb.git
cd rocksdb
make -j$(nproc --all) db_bench zenfs
```

To test RocksDB, create a file named run_zenfs.sh, in ~/src/rocksdb, containing the following:
(If running on null_blk use: DEV=nullb0 instead of DEV=nvme0n1)
```
#!/bin/sh -ex

DEV=nvme0n1
FUZZ=5
ZONE_SZ_SECS=$(cat /sys/class/block/$DEV/queue/chunk_sectors)
ZONE_CAP=$((ZONE_SZ_SECS * 512))
BASE_FZ=$(($ZONE_CAP  * (100 - $FUZZ) / 100))
WB_SIZE=$(($BASE_FZ * 2))

TARGET_FZ_BASE=$WB_SIZE
TARGET_FILE_SIZE_MULTIPLIER=2
MAX_BYTES_FOR_LEVEL_BASE=$((2 * $TARGET_FZ_BASE))

MAX_BACKGROUND_JOBS=8
MAX_BACKGROUND_COMPACTIONS=8
OPEN_FILES=16

echo deadline > /sys/class/block/$DEV/queue/scheduler
./zenfs mkfs --zbd=$DEV --aux_path=/tmp/zenfs_$DEV --finish_threshold=0 --force
./db_bench --fs_uri=zenfs://$DEV --key_size=16 --value_size=800 --target_file_size_base=$TARGET_FZ_BASE \
 --write_buffer_size=$WB_SIZE --max_bytes_for_level_base=$MAX_BYTES_FOR_LEVEL_BASE \
 --max_bytes_for_level_multiplier=4 --use_direct_io_for_flush_and_compaction \
 --num=1500000 --benchmarks=fillrandom,overwrite --max_background_jobs=$MAX_BACKGROUND_JOBS \
 --max_background_compactions=$MAX_BACKGROUND_COMPACTIONS --open_files=$OPEN_FILES
```

Make the script executable and run it with sudo:
```
chmod +x run_zenfs.sh
sudo ./run_zenfs.sh
```

You can use gzbd-viewer to see how zenfs will actually write the data.
