## Install Prerequisites

```
sudo apt update
sudo apt install make gcc flex bison libncurses-dev libelf-dev libssl-dev
```

## Kernel

``` bash
git clone --branch v6.2 git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

```
cd linux
make mrproper
cp /boot/config-$(uname -r) .config
```

### Prepare kernel for qemu and syzkaller
- Enable fuzzing/debug options in .config (you can also use make menuconfig)

```
# Coverage collection.
CONFIG_KCOV=y

# Debug info for symbolization.
CONFIG_DEBUG_INFO_DWARF4=y

# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

# Required for Debian Stretch and later
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
```

- Since enabling these options results in more sub options being available, we need to regenerate config:

```
make olddefconfig
```

### Build kernel:

```
make -j$(nproc)
sudo make modules_install
sudo make install
```

- Update grub and reboot into your new kernel:

```
sudo update-grub
sudo reboot
```

# Building syzkaller

- Get GO

```
wget https://dl.google.com/go/go1.23.6.linux-amd64.tar.gz
tar -xf go1.23.6.linux-amd64.tar.gz
export GOROOT=`pwd`/go
export PATH=$GOROOT/bin:$PATH
```

- download and build syzkaller:

```
git clone https://github.com/google/syzkaller
cd syzkaller
make
```

# Building Bullseye Image

### Install debootstrap

```
sudo apt install debootstrap
```

- Create a Debian Bullseye Linux image with the minimal set of required packages.

```
mkdir Image
cd Image/
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
./create-image.sh
The result should be Image/bullseye.img disk image.
```

- [Or to create Debian Linux Image with different version](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md#or-create-debian-linux-image-with-a-different-version)

# Install QEMU

```
sudo apt install qemu-system-x86
```

## Verify 

- Make sure kernel boots and you can access it 

```
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel $KERNEL/arch/x86/boot/bzImage \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=$IMAGE/bullseye.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log
```

> [!NOTE]
> I have not used enable-kvm flag in my setup as i using a remote vm which does not support KVM capabilities

 



