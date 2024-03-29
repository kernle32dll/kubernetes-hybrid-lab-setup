# 01: Setting up the master

## Overview and grand scheme of things

The installation on the Raspberry Pi is a bit more challenging, than a usual
install (such as we will do with the worker in the next chapter).

In the grand scheme of things, it will go something like this:

- We boot off a SD card, which contains the default `Arch Linux ARM` live image
(`aarch64` variant - important!)
- We use this system, to bootstrap the actual system onto a different sd card,
which is connected via an USB adapter
- We chroot into this install, and do the initial setup and config there
- Finally, we swap sd cards, and finalize the installation after ssh'ing into
the new system

Early on, I was thinking about bootstrapping the system from an USB stick,
or installing the system itself onto a USB stick. I discarded the latter as
I went for an industrial grade SD card, which levitated by doubts about
system lifetime. I discarded the former, as I was unable to properly boot up
the aarch64 variant. It might have something to do with the fact that the aarch64
variant uses `U-Boot` instead of the Raspberry Pi own bootloader. This might
get resolved some time, but it did not in time for this guide.

## Update bootloader / eeprom (optional)

Initially required to get USB booting required. We will not use this feature,
but its good practice to update the Raspberry Pi anyway, as this cannot be
done easily from anything but the foundations own `Raspberry Pi OS` (formerly
`Raspbian`).

First, get the [Raspberry Pi OS Lite](https://www.raspberrypi.org/software/operating-systems/),
and [install that onto a SD card](https://www.raspberrypi.org/documentation/installation/installing-images/).
Boot up the system, and you are good to proceed.

Hint: Per default, the Raspberry Pi OS does not sport a SSH server, which makes
working with it a tad uncomfortable. You can however temporarily enable a SSH
server, by placing an empty file called `ssh` onto the SD cards boot partition.
For more details, see point `3` [here](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md).

You can use `vcgencmd bootloader_version` to check your current bootloader version.

Make sure the system itself it up to date with `sudo apt update`, and then check
for a bootloader update via `sudo rpi-eeprom-update -a`. If required, restart
your Pi to execute the installation. Let it finish and boot up into the system again.

If you want to change any of the Pi eeprom config (such as the boot order), now is
the time. You can find available config options to change
[here](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711_bootloader_config.md).
You can edit the config via `sudo -E rpi-eeprom-config --edit`. Hint:
Prefix the command e.g. with `EDITOR=nano ...`, to use a different editor (in this
case `nano`) than the default `vi`.

## Preparing the installation media

First, we get the image, and set up our SD card. The process is outlined nicely
[here](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4).

While it might not be an issue for some people - for me it is pretty annoying
that the default installation media has DHCP disabled. You can either boot up
the image with keyboard and display attached once, and enable and start it via
`systemctl enable dhcpcd --now`, or you can do it the smart way:

While still having the prepared SD card in your computer, mount the second partition
(root partition, not boot), and `ln` the ssh server service. The latter is basically
the way how to enable `systemd` services without `systemctl`. 

In my case, the SD card is at `/dev/sdd` - adjust accordingly!

```shell script
mnt /dev/sdd2 /mnt
mkdir -p /mnt/etc/systemd/system/multi-user.target.wants
ln -s /usr/lib/systemd/system/dhcpcd.service /mnt/etc/systemd/system/multi-user.target.wants/
umount /mnt
```

We will use a more elegant solution to prepare the installation medium with more
customization possibilities for the worker, but for now, this is how it goes for
the master installation.

Now, insert the SD card into the Pi, and boot into the system. We will first make
sure it is up-to-date (important for btrfs, more on this later), as well as
installing a few more tools we will need later.

Note, you can log in as `alarm` or `root` (password is the same as the username).
For SSH, you can only log in as `alarm`. The following commands assume root level
privileges, so if you did that, don't forget to execute `su` to switch to the
root user.

```shell script
pacman-key --init
pacman-key --populate archlinuxarm

# Update system and reboot
# (required, or btrfs module might not load)
pacman -Syu gptfdisk dosfstools arch-install-scripts btrfs-progs

reboot
```

The reboot is required, as it might happen that your kernel is not the most recent
at time of booting the system. What this entails is that the btrfs modules get
installed into the most recent kernel module folder, and not the running kernel.
This causes following btrfs related commands to fail. So it's best to reboot now,
so you are all set.

## Bootstrapping the actual system

Now with our bootstrapping system up and running, It's time to bootstrap the final
system.

SSH into the system, and for the following commands **ALWAYS** make sure you
are on the Pi, and not your host system. You might cause irreparable damage
to your host system otherwise!

Now is the time to attach the final SD card via USB adapter to the Pi. If no
other storage is connected, it will get listed as `/dev/sda`. Let's begin then.

### 1: Prepare SD card

First, we wipe the partition table, and create our partitions as needed:

```shell script
wipefs -a -f /dev/sda

sgdisk -n 0:0:+512MB -c 0:"Arch Boot" -t 0:0700 /dev/sda
sgdisk -n 0:0:0 -c 0:"Arch Root" -t 0:8300 /dev/sda
```

Next, we create our file systems. This also entails LUKS encrypting the root
partitions. Use a secure password, when asked about it, and keep it safe!

```shell script
# Encrypt root partition, and open it under /dev/mapper/root
cryptsetup -y -v --use-random luksFormat /dev/sda2
cryptsetup luksOpen /dev/sda2 root

mkfs.vfat -F32 /dev/sda1
mkfs.btrfs -f /dev/mapper/root
```

As we are using BTRFS, now is the time to create all the subvolumes. Adjust
if you want, I think these are some sane defaults. Note that the `/usr` subvolume
is a special case, which needs to be handled via a `mkinitcpio` hook later.

Note, I also used `compress=lzo` to save space, and `noatime` to reduce unnecessary
writes to the SD card.

```shell script
# Mount btrfs and create subvolumes
mount /dev/mapper/root /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@usr

# Remount with proper subvolumes
umount /mnt
mount -o subvol=@,compress=lzo,noatime /dev/mapper/root /mnt

# Create mount points for subvolumes
mkdir -p /mnt/home
mkdir -p /mnt/.snapshots
mkdir -p /mnt/opt
mkdir -p /mnt/var
mkdir -p /mnt/usr

# Mount subvolumes
mount -o subvol=@home,compress=lzo,noatime /dev/mapper/root /mnt/home
mount -o subvol=@snapshots,compress=lzo,noatime /dev/mapper/root /mnt/.snapshots
mount -o subvol=@opt,compress=lzo,noatime /dev/mapper/root /mnt/opt
mount -o subvol=@var,compress=lzo,noatime /dev/mapper/root /mnt/var
mount -o subvol=@usr,compress=lzo,noatime /dev/mapper/root /mnt/usr

# Mount the boot partition
mkdir /mnt/boot 
mount /dev/sda1 /mnt/boot
```

We are now ready to set up the system!

### 2: Install the system

Arch Linux - in any architecture - contains a very cool tool to bootstrap (or how
its called here - `pacstrap`) a new system. As outlined at the beginning of this
document, it's a bit of a hassle to pull of with the Raspberry Pi. However, I think
the result is very rewarding, as we can install and customize the system down to
each package.

```shell script
pacstrap /mnt base linux-rpi firmware-raspberrypi raspberrypi-bootloader \ 
mkinitcpio-systemd-tool python tinyssh busybox btrfs-progs cryptsetup \ 
sudo openssh dhcpcd htop lm_sensors nano zsh zsh-completions grml-zsh-config dnsutils \ 
containerd cni-plugins conntrack-tools ethtool ebtables socat \ 
raspberrypi-utils rpi4-eeprom vim
```

Let's break down the packages, while they get installed onto your Pi (it may take a
good while):

```
base raspberrypi-firmware linux-rpi firmware-raspberrypi raspberrypi-bootloader
```

These are the base files, which are always required. Kudos to `ptanmay143` for
[figuring these out](https://www.reddit.com/r/archlinuxarm/comments/harxbk/how_to_build_archlinuxarm_rpi3_tarballs/fv8loy2/).
However, contrary to the setup of `ptanmay143`, I opted for a setup without U-Boot.
There is nothing inherently wrong with U-Boot, but since the mainline Kernel can now
be booted by the Raspberry Pi Bootloader (more or less) directly, I wanted to simplify
the boot process a bit. Also, forgoing U-Boot means that the very good documented and
well understood `cmd.txt` and `config.txt` files can be used for the boot process.
More on these files later.

Note, that we are using the Raspberry Pi patched Kernel `linux-rpi`, and not the mainline 
Kernel `linux-aarch64`.
Since all the important functionality has landed in the mainline Kernel, it makes no difference
which Kernel to use for the most part.
However, at the time of writing the `linux-aarch64` package has not been updated for a very
long time.
Hence, I opted for the `linux-rpi` package.

```
mkinitcpio-systemd-tool python tinyssh busybox btrfs-progs cryptsetup
```

These are required for the init stage of the system, where we will implement
remote LUKS unlocking via `tinyssh` (don't worry - you can always unlock
with a keyboard connected to the Pi too, if something goes wrong)

```
sudo openssh dhcpcd htop lm_sensors nano zsh zsh-completions grml-zsh-config dnsutils
```

Personal choice for packages I like. `sudo` allows us to use a dedicated user instead
of using the root user directly (there is a good reason for that down the line),
`openssh` server makes administration much easier, and as I use dhcp in my network,
`dhcpcd` is pretty much required.

The rest of these packages are completely personal choices, and can be substituted at
will. The rest of the document will assume these however, so adjust as necessary.

```
containerd cni-plugins conntrack-tools ethtool ebtables socat 
```

These packages are required for Kubernetes (or to be more precise, `kubeadm`) to run. 

```
raspberrypi-utils rpi4-eeprom vim
```

Lastly, these are three (actually two) optional packages. The first (not to be confused with`firmware-raspberrypi`)
contains some handy tools, such as `vcgencmd` (see `Update bootloader` paragraph at the top). `rpi4-eeprom` contains
the actual tools to update the firmware (which is stored on the eeprom, hence the name). `vim` is required, as it
contains the `xxd` tool, required for bootloader updates.

### 3: Configure the system in chroot

With the installation complete, we only need to generate the new system `fstab`, and then
we can finally enter the chroot of our new system.

```shell script
genfstab -U /mnt >> /mnt/etc/fstab

arch-chroot /mnt zsh
```

Note: Substitute or omit `zsh` in `arch-chroot` with the shell of your desire.

Now we start with some basic system configuration. Check the official
[Arch Linux install guide](https://wiki.archlinux.org/index.php/Installation_guide) guide to
adjust these as required. These commands as-is set up a US-English system, with a
German QWERTZ keymap and timezone. For me, my master will be called `claystone-master1`,
and the hostname will be set accordingly.

```shell script
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime

sed -i 's/#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen

echo "LANG=en_US.UTF-8" >> /etc/locale.conf
echo "KEYMAP=de-latin1" >> /etc/vconsole.conf
echo "claystone-master1" >> /etc/hostname

cat >> /etc/hosts <<END
127.0.0.1 localhost
::1 localhost
END
```

One small but important change is to enable ntp synchronization via `systemd-timesyncd`.
This has a small advantage over other solutions such as `ntpd` or `openntpd`: Once synchronized,
time is also stored in the file system. This is significant, as the Raspberry Pi has no RTC, which effectively
means that after **each reboot** Pi's have a (sometimes quite long) time period where the system clock is wrong,
till ntp synchronization is achieved again. `systemd-timesyncd` does not suffer from this, as it uses the
time _last stored_ in the filesystem on boot (which means time can still drift apart between long shutdown
times, but shouldn't be much of an issue for quick reboots), till synchronization is achieved again.

```shell script
timedatectl set-ntp 1
```

Now is also the time to do some config for Kubernetes and containerd. This could be done
later, but It's convenient to do this in chroot, as the services are not running yet.

```shell script
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# Not necessary if you are still in chroot like this guide says
# If you repeat the steps while in the final system, execute these two:
modprobe overlay
modprobe br_netfilter

# Add containerd default config and adjust systemd stuff
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# https://blog.nobugware.com/post/2019/bare-metal-kubernetes-quick-installations-on-arch/
# In short: arch linux installs its cni stuff into /usr/lib/cni/ instead of /opt/cni/bin/
sed -i 's/bin_dir = "\/opt\/cni\/bin"/bin_dir = "\/usr\/lib\/cni"/' /etc/containerd/config.toml
```

Additionally, you need to set the system cgroup for containerd to `systemd`. You can simply
use the following sed command, to do the necessary change:

```shell script
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Next baby step - enable required services, and set a new `root` password.

```shell script
systemctl enable sshd dhcpcd containerd

chsh --shell $(which zsh)
passwd root

# If you don't want to use stdin for the password:
#echo "root:rootpw" | chpasswd
```

While we are at it, we will create a dedicated user, with `sudo` privileges:

```shell script
useradd --shell $(which zsh) --create-home alarm
passwd alarm

# If you don't want to use stdin for the password:
#echo "alarm:rootpw" | chpasswd

usermod --append --groups wheel alarm

sed -i "s/# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/" /etc/sudoers
```

The last step to finish the preliminary setup is to configure our init process, so we can
unlock the encrypted root partition, as well as some other shenanigans.

```shell script
# add btrfs utils for rescue operations
# /usr/lib/libgcc_s.so.1 is required for ssh cryptsetup 
sed -i "s/^BINARIES=()/BINARIES=(\/usr\/bin\/btrfs \/usr\/lib\/libgcc_s.so.1)/" /etc/mkinitcpio.conf

# Load USB (pcie_brcmstb) and ethernet (genet broadcom) modules for recovery and remote unlock
sed -i "s/^MODULES=()/MODULES=(genet broadcom pcie_brcmstb)/" /etc/mkinitcpio.conf

# Add correct keymap and encryption, and add support for seperate /usr partition
sed -i "s/^HOOKS=.*/HOOKS=(base autodetect block filesystems keyboard keymap systemd systemd-tool usr fsck)/" /etc/mkinitcpio.conf

# Enable required initrd services
systemctl enable initrd-cryptsetup.path
systemctl enable initrd-network.service
systemctl enable initrd-sysroot-mount.service
systemctl enable initrd-shell.service
systemctl enable initrd-tinysshd.service
```

For unlocking to work, we need to tell our init what it needs to unlock: The second partition.
Its good practice to use UUIDs, so that's what we do here:

```shell script
SDA_LUKS_UUID=$(lsblk -l -f -n -o UUID /dev/sda2 | head -n 1)
echo "root    UUID=$SDA_LUKS_UUID   none   luks,discard" > /etc/mkinitcpio-systemd-tool/config/crypttab
```

To complete the init config, we need to generate and add a public key.

As we use `tinyssh` for remote unlocking ssh server, we need to create a `ed25519` key.
If you don't have one or want to use a dedicated one, create it now **on your host system**,
and `cat` the public key, so we can install it on our new system:

```shell script
ssh-keygen -t ed25519 -b 4096 -a 100 -f ~/.ssh/ida_ed25519_claystone_master_unlock
cat ~/.ssh/ida_ed25519_claystone_master_unlock.pub
```

Back in our new system, add the public key retrieved from the `cat` command above (your
output will differ to mine) to `/etc/mkinitcpio-systemd-tool/config/authorized_keys`:

```shell script
mkdir -p /etc/mkinitcpio-systemd-tool/config/
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOx49aPqVW9SocPLnAOldDKDOoyiQN8oFS1AYSD9BcuW bgerda@voidlight" >> /etc/mkinitcpio-systemd-tool/config/authorized_keys
```

This public key can only be used to remotely unlock the Pi, it cannot be used to remotely log into
the booted and unlocked machine.

If you want this, you might also create and add a public key for the `alarm` user while you are at it.
The process is the same, but the authorized keys reside at `/home/alarm/.ssh/authorized_keys`.
Don't forget the `mkdir` call beforehand.

With the public key(s) in place, we ensure that the host keys for our regular SSH server
are generated. This needs to be done *now* (as opposed to first system start), as these
are converted and copied into the tinyssh config while building the initramfs.

```shell script
ssh-keygen -A
```

With the configuration complete, it's now time to regenerate our initramfs. This will
take some time, and should finish without any error. Ignore warnings regarding missing
firmwares - this is completely normal, and can be ignored on the Pi.

```shell script
mkinitcpio -P
```

The final step is to write the `cmd.txt` and `config.txt` files.
The former is the actual Kernel command line, the `config.txt` is used to configure the Pi hardware.

```shell
cat <<EOF | tee /boot/cmd.txt
cgroup_enable=cpuset cgroup_enable=memory cryptdevice=UUID=$SDA_LUKS_UUID:root:allow-discards root=/dev/mapper/root rootflags=subvol=@,compress=lzo rw
EOF
```

The `cryptdevice` parameter points to our encrypted partition, and the parameters after that correctly
set up the initial root with that.
The cgroup parameters ensure that required cgroups for Kubernetes to run are enabled.
For me, some of them were not enabled on the Raspberry Pi `linux-rpi` Kernel.

```shell
cat <<EOF | tee /boot/config.txt
# initramfs path
initramfs initramfs-linux.img followkernel

# Enable 64bit mode
arm_64bit=1

# Run as fast as firmware / board allows
arm_boost=1
EOF
```

Note, if you use the `linux-aarch64` Kernel, you need to execute the following commands as well:

```shell
echo "device_tree=dtbs/broadcom/bcm2711-rpi-4-b.dtb" > /boot/config.txt
echo "kernel=Image" > /boot/config.txt
```

For a final touch, we will tighten security by disallowing SSH root logins altogether, as we created a dedicated user.

```shell script
echo "PermitRootLogin no" >> /etc/ssh/sshd_config
```

The base system is now configured. You can now shutdown the Pi, and switch to the SD
card we installed the system to.

```shell script
exit
umount -R /mnt && shutdown -h now
```

Wait a few seconds for the init procedure to complete. If you are curious, connect a
display to the Pi to at least once see how the boot up looks (you will not need a display
later).

To unlock the system, ssh into the Pi as `root` (which has nothing to do with the actual
system root user). You might want to provide your key file, too. Enter the password for
the encrypted partition - if all goes well, you should get disconnected.

You now can ssh into the Pi again - this time as the `alarm` user.

I made two aliases on my host system to unlock and connect regularly (adjust and add
to `.bashrc`).

```shell script
alias claystone-master-unlock='ssh -i ~/.ssh/ida_rsa_claystone_master_unlock root@claystone-master1'
alias claystone-master='ssh -i ~/.ssh/ida_rsa_claystone_master_alarm alarm@claystone-master1'
```

Note: `claystone-master1` is a fixed dns entry pointing to my Pi. Adjust accordingly
on your end.

If you were unable to unlock the system via SSH, or cannot SSH into the system afterwards,
double-check the steps above. You can at any time revert to using the bootstrap system in
the SD slot, and your target system in the USB adapter, setting up the mounts again, and
chroot'ing into the system again. If nothing helps, just start over at the `wipefs` step.

## Installing Kubernetes

You should now be running inside your newly setup system. What follows is basically a
run-down of the official [Creating a cluster with kubeadm tutorial](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

First, install the Kubernetes binaries, and start up the `kubelet` service.

Note: this also serves as the purpose of fixing the version, as Kubernetes requires
special attention for updates.

```shell script
# Get root access first!
sudo su

DOWNLOAD_DIR=/usr/local/bin
mkdir -p $DOWNLOAD_DIR

CRICTL_VERSION="v1.23.0"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-arm64.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz

RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
cd $DOWNLOAD_DIR
curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/arm64/{kubeadm,kubelet,kubectl}
chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | tee /usr/lib/systemd/system/kubelet.service
mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable kubelet --now

kubeadm config images pull
```

Finally, we are ready to create our cluster, and do some preliminary configuration.

```shell script
cat > k8s-init-config.cfg <<END
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controlPlaneEndpoint: claystone-master1
networking:
  podSubnet: 10.244.0.0/16
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# Enable certificate rotation and kubelet tls serving
serverTLSBootstrap: true
END

kubeadm init --config k8s-init-config.cfg

# Persist kubectl config
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Tell crictl that we are using containerd
export CONTAINER_RUNTIME_ENDPOINT="unix:///run/containerd/containerd.sock"
echo "export CONTAINER_RUNTIME_ENDPOINT=\"unix:///run/containerd/containerd.sock\"" >> /root/.zshrc

# weave insists on getting installed into /opt/cni/bin/
# make a link into /usr/lib/cni/
ln -s /opt/cni/bin/weave-net /usr/lib/cni/weave-net

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.NO_MASQ_LOCAL=1"
```

Some further notes about the config above:

The `controlPlaneEndpoint` config serves as preparation  for elevating the master
setup to a HA cluster later down the line. As noted by the kubeadm docs: You cannot
upgrade the master setup to a cluster later, if you don't  specify the
`controlPlaneEndpoint` **now**. It is possible to _change_ the value later, but it
must be set now.

If not using `Docker` as the container runtime, extra care must be taken that the correct
cgroup is used. For `Docker` it can me detected automatically, but for `containerd` and
others, it needs to be explicitly set to `systemd`. You might remember something about
cgroups and systemd during the `containerd` configuration.

I decided to go with `weave` for the pod network add-on. There is no specific reason for
it, other than it known to play well with both ARM and x86. 

```shell script
mkdir -p ${fpath[1]}
kubeadm completion zsh > "${fpath[1]}/_kubeadm"
kubectl completion zsh > "${fpath[1]}/_kubectl"
```

For good measure, I also installed ZSH shell completion. This is optional.

## Finish

Congratulations, you successfully installed your first Kubernetes cluster with a single
master on your Raspberry Pi! What a ride. Take some depth breaths, and play around a bit
with your new cluster. E.g. execute `kubectl get pods -A` to see what's running behind the
scenes.

If you are ready, we will do pretty much the same again, with some important twists, in
the next step - where we set up the worker, so we can actually deploy something on our cluster.

If you need to set up your Kubernetes cluster anew, just issue `kubeadm reset`, and you
can start over gain.

## Bonus: Overclocking

*Note*: The following is based on [this MagPi](https://magpi.raspberrypi.org/articles/how-to-overclock-raspberry-pi-4])
article. Give it a read if you want a bit more details.

Now that you got your Pi running, it's time to give it a bit more juice. If you followed my
[hardware decisions](./00-hardware-software-decisions.md) along, you are good to go. If not, make **absolutely sure**,
that you have adequate cooling before continuing!

**Disclaimer**: Overclocking is on you - I don't take any responsibility for any damages done!

If you have not installed the `raspberrypi-utils` package during pacstrap time - now is the time.

When installed, you get (amongst other things) access to the `vcgencmd` command, which allows you to inspect properties
about the Pi. The two most interesting commands are as follows:

```shell
# Measures the current clock speed of the CPU - ~1500000000 is normal top-clock
/opt/vc/bin/vcgencmd measure_clock arm
```

```shell
# Measures the current CPU temperature - for my setup its ~30°C during idle.
/opt/vc/bin/vcgencmd measure_temp
```

Now, to actually overclock the Pi, we need to adjust the `confix.txt`. If our setup, it resides at `/boot/config.txt`.

Either as root or via `sudo`, add the following lines to crank it up a good bit:

```text
over_voltage=6
arm_freq=2000
```

Reboot the Pi and - voilà. Enjoy your overclocked Pi. Make sure to check temperatures and clock, to see if everything is
working fine. You might want to experiment (**carefully**) a bit, to see if you can push the clock a bit further, but
the 2ghz above are very pretty safe for *me*.