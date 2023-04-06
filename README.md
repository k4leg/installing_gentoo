
# Table of Contents

1.  [Introduction](#orgf1cdd91)
2.  [Installation media](#org78d7c2b)
3.  [Partitioning the data storage](#orgdbd08f9)
    1.  [Caveats](#orgcea2ea8)
4.  [ZFS caveats](#orgd33baf8)
    1.  [`ashift` property](#org6e9675f)
    2.  [Path to partition](#org2862182)
    3.  [Swap](#org86d6099)
    4.  [Root dataset encryption](#org83d17ef)
5.  [Creating and mounting filesystems](#orgce9bada)
6.  [Installing a stage tarball](#org3dad6f8)
7.  [Chrooting](#org4b6d9b1)
8.  [Configuring Portage](#org99539f5)
9.  [Configuring timezone](#orgdbadafc)
10. [Configuring locale](#org43d0272)
    1.  [Caveats](#org7e85319)
11. [Installing Linux kernel](#org29bb550)
12. [Installing system tools](#orgb8a1bbb)
13. [Configuring system](#orgf11e3c5)
14. [Installing boot loader](#org999fe8e)
15. [Finalizing](#org73eb9dd)



<a id="orgf1cdd91"></a>

# Introduction

The key words &ldquo;MUST&rdquo;, &ldquo;MUST NOT&rdquo;, &ldquo;REQUIRED&rdquo;, &ldquo;SHALL&rdquo;, &ldquo;SHALL NOT&rdquo;,
&ldquo;SHOULD&rdquo;, &ldquo;SHOULD NOT&rdquo;, &ldquo;RECOMMENDED&rdquo;, &ldquo;MAY&rdquo;, and &ldquo;OPTIONAL&rdquo; in this
document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

This document describes how to install Gentoo with systemd, ZFS with
native encryption on root, swap on zvol, and systemd-boot on an amd64
system.

This document is not a replacement for official [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page), you
SHOULD read it before your first installation.


<a id="org78d7c2b"></a>

# Installation media

You can use any installation media which contains the following:

-   Userland utilities for ZFS
-   DOS filesystem tools
-   fio (for `ashift` property test)

[Gentoo Admin CD](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation#Downloading) fits.


<a id="orgdbd08f9"></a>

# Partitioning the data storage

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Partition</th>
<th scope="col" class="org-left">Size</th>
<th scope="col" class="org-left">Type</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left"><code>/dev/efi_system_partition</code></td>
<td class="org-left">256&#x2013;1024 MiB</td>
<td class="org-left">EFI System</td>
</tr>


<tr>
<td class="org-left"><code>/dev/root_partition</code></td>
<td class="org-left">Rest of data storage</td>
<td class="org-left">Solaris root</td>
</tr>
</tbody>
</table>

You can use `fdisk` for partitioning.


<a id="orgcea2ea8"></a>

## Caveats

You SHOULD NOT use filesystem type like &ldquo;Linux root (x86-64)&rdquo;.  There
is [systemd-gpt-auto-generator](systemd-gpt-auto-generator(8)) that automatically tries to mount that
filesystems.  systemd will not find there what expects.

If you want it anyway, then add `systemd.gpt_auto=0` to the kernel
command line parameters.


<a id="orgd33baf8"></a>

# ZFS caveats

You SHOULD see [OpenZFS FAQ](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/FAQ.html) and [OpenZFS issues](https://github.com/openzfs/zfs/issues).


<a id="org6e9675f"></a>

## `ashift` property

To test for optimal `ashift` property run the following command for
each value of `ashift`, and look for the lowest latency values<sup><a id="fnr.1" class="footref" href="#fn.1" role="doc-backlink">1</a></sup>:

    fio \
        --name=ashift_test \
        --rw=write \
        --bs=64K \
        --fsync=1 \
        --size=1G \
        --numjobs=1 \
        --iodepth=1


<a id="org2862182"></a>

## Path to partition

Instead of using `/dev` you SHOULD use `/dev/disk/by-id`<sup><a id="fnr.2" class="footref" href="#fn.2" role="doc-backlink">2</a></sup>.  To
find your data storage in `/dev/disk/by-id` you can use the following
command:

    find -L /dev/disk/by-id -samefile /dev/dspart3


<a id="org86d6099"></a>

## Swap

Swap on zvol may lead to deadlock<sup><a id="fnr.3" class="footref" href="#fn.3" role="doc-backlink">3</a></sup>.


<a id="org83d17ef"></a>

## Root dataset encryption

You SHOULD NOT add `encryption` property to root dataset<sup><a id="fnr.4" class="footref" href="#fn.4" role="doc-backlink">4</a></sup>.


<a id="orgce9bada"></a>

# Creating and mounting filesystems

    mkfs.vfat -F 32 /dev/efi_system_partition
    modprobe zfs
    zpool create \
        -o ashift="$ashift" \
        -O dnodesize=auto \
        -O acltype=posix \
        -O xattr=sa \
        -O normalization=formD \
        -O relatime=on \
        -O mountpoint=none \
        -O canmount=off \
        -O compression=on \
        -R /mnt/gentoo \
        rpool /dev/root_partition
    zfs create \
        -o mountpoint=none \
        -o canmount=off \
        -o encryption=on \
        -o keyformat=passphrase \
        -o keylocation=prompt \
        rpool/enc
    zfs create -o mountpoint=none -o canmount=off rpool/enc/root
    zfs create -o mountpoint=/ -o canmount=noauto rpool/enc/root/gentoo
    zpool set bootfs=rpool/enc/root/gentoo rpool
    zfs mount rpool/enc/root/gentoo
    mkdir /mnt/gentoo/boot
    mount /dev/efi_system_partition /mnt/gentoo/boot
    zfs create -o canmount=off rpool/enc/root/gentoo/usr
    zfs create -o canmount=off rpool/enc/root/gentoo/var
    zfs create -o canmount=off rpool/enc/root/gentoo/var/lib
    zfs create rpool/enc/root/gentoo/srv
    zfs create rpool/enc/root/gentoo/usr/local
    zfs create rpool/enc/root/gentoo/var/cache
    zfs create rpool/enc/root/gentoo/var/games
    zfs create rpool/enc/root/gentoo/var/lib/AccountsService
    zfs create rpool/enc/root/gentoo/var/lib/NetworkManager
    zfs create rpool/enc/root/gentoo/var/lib/docker
    zfs create rpool/enc/root/gentoo/var/lib/machines
    zfs create rpool/enc/root/gentoo/var/lib/portables
    zfs create rpool/enc/root/gentoo/var/log
    zfs create rpool/enc/root/gentoo/var/mail
    zfs create rpool/enc/root/gentoo/var/spool
    zfs create rpool/enc/root/gentoo/var/tmp
    zfs create -o mountpoint=/home rpool/enc/home
    zfs create -o mountpoint=/root rpool/enc/home/root
    zfs create \
        -o logbias=throughput \
        -o sync=always \
        -o primarycache=metadata \
        -o secondarycache=none \
        -o compression=zle \
        -b "$(getconf PAGESIZE)" \
        -V "$swap_size" \
        rpool/enc/swap
    mkswap /dev/zvol/rpool/enc/swap

Note that datasets for `/srv`, `/var/lib/machines`, and
`/var/lib/portables` wanted by systemd<sup><a id="fnr.5" class="footref" href="#fn.5" role="doc-backlink">5</a></sup>.  To view all datasets
that systemd wants:

    grep '^[vqQ]' /usr/lib/tmpfiles.d/*


<a id="org3dad6f8"></a>

# Installing a stage tarball

    ntpd -qg
    cd /mnt/gentoo
    wget "$stage_file"{,.asc,.sha256}
    gpg --import /usr/share/openpgp-keys/gentoo-release.asc
    gpg --verify *.asc
    gpg --verify *.sha256
    chksum="$(sha256sum *.tar.xz | cut -d' ' -f1)"
    grep "$chksum" *.sha256
    tar xpvf *.tar.xz --xattrs-include='*.*' --numeric-owner
    echo $?  # Verify that tar unpack archive successfully.


<a id="org4b6d9b1"></a>

# Chrooting

    mirrorselect -io >>/mnt/gentoo/etc/portage/make.conf
    mkdir /mnt/gentoo/etc/portage/repos.conf
    cp /mnt/gentoo/usr/share/portage/config/repos.conf \
        /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
    cp -L /etc/resolv.conf /mnt/gentoo/etc
    mount -t proc /proc /mnt/gentoo/proc
    mount -R /sys /mnt/gentoo/sys
    mount --make-rslave /mnt/gentoo/sys
    mount -R /dev /mnt/gentoo/dev
    mount --make-rslave /mnt/gentoo/dev
    mount -B /run /mnt/gentoo/run
    mount --make-slave /mnt/gentoo/run
    chroot /mnt/gentoo /bin/bash
    source /etc/profile
    export PS1="(chroot) $PS1"


<a id="org99539f5"></a>

# Configuring Portage

    emerge-webrsync
    emerge --sync
    eselect news list
    eselect news read
    eselect profile list
    eselect profile set "$profile"
    emerge -avuDN @world
    mkdir /etc/portage/{package.{env,license},env}


<a id="orgdbadafc"></a>

# Configuring timezone

    ln -sfr /usr/share/zoneinfo/Region/City /etc/localtime


<a id="org43d0272"></a>

# Configuring locale

`/etc/locale.gen` example:

    en_US.UTF-8 UTF-8

Generate locales:

    locale-gen

Select locale:

    eselect locale list
    eselect locale set "$locale"

`/etc/locale.conf` example:

    LANG="en_US.utf8"
    LC_COLLATE="C.utf8"

Reload the environment:

    env-update
    source /etc/profile
    export PS1="(chroot) $PS1"


<a id="org7e85319"></a>

## Caveats

You SHOULD use &ldquo;C.utf8&rdquo; locale for `LC_COLLATE` environment.


<a id="org29bb550"></a>

# Installing Linux kernel

    echo 'sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE' \
        >/etc/portage/package.license/10-linux-firmware
    emerge -av sys-kernel/linux-firmware
    emerge -av sys-firmware/intel-microcode  # For Intel CPUs.
    emerge -av sys-kernel/installkernel-systemd-boot
    emerge -av sys-kernel/gentoo-kernel-bin


<a id="orgb8a1bbb"></a>

# Installing system tools

Installing filesystem tools:

    echo 'USE="dist-kernel"' >>/etc/portage/make.conf
    emerge -av sys-fs/{dosfstools,zfs}
    zgenhostid

Installing network tools (e. g. use iwd with systemd-networkd):

    emerge -av net-wireless/iwd

`/etc/systemd/network/25-wireless.network` example:

    [Match]
    Name=wlan0
    
    [Network]
    DHCP=yes
    IgnoreCarrierLoss=3s
    
    [DHCPv4]
    RouteMetric=20
    
    [IPv6AcceptRA]
    RouteMetric=20

`/etc/systemd/network/20-wired.network` example:

    [Match]
    Name=enp0s3
    
    [Network]
    DHCP=yes
    
    [DHCPv4]
    RouteMetric=10
    
    [IPv6AcceptRA]
    RouteMetric=10


<a id="orgf11e3c5"></a>

# Configuring system

    echo "$hostname" >/etc/hostname
    passwd
    systemd-firstboot --prompt --setup-machine-id
    systemctl preset-all

`/etc/fstab` example (`discard` option for SSD):

    UUID=XXXX-XXXX            /boot  vfat  defaults             0  2
    /dev/zvol/rpool/enc/swap  none   swap  defaults,sw,discard  0  0


<a id="org999fe8e"></a>

# Installing boot loader

Installing systemd-boot:

    echo 'sys-apps/systemd gnuefi' >/etc/portage/package.use/10-systemd
    emerge -avDU @world
    bootctl install

`/etc/kernel/cmdline` example:

    root=zfs:rpool/enc/root/gentoo splash quiet ro

Setup `rpool` pool mounting:

    1  mkdir /etc/zfs/zfs-list.cache
    2  touch /etc/zfs/zfs-list.cache/rpool
    3  zed -F &
    4  kill -3 %

If `/etc/zfs/zfs-list.cache/rpool` non-empty, then re-run from
beginning of the 3 line.

Fix paths to eliminate `/mnt/gentoo`:

    sed -Ei 's|/mnt/gentoo/?|/|' /etc/zfs/zfs-list.cache/rpool

Setup initramfs:

    mkdir /etc/dracut.conf.d

`/etc/dracut.conf.d/compress.conf` example:

    compress="lz4"

Reconfigure kernel:

    emerge --config "$kernel_atom"


<a id="org73eb9dd"></a>

# Finalizing

Exit the chrooted environment, unmount all mounted partitions, and
reboot:

    exit
    cd
    umount -l /mnt/gentoo/dev{/shm,/pts,}
    umount /mnt/gentoo/boot
    mount \
        | grep -v zfs \
        | tac \
        | awk '/\/mnt\/gentoo/ { print $3 }' \
        | xargs umount -l
    zpool export -a
    reboot

Enable and setup services:

    systemctl enable iwd.service  # For Wi-Fi.
    systemctl enable fstrim.timer  # For SSD.
    ln -sfr /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

Creating a user:

    useradd -mG wheel,users "$user"
    passwd "$user"

Giving a power to user:

    emerge -av app-admin/sudo
    sed -i '/^#%wheel ALL=(ALL:ALL) ALL$/ s/#//' /etc/sudoers
    cat >/etc/polkit-1/rules.d/10-admin.rules <<-EOF
    	polkit.addAdminRule(function(action, subject) {
    	    return ["unix-group:wheel"];
    	});
    EOF

Removing tarball files:

    rm /*.tar.xz*


# Footnotes

<sup><a id="fn.1" href="#fnr.1">1</a></sup> Taken from [this](https://www.reddit.com/r/zfs/comments/112v7n9/comment/j8nxbru) comment.

<sup><a id="fn.2" href="#fnr.2">2</a></sup> Taken from [this](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/FAQ.html#selecting-dev-names-when-creating-a-pool-linux) document.

<sup><a id="fn.3" href="#fnr.3">3</a></sup> Taken from [this](https://github.com/openzfs/zfs/issues/7734) issue.

<sup><a id="fn.4" href="#fnr.4">4</a></sup> See [this](https://www.reddit.com/r/zfs/comments/bnvdco/zol_080_encryption_dont_encrypt_the_pool_root) for more information.

<sup><a id="fn.5" href="#fnr.5">5</a></sup> See [1](https://github.com/systemd/systemd/blob/822cd601357f6f45d0176ae38fe9f86077462f06/tmpfiles.d/home.conf#L11), [2](https://github.com/systemd/systemd/blob/822cd601357f6f45d0176ae38fe9f86077462f06/tmpfiles.d/systemd-nspawn.conf#L10), and [3](https://github.com/systemd/systemd/blob/61d0578b07b97cbffebfd350bac481274e310d39/tmpfiles.d/portables.conf#L4).
