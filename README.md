# manjaro-install

```
DISK=(/dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209)
SWAP=16
UUID=$(dd if=/dev/urandom bs=1 count=100 2>/dev/null | tr -dc 'a-z0-9' | cut -c-6)
last=$(($SWAP / ${#DISK[@]} + $SWAP % ${#DISK[@]}))

for i in ${DISK[@]}; do
    sudo sgdisk --zap-all $i
    sudo sgdisk -n1:1M:+1G -t1:EF00 $i
    sudo sgdisk -n2:0:+4G -t2:BE00 $i
    if [ $last -gt 0 ]
    then
        sudo sgdisk -n3:0:-${last}G -t3:BF00 $i
        sudo sgdisk -n4:0:0 -t4:8308 $i
    else
        sudo sgdisk -n3:0:0 -t3:BF00 $i
    fi
done

sudo zpool create -f \
-o ashift=12 \
-o autotrim=on \
-d -o feature@async_destroy=enabled \
-o feature@bookmarks=enabled \
-o feature@embedded_data=enabled \
-o feature@empty_bpobj=enabled \
-o feature@enabled_txg=enabled \
-o feature@extensible_dataset=enabled \
-o feature@filesystem_limits=enabled \
-o feature@hole_birth=enabled \
-o feature@large_blocks=enabled \
-o feature@lz4_compress=enabled \
-o feature@spacemap_histogram=enabled \
-O acltype=posixacl \
-O canmount=off \
-O compression=lz4 \
-O devices=off \
-O normalization=formD \
-O relatime=on \
-O xattr=sa \
-O mountpoint=/boot \
-R /mnt \
bpool_$UUID \
$(for i in ${DISK[@]}; do
printf "$i-part2 ";
done)

sudo zpool create -f \
-o ashift=12 \
-o autotrim=on \
-R /mnt \
-O acltype=posixacl \
-O canmount=off \
-O compression=lz4 \
-O dnodesize=auto \
-O normalization=formD \
-O relatime=on \
-O xattr=sa \
-O mountpoint=/ \
-O encryption=aes-256-gcm \
-O keyformat=passphrase \
-O keylocation=prompt \
rpool_$UUID \
$(for i in ${DISK[@]}; do
printf "$i-part3 ";
done)

sudo zfs create -o canmount=off -o mountpoint=none bpool_$UUID/BOOT
sudo zfs create -o canmount=off -o mountpoint=none rpool_$UUID/DATA
sudo zfs create -o canmount=off -o mountpoint=none rpool_$UUID/ROOT

sudo zfs create -o mountpoint=legacy -o canmount=noauto bpool_$UUID/BOOT/default
sudo zfs create -o mountpoint=/ -o canmount=off rpool_$UUID/DATA/default
sudo zfs create -o mountpoint=/ -o canmount=noauto rpool_$UUID/ROOT/default

sudo zfs mount rpool_$UUID/ROOT/default
sudo mkdir /mnt/boot
sudo mount -t zfs bpool_$UUID/BOOT/default /mnt/boot

for i in {usr,var,var/lib};
    do
        sudo zfs create -o canmount=off rpool_$UUID/DATA/default/$i
done

for i in {home,root,srv,usr/local,var/log,var/spool};
    do
        sudo zfs create -o canmount=on rpool_$UUID/DATA/default/$i
done

sudo chmod 750 /mnt/root

sudo zfs create -o canmount=on rpool_$UUID/DATA/default/var/lib/docker

sudo mkfs.vfat -n EFI /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part1
sudo mkdir -p /mnt/boot/efis/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209
sudo mount -t vfat /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part1 /mnt/boot/efis/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209

sudo mkdir -p /mnt/boot/efi
sudo mount -t vfat /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part1 /mnt/boot/efi

sudo zpool set bootfs=bpool_$UUID/BOOT/default bpool_$UUID
sudo zpool set cachefile=/etc/zfs/zpool.cache bpool_$UUID
sudo zpool set cachefile=/etc/zfs/zpool.cache rpool_$UUID

sudo pacman -Sy
git clone https://gitlab.manjaro.org/applications/manjaro-architect.git
cd manjaro-architect
git checkout 0.9.34
sudo pacman -S base-devel --noconfirm
sudo pacman -S f2fs-tools gptfdisk manjaro-architect-launcher manjaro-tools-base mhwd nilfs-utils pacman-mirrorlist parted --noconfirm
makepkg -si --noconfirm
cd ..
rm -rf manjaro-architect

setup

sudo zfs mount rpool_$UUID/ROOT/default
sudo mount -t zfs bpool_$UUID/BOOT/default /mnt/boot
sudo zfs mount -a
sudo mount /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part1 /mnt/boot/efi
sudo mount -t vfat /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part1 /mnt/boot/efis/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209

sudo sed 's/GRUB_CMDLINE_LINUX=\"/GRUB_CMDLINE_LINUX=\"zfs_import_dir=\/dev\/disk\/by-id\ /' /mnt/etc/default/grub | sudo tee /mnt/etc/default/grub.new
sudo rm /mnt/etc/default/grub
sudo mv /mnt/etc/default/grub.new /mnt/etc/default/grub

sudo zpool set bootfs=rpool_$UUID/ROOT/default rpool_$UUID
sudo zpool set cachefile=/etc/zfs/zpool.cache rpool_$UUID
sudo zpool set cachefile=/etc/zfs/zpool.cache bpool_$UUID

sudo cp /etc/zfs/zpool.cache /mnt/etc/zfs/zpool.cache

echo " " | sudo tee -a /mnt/etc/fstab
echo swap-nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209 /dev/disk/by-id/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209-part4 /dev/urandom swap,cipher=aes-cbc-essiv:sha256,size=256,discard | sudo tee -a /mnt/etc/crypttab > /dev/null
echo /dev/mapper/swap-nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209 none swap defaults 0 0 | sudo tee -a /mnt/etc/fstab > /dev/null

sudo systemctl enable zfs.target --root=/mnt
sudo systemctl enable zfs-import-cache --root=/mnt
sudo systemctl enable zfs-mount --root=/mnt
sudo systemctl enable zfs-import.target --root=/mnt

sudo manjaro-chroot /mnt zgenhostid
sudo manjaro-chroot /mnt mkinitcpio -P

sudo manjaro-chroot /mnt echo 'export ZPOOL_VDEV_NAME_PATH=YES' | sudo tee -a /etc/profile
sudo manjaro-chroot /mnt grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=manjaro --removable
sudo manjaro-chroot /mnt grub-mkconfig -o /boot/grub/grub.cfg
sudo manjaro-chroot /mnt echo 'Defaults env_keep += "ZPOOL_VDEV_NAME_PATH"' | sudo tee -a /etc/sudoers

sudo umount /mnt/boot/efi
sudo umount /mnt/boot/efis/nvme-WDC_PC_SN730_SDBQNTY-512G-1001_19385J470209
sudo umount /mnt/boot
sudo zfs umount -a
sudo zfs umount rpool_$UUID/ROOT/default
sudo zpool export rpool_$UUID
sudo zpool export bpool_$UUID

sudo reboot
```
