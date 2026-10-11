# NixOS ZFS Tips

### Installing NixOS with encrypted ZFS root

First see https://nixos.wiki/wiki/ZFS -- which is probably better / more up to date than below

Some other examples:
* https://notthebe.ee/blog/nixos-ephemeral-zfs-root/
* https://gist.github.com/byrongibson/1578914d03a5c0a01a13f9ec53ee0b0a
* https://gist.github.com/whimbree/c13dd799693851de98936c3105111403
* https://git.thomasave.be/thomasave/dotfiles/src/branch/master/README.md

```sh
#!/bin/sh

# FIRST STOP THE zfs-zed SERVICE
systemctl stop zfs-zed

# FORCE UNLOAD ZFS KERNEL MODULES
lsmod | grep zfs | cut -d' ' -f1 | xargs rmmod -f

# NOW ADD THE FOLLOWING TO /etc/nixos/configuration.nix
#
#   boot.supportedFilesystems = [ "zfs" ];
#   boot.zfs.enableUnstable = true;
#

# AND REBUILD
nixos-rebuild switch --upgrade

# PARTITON DISK: 1 512MB EFI & REST ZFS
parted --script /dev/nvme0n1 -- \
       mklabel gpt \
       mkpart esp fat32 1MiB 512MiB \
       mkpart primary 512MiB 100% \
       set 1 boot on

# CREATE AN ENCRYPTED ZFS POOL
zpool create -f \
      -o ashift=12 \
      -O encryption=on \
      -O keyformat=passphrase \
      -O mountpoint=none \
      rpool \
      /dev/nvme0n1p2

# CREATE A SWAP PARTITION
zfs create \
    -V 4G \
    -b $(getconf PAGESIZE) \
    -o compression=zle \
    -o logbias=throughput \
    -o sync=always \
    -o primarycache=metadata \
    -o secondarycache=none \
    -o com.sun:auto-snapshot=false \
    rpool/swap
mkswap -f /dev/zvol/rpool/swap
swapon /dev/zvol/rpool/swap

# CREATE A ROOT PARTITION
zfs create \
    -o mountpoint=legacy \
    rpool/root
mkdir -p /mnt
mount -t zfs rpool/root /mnt

# CREATE A NIX PARTITION (adding this step is untested)
#
# The Nix store holds millions of small files that are written once, never
# modified, and can always be rebuilt or re-downloaded.
# It also means `zfs list` can easily show size of nix store.
#
# Options (according to AI recommendations):
#
#   mountpoint=legacy  Mounted by NixOS via fileSystems."/nix", like root.
#   atime=off          Nix never reads access times (GC uses roots, not atime),
#                      so recording them only turns every read into a write.
#   compression=zstd   Store contents are mostly text and binaries, which
#                      typically compress 1.5-2x. Use lz4 if CPU is tight.
#   xattr=sa           Store extended attributes in the inode instead of
#                      hidden directories; less I/O with this many files.
#   dnodesize=auto     Gives xattr=sa room in the dnode.
#   com.sun:auto-snapshot=false
#                      Snapshots would keep garbage-collected paths alive,
#                      so nix-collect-garbage would free no space. The store
#                      is reproducible, so it doesn't need snapshots.
#
# Deliberately left at defaults:
#   dedup=off          Use nix.settings.auto-optimise-store instead; it
#                      hard-links identical files without ZFS dedup's RAM cost.
#   sync=standard      sync=disabled can leave paths the Nix database marks
#                      valid but whose contents were never written to disk.
#   recordsize=128K    Most store files are smaller than one record anyway.
#
# Compression performance data:
# https://github.com/openzfs/zfs/pull/9735#issuecomment-570082078
zfs create \
    -o mountpoint=legacy \
    -o atime=off \
    -o compression=zstd \
    -o xattr=sa \
    -o dnodesize=auto \
    -o com.sun:auto-snapshot=false \
    rpool/nix
mkdir -p /mnt/nix
mount -t zfs rpool/nix /mnt/nix

# CREATE A HOME PARTITION
zfs create \
    -o mountpoint=legacy \
    -o compression=on \
    rpool/home
mkdir -p /mnt/home
mount -t zfs rpool/home /mnt/home

# CREATE A BOOT PARTITON
mkfs.fat -F 32 -n BOOT /dev/nvme0n1p1
mkdir -p /mnt/boot
mount -t vfat /dev/nvme0n1p1 /mnt/boot

# NOW GENERATE NIXOS CONFIG FOR /mnt
nixos-generate-config --root /mnt

# NOW ADD THE FOLLOWING TO /mnt/etc/nixos/configuration.nix
#
#   boot.initrd.supportedFilesystems = [ "zfs" ];
#   boot.supportedFilesystems = [ "zfs" ];
#   boot.zfs.enableUnstable = true;
#   services.zfs.autoScrub.enable = true;

#   # For https://nixos.wiki/wiki/Storage_optimization one of
#   nix.settings.auto-optimise-store = true; (image build time)
#    XOR
#   nix.optimise.automatic = true;     (3:45am)

# 
#   networking.hostName = "pants";
#   networking.hostId = "abcdef01";
# 

# NOW INSTALL NIXOS
nixos-install

# NOW CLEANUP & REBOOT
umount /mnt/{home,boot,nix}
umount /mnt
swapoff -a
zpool export -a
reboot
```

[Source](https://gist.github.com/dysinger/a0031aca70f9dc8df989010c88fc9c27/18a96fe6c31162f04965e9039ac765266395af39)

### What ashift should I use?

> This issue is rare on hard disk drives, and so it might not be necessary to alter the ashift value. However, there are reports that SSDs can benefit from setting the ashift property explicitly to match the 4096 byte sector size.
> The value of ashift is actually a bit shift value, so the ashift value for 512 bytes is 9 (29 = 512) while the ashift value for 4,096 bytes is 12 (212 = 4,096). To force the pool to use 4,096 byte sectors at pool creation time:
>
> ```
> $ sudo zpool create -o ashift=12 tank mirror sda sdb
> ```
> 
> To force the pool to use 4,096 byte sectors when adding a vdev to a pool:
>
> ```
> $ sudo zpool add -o ashift=12 tank mirror sdc sdd
> ```
> 
> Note: Do not set the ashift property to a value that will assign a sector size larger than the device physical block size. The logical and physical block sizes can be read from sysfs:
>
> ```
> cat /sys/class/block/<dev>/queue/logical_block_size
> cat /sys/class/block/<dev>/queue/physical_block_size
> ```

[Source](https://wiki.lustre.org/Optimizing_Performance_of_SSDs_and_Advanced_Format_Drives)

But apparently it lies somtimes?
[thread1](https://forum.level1techs.com/t/samsung-870-evo-sector-size-ashift-for-zfs/179973)
[thread2](https://github.com/openzfs/zfs/issues/6373)
