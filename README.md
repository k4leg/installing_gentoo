
# Table of Contents

1.  [Introduction](#orgdbaf3fe)
2.  [Installation media](#orgdd9ecb3)
3.  [Partitioning the data storage](#orgc4e673e)
    1.  [Caveats](#org420bc65)
4.  [Creating and mounting filesystems](#org930bf0a)
5.  [Installing a stage tarball](#org6bed703)
6.  [Chrooting](#org5c665af)
7.  [Configuring Portage](#org73e073c)
8.  [Configuring timezone](#org03fed77)
9.  [Configuring locale](#org655d386)
    1.  [Caveats](#org4e74fea)
10. [Installing Linux kernel](#org6565439)
11. [Installing system tools](#org0e4e61c)
12. [Configuring system](#org8867d6f)
13. [Installing boot loader](#org27e7629)
14. [Finalizing](#org6011517)



<a id="orgdbaf3fe"></a>

# Introduction

The key words &ldquo;MUST&rdquo;, &ldquo;MUST NOT&rdquo;, &ldquo;REQUIRED&rdquo;, &ldquo;SHALL&rdquo;, &ldquo;SHALL NOT&rdquo;,
&ldquo;SHOULD&rdquo;, &ldquo;SHOULD NOT&rdquo;, &ldquo;RECOMMENDED&rdquo;, &ldquo;MAY&rdquo;, and &ldquo;OPTIONAL&rdquo; in this
document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

This document describes how to install Gentoo with systemd, btrfs on
LUKS root, swap on LUKS, and systemd-boot on an amdm64 system.

This document is not a replacement for official [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page), you
SHOULD read it before your first installation.


<a id="orgdd9ecb3"></a>

# Installation media

You can use any installation media which contains the following:

-   Btrfs utilities
-   DOS filesystem tools
-   Tool to setup encrypted devices with dm-crypt

[Minimal installation CD](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation#Downloading) fits.


<a id="orgc4e673e"></a>

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
<td class="org-left"><code>/dev/swap_partition</code></td>
<td class="org-left">Any size</td>
<td class="org-left">Linux filesystem</td>
</tr>


<tr>
<td class="org-left"><code>/dev/root_partition</code></td>
<td class="org-left">Rest of data storage</td>
<td class="org-left">Linux filesystem</td>
</tr>
</tbody>
</table>

You can use `fdisk` for partitioning.


<a id="org420bc65"></a>

## Caveats

You SHOULD NOT use filesystem type like &ldquo;Linux root (x86-64)&rdquo; or
&ldquo;Linux swap&rdquo;.  There is [systemd-gpt-auto-generator](systemd-gpt-auto-generator(8)) that automatically
tries to mount that filesystems.  systemd will not find there what
expects.

If you want it anyway, then you MUST add `systemd.gpt_auto=0` to the
kernel command line parameters.


<a id="org930bf0a"></a>

# Creating and mounting filesystems

    mkfs.vfat -F 32 /dev/efi_system_partition
    cryptsetup luksFormat /dev/root_partition
    cryptsetup open /dev/root_partition cryptos
    mkfs.btrfs /dev/mapper/cryptos
    
    ROOT=/mnt/gentoo
    mount -o defaults,compress=zstd /dev/mapper/cryptos "$ROOT"
    btrfs subvolume create "${ROOT}/sv_snapshots"
    btrfs subvolume create "${ROOT}/sv_roots"
    btrfs subvolume create "${ROOT}/sv_roots/sv_gentoo"
    for i in sv_{root,srv,usr,var}; do
        btrfs subvolume create "${ROOT}/sv_roots/sv_gentoo/${i}"
    done
    btrfs subvolume create "${ROOT}/sv_roots/sv_gentoo/sv_usr/sv_local"
    for i in sv_{cache,games,lib,log,mail,spool,tmp}; do
        btrfs subvolume create "${ROOT}/sv_roots/sv_gentoo/sv_var/${i}"
    done
    for i in sv_{AccountsService,NetworkManager,docker,libvirt,machines,portables}
    do
        btrfs subvolume create \
    	"${ROOT}/sv_roots/sv_gentoo/sv_var/sv_lib/${i}"
    done
    for i in sv_userdata{,/sv_{home,root}}; do
        btrfs subvolume create "${ROOT}/${i}"
    done
    umount /mnt/gentoo
    
    OPTS=defaults,compress=zstd
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_root" \
        /dev/mapper/cryptos "$ROOT"
    mkdir -p \
        "$ROOT"/{.btrfs/snapshots,boot,home,root,srv,usr/local,var} \
        "$ROOT"/var/{cache,games,lib,log,mail,spool,tmp} \
        "$ROOT"/var/lib/{AccountsService,NetworkManager,docker,libvirt,machines,portables}
    mount /dev/efi_system_partition "${ROOT}/boot"
    mount -o "${OPTS},subvol=/sv_snapshots" /dev/mapper/cryptos \
        "${ROOT}/.btrfs/snapshots"
    mount -o "${OPTS},subvol=/sv_userdata/sv_home" \
        /dev/mapper/cryptos "${ROOT}/home"
    mount -o "${OPTS},subvol=/sv_userdata/sv_root" \
        /dev/mapper/cryptos "${ROOT}/root"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_srv" \
        /dev/mapper/cryptos "${ROOT}/srv"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_usr/sv_local" \
        /dev/mapper/cryptos "${ROOT}/usr/local"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_cache" \
        /dev/mapper/cryptos "${ROOT}/var/cache"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_games" \
        /dev/mapper/cryptos "${ROOT}/var/games"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_log" \
        /dev/mapper/cryptos "${ROOT}/var/log"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_mail" \
        /dev/mapper/cryptos "${ROOT}/var/mail"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_spool" \
        /dev/mapper/cryptos "${ROOT}/var/spool"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_tmp" \
        /dev/mapper/cryptos "${ROOT}/var/tmp"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_AccountsService" \
        /dev/mapper/cryptos "${ROOT}/var/lib/AccountsService"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_NetworkManager" \
        /dev/mapper/cryptos "${ROOT}/var/lib/NetworkManager"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_docker" \
        /dev/mapper/cryptos "${ROOT}/var/lib/docker"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_libvirt" \
        /dev/mapper/cryptos "${ROOT}/var/lib/libvirt"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_machines" \
        /dev/mapper/cryptos "${ROOT}/var/lib/machines"
    mount -o "${OPTS},subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_portables" \
        /dev/mapper/cryptos "${ROOT}/var/lib/portables"
    mkdir -p \
        "$ROOT"/.btrfs/snapshots/{boot,home,root,srv,usr{,/local},var} \
        "$ROOT"/.btrfs/snapshots/var/{cache,games,lib,log,mail,spool,tmp} \
        "$ROOT"/.btrfs/snapshots/var/lib/{AccountsService,NetworkManager,docker,libvirt,machines,portables}

Note that subvolumes for `/srv`, `/var/lib/machines`, and
`/var/lib/portables` wanted by systemd<sup><a id="fnr.1" class="footref" href="#fn.1" role="doc-backlink">1</a></sup>.  To view all datasets
that systemd wants:

    grep '^[vqQ]' /usr/lib/tmpfiles.d/*


<a id="org6bed703"></a>

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


<a id="org5c665af"></a>

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


<a id="org73e073c"></a>

# Configuring Portage

    emerge-webrsync
    eselect news list
    eselect news read
    emerge -avuDN @world
    mkdir /etc/portage/{package.{env,license},env}


<a id="org03fed77"></a>

# Configuring timezone

    ln -sfr /usr/share/zoneinfo/Region/City /etc/localtime


<a id="org655d386"></a>

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


<a id="org4e74fea"></a>

## Caveats

You SHOULD use &ldquo;C.utf8&rdquo; locale for `LC_COLLATE` environment.


<a id="org6565439"></a>

# Installing Linux kernel

    echo 'sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE' \
        >/etc/portage/package.license/10-linux-firmware
    emerge -av sys-kernel/linux-firmware
    emerge -av sys-firmware/intel-microcode  # For Intel CPUs.
    emerge -av sys-kernel/installkernel-systemd-boot
    emerge -av sys-kernel/gentoo-kernel-bin


<a id="org0e4e61c"></a>

# Installing system tools

Installing filesystem tools:

    emerge -av sys-fs/{btrfs-progs,cryptsetup,dosfstools}

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


<a id="org8867d6f"></a>

# Configuring system

    echo "$hostname" >/etc/hostname
    passwd
    systemd-firstboot --prompt --setup-machine-id
    systemctl preset-all

`/etc/crypttab` example (`discard` option for devices that support
trim):

    cryptswap	/dev/disk/by-partuuid/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX	/dev/urandom	swap,discard

`/etc/fstab` example:

    /dev/mapper/cryptos     /                               btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_root                               0       0
    /dev/mapper/cryptos     /srv                            btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_srv                                0       0
    /dev/mapper/cryptos     /usr/local                      btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_usr/sv_local                       0       0
    /dev/mapper/cryptos     /var/cache                      btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_cache                       0       0
    /dev/mapper/cryptos     /var/games                      btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_games                       0       0
    /dev/mapper/cryptos     /var/log                        btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_log                         0       0
    /dev/mapper/cryptos     /var/mail                       btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_mail                        0       0
    /dev/mapper/cryptos     /var/spool                      btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_spool                       0       0
    /dev/mapper/cryptos     /var/tmp                        btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_tmp                         0       0
    /dev/mapper/cryptos     /var/lib/AccountsService        btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_AccountsService      0       0
    /dev/mapper/cryptos     /var/lib/NetworkManager         btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_NetworkManager       0       0
    /dev/mapper/cryptos     /var/lib/docker                 btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_docker               0       0
    /dev/mapper/cryptos     /var/lib/libvirt                btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_libvirt              0       0
    /dev/mapper/cryptos     /var/lib/machines               btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_machines             0       0
    /dev/mapper/cryptos     /var/lib/portables              btrfs   defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_var/sv_lib/sv_portables            0       0
    /dev/mapper/cryptos     /home                           btrfs   defaults,compress=zstd,subvol=/sv_userdata/sv_home                                      0       0
    /dev/mapper/cryptos     /root                           btrfs   defaults,compress=zstd,subvol=/sv_userdata/sv_root                                      0       0
    /dev/mapper/cryptos     /.btrfs/snapshots               btrfs   defaults,compress=zstd,subvol=/sv_snapshots                                             0       0
    UUID=XXXX-XXXX          /boot                           vfat    defaults                                                                                0       2
    /dev/mapper/cryptswap   none                            swap    sw,discard                                                                              0       0


<a id="org27e7629"></a>

# Installing boot loader

Installing systemd-boot (and systemd cryptsetup):

    echo 'sys-apps/systemd cryptsetup gnuefi' \
        >/etc/portage/package.use/10-systemd
    emerge -avDU @world
    bootctl install

`/etc/kernel/cmdline` example:

    rd.luks.name="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX=cryptos" rd.luks.options=discard root=/dev/mapper/cryptos rootfstype=btrfs rootflags="defaults,compress=zstd,subvol=/sv_roots/sv_gentoo/sv_root" splash quiet ro

where `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX` is UUID
`/dev/root_partition`.

Setup initramfs:

    mkdir /etc/dracut.conf.d

`/etc/dracut.conf.d/compress.conf` example:

    compress="zstd"

Reconfigure kernel:

    emerge --config "$kernel_atom"


<a id="org6011517"></a>

# Finalizing

Exit the chrooted environment, unmount all mounted partitions, and
reboot:

    exit
    cd
    umount -l /mnt/gentoo/dev{/shm,/pts,}
    umount -R /mnt/gentoo
    cryptsetup close cryptos
    reboot

Enable and setup services:

    systemctl enable iwd.service  # For Wi-Fi.
    systemctl enable fstrim.timer  # For devices that support trim.
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

<sup><a id="fn.1" href="#fnr.1">1</a></sup> See [1](https://github.com/systemd/systemd/blob/822cd601357f6f45d0176ae38fe9f86077462f06/tmpfiles.d/home.conf#L11), [2](https://github.com/systemd/systemd/blob/822cd601357f6f45d0176ae38fe9f86077462f06/tmpfiles.d/systemd-nspawn.conf#L10), and [3](https://github.com/systemd/systemd/blob/61d0578b07b97cbffebfd350bac481274e310d39/tmpfiles.d/portables.conf#L4).
