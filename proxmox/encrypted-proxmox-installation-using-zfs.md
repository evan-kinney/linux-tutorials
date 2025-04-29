# Encrypted Proxmox installation using ZFS

## Install Debian 12 Bookworm

### Step 1: Prepare The Install Environment

1. Boot the Debian GNU/Linux Live CD. If prompted, login with the username `user`
and password `live`. Connect your system to the Internet as appropriate (e.g. join
your WiFi network). Open a terminal.
2. Setup and update the repositories:

    ```shell
    sudo nano /etc/apt/sources.list
    ```

    ```txt
    deb http://deb.debian.org/debian bookworm main contrib non-free-firmware
    ```

    ```shell
    sudo apt update
    ```

3. Optional: Install and start the OpenSSH server in the Live CD environment:

    If you have a second system, using SSH to access the target system can be convenient:

    ```shell
    sudo apt install --yes openssh-server

    sudo systemctl restart ssh
    ```

    > [!TIP]
    > You can find your IP address with `ip addr show scope global | grep inet`.
    > Then, from your main machine, connect with `ssh user@IP`.

4. Disable automounting:

    If the disk has been used before (with partitions at the same offsets), previous
    filesystems (e.g. the ESP) will automount if not disabled:

    ```shell
    gsettings set org.gnome.desktop.media-handling automount false
    ```

5. Become root:

    ```shell
    sudo -i
    ```

6. Install ZFS in the Live CD environment:

    ```shell
    apt install --yes debootstrap \
                      gdisk \
                      zfsutils-linux
    ```

### Step 2: Disk Formatting

1. Set variables with the disk names:

    ```shell
    DISK=/dev/disk/by-id/scsi-SATA_disk
    ```

    Always use the long `/dev/disk/by-id/*` aliases with ZFS. Using the `/dev/sd*`
    device nodes directly can cause sporadic import failures, especially on systems
    that have more than one storage pool.

    <!-- markdownlint-disable MD028 -->
    > [!TIP]
    > `ls -la /dev/disk/by-id` will list the aliases.

    > [!TIP]
    > Are you doing this in a virtual machine? If your virtual disk is missing from
    > `/dev/disk/by-id`, use `/dev/vda` if you are using KVM with virtio. Also when
    > using /dev/vda, the partitions used later will be named differently.

    > [!TIP]
    > When choosing a boot pool size, consider how you will use the space. A kernel
    > and initrd may consume around 100M. If you have multiple kernels and take
    > snapshots, you may find yourself low on boot pool space, especially if you
    > need to regenerate your initramfs images, which may be around 85M each. Size
    > your boot pool appropriately for your needs.
    <!-- markdownlint-enable MD028 -->

2. If you are re-using a disk, clear it as necessary:

    Ensure swap partitions are not in use:

    ```shell
    swapoff --all
    ```

    Clear the partition table:

    ```shell
    sgdisk --zap-all $DISK
    ```

3. Partition your disk(s):

    ```shell
    sgdisk     -n2:1M:+512M   -t2:EF00 $DISK

    sgdisk     -n3:0:+1G      -t3:BF01 $DISK

    sgdisk     -n4:0:0        -t4:8309 $DISK
    ```

4. Create the boot pool:

    ```shell
    zpool create \
        -o ashift=12 \
        -o autotrim=on \
        -o compatibility=grub2 \
        -o cachefile=/etc/zfs/zpool.cache \
        -O devices=off \
        -O acltype=posixacl -O xattr=sa \
        -O compression=lz4 \
        -O normalization=formD \
        -O relatime=on \
        -O canmount=off -O mountpoint=/boot -R /mnt \
        bpool \
        ${DISK}-part3
    ```

5. Create the root pool:

    ```shell
    apt install --yes cryptsetup
    ```

    ```shell
    cryptsetup luksFormat -c aes-xts-plain64 -s 512 -h sha256 ${DISK}-part4

    cryptsetup luksOpen ${DISK}-part4 luks1

    zpool create \
        -o ashift=12 \
        -o autotrim=on \
        -O acltype=posixacl -O xattr=sa -O dnodesize=auto \
        -O compression=lz4 \
        -O normalization=formD \
        -O relatime=on \
        -O canmount=off -O mountpoint=/ -R /mnt \
        rpool \
        /dev/mapper/luks1
    ```

### Step 3: System Installation

1. Create filesystem datasets to act as containers:

    ```shell
    zfs create -o canmount=off -o mountpoint=none rpool/ROOT
    zfs create -o canmount=off -o mountpoint=none bpool/BOOT
    ```

2. Create filesystem datasets for the root and boot filesystems:

    ```shell
    zfs create -o canmount=noauto -o mountpoint=/ rpool/ROOT/proxmox
    zfs mount rpool/ROOT/proxmox

    zfs create -o mountpoint=/boot bpool/BOOT/proxmox
    ```

3. Create datasets:

    ```shell
    zfs create                     rpool/home
    zfs create -o mountpoint=/root rpool/home/root
    chmod 700 /mnt/root
    zfs create -o canmount=off     rpool/var
    zfs create -o canmount=off     rpool/var/lib
    zfs create                     rpool/var/log
    zfs create                     rpool/var/spool
    zfs create -o com.sun:auto-snapshot=false rpool/var/cache
    zfs create -o com.sun:auto-snapshot=false rpool/var/lib/nfs
    zfs create -o com.sun:auto-snapshot=false rpool/var/tmp
    chmod 1777 /mnt/var/tmp
    zfs create rpool/srv
    zfs create -o canmount=off rpool/usr
    zfs create                 rpool/usr/local
    zfs create rpool/var/lib/AccountsService
    zfs create rpool/var/lib/NetworkManager
    zfs create -o com.sun:auto-snapshot=false rpool/var/lib/docker
    zfs create rpool/var/snap
    zfs create rpool/var/www
    zfs create -o com.sun:auto-snapshot=false rpool/var/lib/vz
    zfs create rpool/var/lib/nginx
    zfs create -o com.sun:auto-snapshot=false rpool/var/lib/cri-dockerd
    zfs create -o com.sun:auto-snapshot=false rpool/var/lib/containerd
    zfs create rpool/var/lib/portainer
    zfs create rpool/var/lib/kubelet
    zfs create -o mountpoint=/rpool rpool/rpool
    ```

4. Mount a tmpfs at /run:

    ```shell
    mkdir /mnt/run
    mount -t tmpfs tmpfs /mnt/run
    mkdir /mnt/run/lock
    ```

5. Install the minimal system:

    ```shell
    debootstrap bookworm /mnt
    ```

6. Copy in zpool.cache:

    ```shell
    mkdir /mnt/etc/zfs
    cp /etc/zfs/zpool.cache /mnt/etc/zfs/
    ```

### Step 4: System Configuration

1. Configure the hostname:

    Replace `HOSTNAME` with the desired hostname:

    ```shell
    hostname HOSTNAME
    hostname > /mnt/etc/hostname
    nano /mnt/etc/hosts
    ```

    ```text
    Add a line:
    127.0.0.1       HOSTNAME
    or if the system has a real name in DNS:
    127.0.0.1       FQDN hostname
    ```

2. Configure the network interface:

    Find the interface name:

    ```shell
    ip addr show
    ```

    ```shell
    nano /mnt/etc/network/interfaces
    ```

    Adjust NAME below to match your interface name:

    ```text
    auto lo
    iface lo inet loopback

    auto NAME
    iface NAME inet dhcp
    ```

3. Configure the package sources:

    ```shell
    nano /mnt/etc/apt/sources.list
    ```

    <!-- markdownlint-disable MD013 -->
    ```text
    deb http://deb.debian.org/debian bookworm main contrib non-free-firmware
    deb-src http://deb.debian.org/debian bookworm main contrib non-free-firmware

    deb http://deb.debian.org/debian-security bookworm-security main contrib non-free-firmware
    deb-src http://deb.debian.org/debian-security bookworm-security main contrib non-free-firmware

    deb http://deb.debian.org/debian bookworm-updates main contrib non-free-firmware
    deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free-firmware
    ```
    <!-- markdownlint-enable MD013 -->

4. Bind the virtual filesystems from the LiveCD environment to the new system and
chroot into it:

    ```shell
    mount --make-private --rbind /dev  /mnt/dev
    mount --make-private --rbind /proc /mnt/proc
    mount --make-private --rbind /sys  /mnt/sys
    chroot /mnt /usr/bin/env DISK=$DISK bash --login
    ```

5. Configure a basic system environment:

    ```shell
    apt update

    apt install --yes console-setup \
                      locales
    ```

    Even if you prefer a non-English system language, always ensure that `en_US.UTF-8`
    is available:

    ```shell
    dpkg-reconfigure locales tzdata keyboard-configuration console-setup
    ```

6. Install ZFS in the chroot environment for the new system:

    ```shell
    apt install --yes dpkg-dev \
                      linux-headers-generic \
                      linux-image-generic

    apt install --yes zfs-initramfs

    echo REMAKE_INITRD=yes > /etc/dkms/zfs.conf
    ```

    > [!NOTE]
    > Ignore any error messages saying `ERROR: Couldn't resolve device` or
    `WARNING: Couldn't determine root device`, [cryptsetup does not support ZFS](https://bugs.launchpad.net/ubuntu/+source/cryptsetup/+bug/1612906).

7. Setup `/etc/crypttab`:

    ```shell
    apt install --yes cryptsetup \
                      cryptsetup-initramfs

    echo luks1 /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part4) \
      none luks,discard,initramfs > /etc/crypttab
    ```

    The use of `initramfs` is a work-around for [cryptsetup does not support ZFS](https://bugs.launchpad.net/ubuntu/+source/cryptsetup/+bug/1612906).

8. Install GRUB

    ```shell
    apt install dosfstools

    mkdosfs -F 32 -s 1 -n EFI ${DISK}-part2
    mkdir /boot/efi
    echo /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part2) \
      /boot/efi vfat defaults 0 0 >> /etc/fstab
    mount /boot/efi
    apt install --yes grub-efi-amd64 shim-signed
    ```

9. Install GRUB theme

    1. Download theme

        ```shell
        apt install --yes curl
        ```

        <!-- markdownlint-disable MD013 -->
        ```shell
        curl -SL "https://raw.githubusercontent.com/AdisonCavani/distro-grub-themes/master/themes/hp.tar" -o /tmp/hp.tar
        mkdir -p /boot/grub/themes/hp
        tar -C /boot/grub/themes/hp -xf /tmp/hp.tar
        rm /tmp/hp.tar
        ```
        <!-- markdownlint-enable MD013 -->

    2. Edit GRUB config

        ```shell
        nano /etc/default/grub
        # Set: GRUB_GFXMODE=1920x1080
        # Set: GRUB_THEME="/boot/grub/themes/hp/theme.txt"
        ```

    3. Update GRUB config

        ```shell
        update-grub
        ```

10. Set a root password:

    ```shell
    passwd
    ```

11. Enable importing bpool

    This ensures that `bpool` is always imported, regardless of whether `/etc/zfs/zpool.cache`
    exists, whether it is in the cachefile or not, or whether `zfs-import-scan.service`
    is enabled.

    ```shell
    echo /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part3)
    ```

    ```shell
    nano /etc/systemd/system/zfs-import-bpool.service
    ```

    ```ini
    [Unit]
    DefaultDependencies=no
    Before=zfs-import-scan.service
    Before=zfs-import-cache.service

    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/sbin/zpool import -N -o cachefile=none bpool
    # Work-around to preserve zpool cache:
    ExecStartPre=-/bin/mv /etc/zfs/zpool.cache /etc/zfs/preboot_zpool.cache
    ExecStartPost=-/bin/mv /etc/zfs/preboot_zpool.cache /etc/zfs/zpool.cache

    [Install]
    WantedBy=zfs-import.target
    systemctl enable zfs-import-bpool.service
    ```

    > [!NOTE]
    > For some disk configurations (NVMe?), this service may fail with an error
    > indicating that the `bpool` cannot be found. If this happens, add
    > `-d DISK-part3` (replace `DISK` with the correct device path) to the
    > `zpool import` command.

12. Mount a tmpfs to `/tmp`

    ```shell
    cp /usr/share/systemd/tmp.mount /etc/systemd/system/
    systemctl enable tmp.mount
    ```

13. Install SSH:

    1. Install server

        ```shell
        apt install --yes openssh-server
        ```

    2. Configure

        ```shell
        nano /etc/ssh/sshd_config
        # Set: PermitRootLogin prohibit-password
        ```

    3. Generate keys

        ```shell
        ssh-keygen -t rsa -b 4096 -C "" -f ~/.ssh/id_rsa
        ```

    4. Print private key

        ```shell
        cat ~/.ssh/id_rsa
        ```

    5. Enable SSH key for host

        ```shell
        cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
        chmod 600 ~/.ssh/authorized_keys
        ```

    6. Delete local copy of key

        ```shell
        rm ~/.ssh/id_rsa ~/.ssh/id_rsa.pub
        ```

14. Install popcon

    The `popularity-contest` package reports the list of packages install on your
    system. Showing that ZFS is popular may be helpful in terms of long-term
    attention from the distro.

    ```shell
    apt install --yes popularity-contest
    ```

    Choose Yes at the prompt.

### Step 5: GRUB Installation

1. Verify that the ZFS boot filesystem is recognized:

    ```shell
    grub-probe /boot
    ```

2. Refresh the initrd files:

    ```shell
    update-initramfs -c -k all
    ```
  
    > [!NOTE]
    > Ignore any error messages saying `ERROR: Couldn't resolve device` or
    > `WARNING: Couldn't determine root device`, [cryptsetup does not support ZFS](https://bugs.launchpad.net/ubuntu/+source/cryptsetup/+bug/1612906).

3. Workaround GRUB’s missing zpool-features support:

    ```shell
    nano /etc/default/grub
    # Set: GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/proxmox"
    ```

4. Update the boot configuration:

    ```shell
    update-grub
    ```

    > [!NOTE]
    > Ignore errors from `osprober`, if present.

5. Install the boot loader:

    ```shell
    grub-install --target=x86_64-efi --efi-directory=/boot/efi \
        --bootloader-id="Proxmox" --recheck --no-floppy
    ```

6. Install missing firmware

    ```shell
    apt install --yes firmware-amd-graphics \
                      firmware-intel-sound \
                      firmware-iwlwifi \
                      firmware-linux \
                      firmware-linux-free \
                      firmware-linux-nonfree \
                      firmware-misc-nonfree \
                      firmware-realtek \
                      firmware-sof-signed \
                      live-task-non-free-firmware-pc \
                      live-task-non-free-firmware-server
    ```

7. Enable auotmatic decryption using TPM

    ```shell
    apt install --yes clevis \
                      clevis-tpm2 \
                      clevis-luks \
                      clevis-initramfs \
                      initramfs-tools \
                      tss2 \
                      tpm2-tools
    ```

    <!-- markdownlint-disable MD013 -->
    ```shell
    clevis luks bind -d /dev/disk/by-uuid/$(blkid -s UUID -o value ${DISK}-part4) tpm2 '{"pcr_bank":"sha256"}'
    ```
    <!-- markdownlint-enable MD013 -->

    ```shell
    update-initramfs -u -k all
    ```

8. Fix filesystem mount ordering:

    We need to activate `zfs-mount-generator`. This makes systemd aware of the
    separate mountpoints, which is important for things like `/var/log` and `/var/tmp`.
    In turn, `rsyslog.service` depends on `var-log.mount` by way of `local-fs.target`
    and services using the `PrivateTmp` feature of systemd automatically use `After=var-tmp.mount`.

    ```shell
    mkdir /etc/zfs/zfs-list.cache
    touch /etc/zfs/zfs-list.cache/bpool
    touch /etc/zfs/zfs-list.cache/rpool
    zed -F &
    ```

    Verify that `zed` updated the cache by making sure these are not empty:

    ```shell
    cat /etc/zfs/zfs-list.cache/bpool
    cat /etc/zfs/zfs-list.cache/rpool
    ```

    If either is empty, force a cache update and check again:

    ```shell
    zfs set canmount=on     bpool/BOOT/proxmox
    zfs set canmount=noauto rpool/ROOT/proxmox
    ```

    If they are still empty, stop zed (as below), start zed (as above) and try again.

    Once the files have data, stop `zed`:

    ```shell
    fg
    # Press Ctrl-C.
    ```

    Fix the paths to eliminate `/mnt`:

    ```shell
    sed -Ei "s|/mnt/?|/|" /etc/zfs/zfs-list.cache/*
    ```

### Step 6: First Boot

1. Snapshot the initial installation:

    ```shell
    zfs snapshot bpool/BOOT/proxmox@debian-install
    zfs snapshot rpool/ROOT/proxmox@debian-install
    ```

    In the future, you will likely want to take snapshots before each upgrade,
    and remove old snapshots (including this one) at some point to save space.

2. Exit from the `chroot` environment back to the LiveCD environment:

    ```shell
    exit
    ```

3. Run these commands in the LiveCD environment to unmount all filesystems:

    ```shell
    mount | grep -v zfs | tac | awk '/\/mnt/ {print $3}' | \
        xargs -i{} umount -lf {}
    zpool export -a
    ```

4. If this fails for rpool, mounting it on boot will fail and you will need to
`zpool import -f rpool`, then `exit` in the initramfs prompt.
5. Reboot:

    ```shell
    reboot
    ```

    Wait for the newly installed system to boot normally. Login as root.

## Install Proxmox VE

### Step 1: Resolve Node IP Address Through `/etc/hosts` Entry

For a `/etc/hosts` record you need one of the following entries for your hostname:

- 1 IPv4 or
- 1 IPv6 or
- 1 IPv4 and 1 IPv6

While you could keep the entry that maps the `127.0.1.1` loopback address to the
hostname, as Proxmox VE's cluster system cycles through all addresses until it
finds a non-loopback one, it's recommended to remove the hostname from that record
if unsure as this avoids any ambiguity.

For instance, if your IP address is `192.168.15.77`, and your hostname prox4m1,
then your `/etc/hosts` file could look like:

```shell
nano /etc/hosts
```

```txt
127.0.0.1       localhost
192.168.15.77   FQDN hostname

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### Step 2: Adapt your sources.list

1. Add the Proxmox VE repository:

    <!-- markdownlint-disable MD013 -->
    ```shell
    echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
    ```
    <!-- markdownlint-enable MD013 -->

2. Add the Proxmox VE repository key as root (or use sudo):

    <!-- markdownlint-disable MD013 -->
    ```shell
    wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg 
    ```
    <!-- markdownlint-enable MD013 -->

3. Update your repository and system:

    ```shell
    apt update
    apt full-upgrade
    ```

### Step 3: Install the Proxmox VE Kernel

First you need to install and boot the Proxmox VE kernel, as some packages depend
on specific kernel compile flags to be set or feature extensions (e.g., for apparmor)
to be available.

```shell
apt install proxmox-default-kernel

systemctl reboot
```

### Step 4: Install the Proxmox VE packages

Install the Proxmox VE packages

```shell
apt install proxmox-ve postfix open-iscsi chrony pve-zsync
```

### Step 5: Remove the Debian Kernel

Proxmox VE ships its own kernel and keeping the Debian default kernel can lead to
trouble on upgrades, for example, with Debian point releases. Therefore, you must
remove the default Debian kernel:

```shell
apt remove linux-image-amd64 'linux-image-6.1*'
```

Update and check grub2 config by running:

```shell
update-grub
```

### Step 6: Enable zpool in proxmox

```shell
pvesm add zfspool local-zfs -pool rpool
```

### Step 7: Remove the os-prober Package

```shell
apt remove os-prober
```

### Step 8: Disable GUI on boot

```shell
systemctl set-default multi-user.target
```

### Step 7: Connect to the Proxmox VE web interface

Connect to the admin web interface [https://your-ip-address:8006](https://your-ip-address:8006).
If you have a fresh install and have not added any users yet, you should select
PAM authentication realm and login with root user account.

1. Create a Linux Bridge

    Create a Linux Bridge called vmbr0, and add your first network interface to it.

    ```shell
    nano /etc/network/interfaces
    ```

    ```text
    auto lo
    iface lo inet loopback

    iface eno1 inet manual

    auto vmbr0
    iface vmbr0 inet static
            address 192.168.10.2/24
            gateway 192.168.10.1
            bridge-ports eno1
            bridge-stp off
            bridge-fd 0
    ```

2. Upload Subscription Key

    The Proxmox VE enterprise repository is set up automatically during the
    installation as it's the recommended repository for stable, enterprise usage.

    You should upload your subscription key now in the web interface, then you
    can remove the no-subscription repository added for installation.

    ```shell
    rm /etc/apt/sources.list.d/pve-install-repo.list
    ```

3. Snapshot the installation:

    ```shell
    zfs snapshot bpool/BOOT/proxmox@initial-install
    zfs snapshot rpool/ROOT/proxmox@initial-install
    ```
