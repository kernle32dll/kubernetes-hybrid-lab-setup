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

Annoyingly, the default installation media does not per default contain or
enable an SSH server. You can either boot up the image with keyboard and
display attached once, and enable it via `systemctl enable dhcpcd`, or you can
do it the smart way:

While still having the prepared SD card in your computer, mount the second partition
(root partition, not boot), and `ln` the ssh server service. The latter is basically
the way how to enable `systemd` services without `systemctl`. 

In my case, the SD card is at `/dev/sdd` - adjust accordingly!

```shell script
mnt /dev/sdd2 /mnt
mkdir -p /mnt/etc/systemd/system/multi-user.target.wants
ln -s /usr/lib/systemd/system/sshd.service /mnt/etc/systemd/system/multi-user.target.wants/
umount /mnt
```

We will use a more elegant solution for the worker, but for now, this is how
it goes for the master installation.

Now, insert the SD card into the Pi, and boot into the system. We will first make
sure it is up to date (important for btrfs, more on this later), as well as
installing a few more tools we will need later:

```shell script
pacman-key --init
pacman-key --populate archlinuxarm

# Update system and reboot
# (required, or btrfs module might not load)
pacman -Syu parted dosfstools arch-install-scripts btrfs-progs

reboot
```

The reboot is required, as it might happen that your kernel is not the most recent
at time of booting the system. What this entails is that the btrfs modules get
installed into the most recent kernel module folder, and not the running kernel.
This causes following commands to fail. So its best to reboot now, so you are all set.

## Bootstrapping the actual system

Now with our bootstrapping system up and running, its time to bootstrap the final
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

parted /dev/sda mklabel msdos
parted /dev/sda mkpart primary fat32 1MB 200MB
parted /dev/sda mkpart primary btrfs 200MB 100%
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

We are now ready to setup the system!

### 2: Install the system

Arch Linux - in any architecture - contains a very cool tool to bootstrap (or how
its called here - `pacstrap`) a new system. As outlined at the beginning of this
document, its a bit of a hassle to pull of with the Raspberry Pi. However, I think
the result is very rewarding, as we can install and customize the system down to
each package.

```shell script
pacstrap /mnt base uboot-raspberrypi linux-aarch64 firmware-raspberrypi raspberrypi-bootloader-x uboot-tools \ 
mkinitcpio-systemd-tool tinyssh-convert tinyssh busybox btrfs-progs cryptsetup \ 
sudo openssh dhcpcd openntpd htop lm_sensors nano zsh zsh-completions grml-zsh-config \ 
docker cni-plugins conntrack-tools ethtool ebtables socat
```

Lets break down the packages, while they get installed onto your Pi (it may take a
good while):

```
base uboot-raspberrypi linux-aarch64 firmware-raspberrypi raspberrypi-bootloader-x uboot-tools
```

These are the base files, which are always required. Kudos to `ptanmay143` for
[figuring these out](https://www.reddit.com/r/archlinuxarm/comments/harxbk/how_to_build_archlinuxarm_rpi3_tarballs/fv8loy2/).
`uboot-tools` is required for regenerating the U-Boot bootloader down the line.

```
mkinitcpio-systemd-tool tinyssh-convert tinyssh busybox btrfs-progs cryptsetup
```

These are required for the init stage of the system, where we will implement
remote LUKS unlocking via tinyssh (don't worry - you can always unlock
with a keyboard connected to the Pi too, if something goes wrong)

```
sudo openssh dhcpcd openntpd htop lm_sensors nano zsh zsh-completions grml-zsh-config 
```

Personal choice for packages I like. `sudo` allows us to use a dedicated user instead
of using the root user directly (there is a good reason for that down the line),
`openssh` server makes administration much easier, and as I use dhcp in my network,
`dhcpcd` is pretty much required. `openntpd` is a simple implementation of a `ntp`
sync client, to keep the system clock in sync (the Pi does not have a hardware clock!).

The rest of these packages are completely personal choices, and can be substituted at
will. The rest of the document will assume these however, so adjust as necessary.

```
docker cni-plugins conntrack-tools ethtool ebtables socat 
```

These packages are required for Kubernetes (or to be more precise, `kubeadm`) to run. 

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
adjust these as required. These commands as-is setup a US-English system, with a
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

Now is also the time to do some config for Kubernetes and Docker. This could be done
later, but its convenient to do this in chroot, as the services are not running yet.

```shell script
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

mkdir /etc/docker/
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

Next baby step - enable required services, and set a new `root` password.

```shell script
systemctl enable sshd dhcpcd docker openntpd

chsh --shell $(which zsh)
passwd root

# If you don't want to use stdin for the password:
#echo "root:rootpw" | chpasswd
```

While we are at it, we will create a dedicated used, with `sudo` privileges:

```shell script
useradd --shell $(which zsh) --create-home alarm
passwd alarm

# If you don't want to use stdin for the password:
#echo "alarm:rootpw" | chpasswd

usermod --append --groups wheel alarm

sed -i "s/# %wheel ALL=(ALL) ALL/%wheel ALL=(ALL) ALL/" /etc/sudoers
```

The last step to finish the preliminary setup is to configure our init process, so we can
unlock the encrypted root partition, as well as some other shenanigans.

```shell script
# add btrfs utils for rescue operations
# /usr/lib/libgcc_s.so.1 is required for ssh cryptsetup 
sed -i "s/BINARIES=()/BINARIES=(\/usr\/bin\/btrfs \/usr\/lib\/libgcc_s.so.1)/" /etc/mkinitcpio.conf

# Load USB (pcie_brcmstb) and ethernet (broadcom) modules for recovery and remote unlock
sed -i "s/MODULES=()/MODULES=(broadcom pcie_brcmstb)/" /etc/mkinitcpio.conf

# Add correct keymap and encryption, and add support for seperate /usr partition
sed -i 's/keyboard fsck/keyboard keymap systemd systemd-tool usr fsck/' /etc/mkinitcpio.conf

# Enable required initrd services
systemctl enable initrd-cryptsetup.path
systemctl enable initrd-network.service
systemctl enable initrd-sysroot-mount.service
systemctl enable initrd-shell.service
systemctl enable initrd-tinysshd.service
```

For the remote unlock to work, we need to tell our init what it needs to unlock. This is,
the second partition. Its good practice to use UUIDs, so that's what we do here:

```shell script
SDA_LUKS_UUID=$(lsblk -l -f -n -o UUID /dev/sda2 | head -n 1)
echo "root    UUID=$SDA_LUKS_UUID   none   luks,discard" > /etc/mkinitcpio-systemd-tool/config/crypttab
```

To complete the init config, we need to generate and add a public key. However, we need
to - again - jump through a few hoops here.

The way `systemd-tool` works, it poses a little problem: `systemd-tool` is hardwired to
both the `root` user, and the root users authorized keys (`/root/.ssh/authorized_keys`).
This basically means, that anyone able to unlock the system, has (as it stands) the ability
to gain `root` access. This is probably not desirable.

We will tighten security by disallowing SSH root logins altogether. This is another reason
why we created a dedicated user - as it is the only way into our system (remotely) then.

```shell script
echo "PermitRootLogin no" >> /etc/ssh/sshd_config
```

As we use `tinyssh` for the remote unlock ssh server, as need to use a `ed25519` key. If
you don't have one or want to use a dedicated one, create it now **on your host system**,
and `cat` the public key, so we can install it manually on our new system:

```shell script
ssh-keygen -t rsa -b 4096 -a 100 -f ~/.ssh/ida_rsa_claystone_master_unlock
cat ~/.ssh/ida_rsa_claystone_master_unlock.pub
```

Back in our new system, add the public key retrieved from the `cat` command above (your
output will of course differ to mine) to `/root/.ssh/authorized_keys`:

```shell script
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOx49aPqVW9SocPLnAOldDKDOoyiQN8oFS1AYSD9BcuW bgerda@voidlight" >> /root/.ssh/authorized_keys
```

While you are at it, you might or might not also create and add a public key for the
`alarm` user. The process is the same, but the authorized keys reside at
`/home/alarm/.ssh/authorized_keys`. Don't forget the `mkdir` call beforehand.

With the public key(s) in place, we ensure that the host keys for our regular SSH server
are generated. This needs to be done *now* (as opposed to first system start), as these
are converted and copied into the tinyssh config while building the initramfs.

```shell script
ssh-keygen -A
```

With the configuration complete, its now time to regenerate our initramfs. This will
take some time, and should finish without any error. Ignore warnings regarding missing
firmwares - this is completely normal, and can be ignored on the Pi.

```shell script
mkinitcpio -P
```

The final step is to adjust the U-Boot config to point to the unlocked root partition,
and regenerate the `scr` file (note: This **needs** to be done inside `/boot`).

```shell script
sed -i "s/root=PARTUUID=\${uuid}/root=\/dev\/mapper\/root cryptdevice=UUID=\${uuid}:root rootfstype=btrfs rootflags=subvol=@,compress=lzo/" /boot/boot.txt

cd /boot
./mkscr
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
double check the steps above. You can at any time revert to using the bootstrap system in
the SD slot, and your target system in the USB adapter, setting up the mounts again, and
chrooting into the system again. If nothing helps, just start over at the `wipefs` step.

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

RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
cd $DOWNLOAD_DIR
curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/arm64/{kubeadm,kubelet,kubectl}
chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | tee /etc/systemd/system/kubelet.service
mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable kubelet
systemctl start kubelet

kubeadm config images pull
```

Finally, we are ready to create our cluster. But first, we need to talk about a known
problem.

When you execute `kubeadm init --pod-network-cidr=10.244.0.0/16`, it will fail. It will
fail, because `btrfs` is not officially supported. I ignored it, and went ahead anyway.
That's what the second line with `--ignore-preflight-errors=SystemVerification` is for.
However, first try to init without it, to catch any other problem that might exist.

```shell script
kubeadm init --control-plane-endpoint claystone-master1 --pod-network-cidr=10.244.0.0/16

# Check without ignore-preflight-errors first. If you ONLY get
# the btrfs error, proceed.
kubeadm init --control-plane-endpoint claystone-master1 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=SystemVerification 

# https://blog.nobugware.com/post/2019/bare-metal-kubernetes-quick-installations-on-arch/
# In short: arch linux installs its cni stuff into /usr/lib/cni/ instead of /opt/cni/bin/
sed -i "s/--network-plugin=cni/--network-plugin=cni --cni-bin-dir=\/usr\/lib\/cni/" /var/lib/kubelet/kubeadm-flags.env

systemctl restart kubelet

# Persist kubectl config
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# weave insists on getting installed into /opt/cni/bin/
# make a link into /usr/lib/cni/
ln -s /opt/cni/bin/weave-net /usr/lib/cni/weave-net

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.NO_MASQ_LOCAL=1"
```

Some further notes about the commands above:

The `--control-plane-endpoint` parameter for `kubeadm init` serves as preparation
for elevating the master setup to a HA cluster later down the line. As noted by the
kubeadm docs: You cannot upgrade the master setup to a cluster later, if you don't
specify the `control-plane-endpoint` **now**. It is possible to change the value
later, but it must be set now.

As hinted by the comment above, there is a slight difference where the cni binaries
are put on arch linux. This is easily addressed by changing the kubeadm env file
after init.

I decided to go with weave for the pod network add-on. There is no specific reason for
it, other than it known to play well with both ARM and x86. 

## Finish

Congratulations, you successfully installed your first Kubernetes cluster with a single
master on your Raspberry Pi! What a ride. Take some depth breaths, and play around a bit
with your new cluster. E.g. execute `kubectl get pods -A` to see whats running behind the
scenes.

If you are ready, we will do pretty much the same again, with some important twists, in
the next step - where we setup the worker, so we can actually deploy something on our cluster.