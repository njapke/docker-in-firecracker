# How to Set Up Docker inside a KVM in Firecracker

This guide describes how to get `docker` running inside a KVM in `firecracker`. It is assumed that you are using a linux host system, so some sections might need to be adapted for other host systems (particularly regarding networking).

This guide adapts several of Firecrackers own setup guides. In particular, information from [getting-started](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md), [rootfs-and-kernel-setup](https://github.com/firecracker-microvm/firecracker/blob/main/docs/rootfs-and-kernel-setup.md), and [network-setup](https://github.com/firecracker-microvm/firecracker/blob/main/docs/network-setup.md#in-the-guest) is used here.

## Step 1: Setup Firecracker

First, you need to install `firecracker`. On Arch Linux, you can use the AUR package `firecracker-bin`.

Refer to [this guide](https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md) for getting the binary, if it is not available in the repositories of your linux distro.

## Step 2: Compiling the Linux Kernel

Docker needs some kernel settings, which are not set for the kernel of Firecrackers hello world project. So, we need to compile the kernel ourselves.

1. Clone the linux kernel git repository.

```
git clone https://github.com/torvalds/linux.git linux.git
cd linux.git
```

2. Check out kernel version v5.12 (others will likely work too, but this is the one I used).

```
git checkout v5.12
```

3. Use the kernel configuration included in this repository.

```
cd ..
git clone https://github.com/njapke/docker-in-firecracker.git
cp docker-in-firecracker/kernel_config/config linux.git/.config
cd linux.git
```
Optionally, you can also configure the kernel yourself with the following command.
```
make menuconfig
```
To work, docker needs several settings enabled regarding networking (netlink, netfilter), cgroups, namespaces, AppArmor, and SELinux. See [here](https://examples.javacodegeeks.com/devops/docker/docker-kernel-requirements/) for further information.

4. Build the uncompressed kernel image.
```
make vmlinux
```
After a successful build, the kernel image we need can be found under `./vmlinux`.

## Step 3: Creating a rootfs Image

We also need a rootfs image, which contains the init system and filesystem, as well as other utilities. Like in Firecrackers setup guide, we will build an EXT4 image with Alpine Linux.

1. Create the image file. We will use a size of 1 GiB (1024 MiB), since Docker (plus images and containers) needs a lot of space. Depending on image size, you may need even more space.

```
dd if=/dev/zero of=rootfs.ext4 bs=1M count=1024
```

2. Create the EXT4 file system on the image file.

```
mkfs.ext4 rootfs.ext4
```

3. Mount the image file, so that it is accessible.

```
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
```

4. Now we want to install a minimal Alpine Linux inside the rootfs file. Like in Firecrackers guides, we will use the official Alpine Docker image for that.

Start the Alpine container with the mounted rootfs.

```
docker run -it --rm -v /tmp/my-rootfs:/my-rootfs alpine
```

5. Add basic utilities, like the OpenRC init system and Docker.

```
apk add openrc
apk add util-linux
apd add docker
```

6. Set up userspace.
```
# Set up a login terminal on the serial console (ttyS0):
ln -s agetty /etc/init.d/agetty.ttyS0
echo ttyS0 > /etc/securetty
rc-update add agetty.ttyS0 default

# Make sure special file systems are mounted on boot:
rc-update add devfs boot
rc-update add procfs boot
rc-update add sysfs boot

# Set new root password
passwd root

# Then, copy the newly configured system to the rootfs image:
for d in bin etc lib root sbin usr; do tar c "/$d" | tar x -C /my-rootfs; done

# The above command may trigger the following message:
# tar: Removing leading "/" from member names
# However, this is just a warning, so you should be able to
# proceed with the setup process.

for dir in dev proc run sys var; do mkdir /my-rootfs/${dir}; done

# All done, exit docker shell.
exit
```

5. Unmount the rootfs image.

```
sudo umount /tmp/my-rootfs
```

Now the kernel and rootfs are ready!

## Step 4: Set up the Host Machine for Networking

Before we get started with Firecracker, we need to set up a `tap` device on the host machine for networking.

Create the `tap` device.
```
sudo ip tuntap add tap0 mode tap
```
Set up routing for the `tap` device.
```
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i tap0 -o eth0 -j ACCEPT
```
The above commands assume your networking device is `eth0`. Replace it with your standard networking device in case it is different.

## Step 5: First Launch of the Firecracker VM

1. Copy all the bash scripts in the folder `shell_scripts`, the kernel image, and the rootfs file into one folder. The bash scripts contain API calls to Firecracker to set up the VM. In particular, they do the following.

`1_set_guest_kernel.sh`: Set the kernel that Firecracker uses to `./vmlinux`.
`2_set_guest_rootfs.sh`: Set the rootfs that Firecracker uses to `./rootfs`.
`3_set_network.sh`: Set up a device called `eth0` inside the Firecracker VM that is linked to the device `tap0` we created earlier.
`4_set_resources.sh`: Set higher resources for the Firecracker VM (2 vCPUs and 1024MiB RAM).
`5_start_guest_machine.sh`: Start the Firecracker VM.

2. Open two shell prompts inside the folder you put all of the files. Execute the following commands inside the first shell.
```
rm -f /tmp/firecracker.socket
firecracker --api-sock /tmp/firecracker.socket
```

3. In the second shell, execute all of the bash scripts one after another, in the order implied by their naming. After the last one is finished, the Firecracker VM should start up in the first shell.

4. Now we need to set up Docker properly and start a test container to verify our setup works. Login as `root` and execute the following commands.
```
rc-update add docker default
rc-service docker start
```
This starts and enables the Docker daemon.

5. We need to enable the `eth0` device to get network support.
```
ip addr add 172.16.0.2/24 dev eth0
ip link set eth0 up
ip route add default via 172.16.0.1 dev eth0
```
These commands have to be executed each time the VM is restarted.

6. Test Docker.
```
docker pull alpine
docker run -it alpine
```
If everything was successful, you should enter the shell inside the Docker container.

To exit everything, run
```
exit
reboot
```
The kernel of the Firecracker VM will shutdown after the `reboot` command, since Firecracker does not implement guest power management.
