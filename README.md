# Arch Linux Installation with NVIDIA Drivers and Full Disk Encryption Guide

This guide offers step-by-step instructions for installing Arch Linux with NVIDIA drivers and full disk encryption. It assumes a basic familiarity with the Arch installer tools and can be adapted for Arch-derived distributions like Artix.

## Pre-Installation Steps

### 1. Initialize and Populate Keys

```bash
pacman-key --init
pacman-key --populate
pacman-key --refresh-keys
```

### 2. Identify the Drive

```bash
fdisk -l
```

### 3. Optional: Write Random Data

```bash
dd if=/dev/urandom of=/dev/sdX
```

### 4. Partition the Disk

```bash
cfdisk /dev/sdX
```

- For UEFI, create an EFI System Partition (sdX1) and root partition (sdX2). UEFI requires a [GUID Partition Table](https://wiki.archlinux.org/index.php/GUID_Partition_Table).
- For legacy mode, create the root partition (sdX1) and set it as bootable.

### 5. Format the Root Partition using LUKS

```bash
cryptsetup luksFormat --type luks1 /dev/sdXY
```

### 6. Open the LUKS Partition

```bash
cryptsetup open /dev/sdXY root
```

### 7. Install the Filesystem on the Encrypted Root Partition

```bash
mkfs.ext4 /dev/mapper/root
```

- For UEFI, format the UEFI boot partition:

```bash
mkfs.vfat -F32 /dev/sdX1
fatlabel /dev/sdX1 ESP
```

### 8. Mount the Root Filesystem

```bash
mount /dev/mapper/root /mnt
```

- For UEFI, create `/mnt/boot/efi` and mount the ESP partition:

```bash
mkdir /mnt/boot/efi
mount /dev/sdX1 /mnt/boot/efi
```

### 9. Install the Base System

```bash
pacstrap /mnt base linux linux-firmware linux-headers intel-ucode
```

- Additional kernels like `linux-lts` or `linux-zen` can be added. For AMD CPUs, use `amd-ucode`.

### 10. Install NVIDIA Drivers

```bash
pacstrap /mnt nvidia-dkms nvidia-settings nvidia-utils
```

### 11. Install Essential Packages

```bash
pacstrap /mnt grub sudo nano
```

### 12. Generate and Verify `fstab`

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

- For SSDs, consider adding the `discard` option for TRIM support:

```plaintext
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx / ext4 rw,relatime,discard 0 1
```

## Post-Installation Configuration

### 13. Chroot into the New System and Configure Basic Settings

```bash
arch-chroot /mnt
echo KEYMAP=us > /etc/vconsole.conf
ln -sf /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
hwclock --systohc
```

- Install `cryptsetup`:

```bash
pacman -S cryptsetup
```

- Uncomment desired locales:

```bash
nano /etc/locale.gen
```

- Generate locales:

```bash
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

- Set up hostname:

```bash
echo my-hostname > /etc/hostname
```

### 14. Get UUID of Root Partition

```bash
blkid
```

### 15. Create the Keyfile

```bash
dd bs=512 count=4 if=/dev/urandom of=/crypto_keyfile.bin
cryptsetup luksAddKey /dev/disk/by-uuid/{uuid of sdXY} /crypto_keyfile.bin
chmod 000 /crypto_keyfile.bin
```

### 16. Configure `mkinitcpio`

```bash
nano /etc/mkinitcpio.conf
```

- Add necessary modules, files, and hooks:

```bash
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
FILES=(/crypto_keyfile.bin)
HOOKS=(encrypt)
```

### 17. Generate Initial Ramdisk Environment

```bash
mkinitcpio -P
```

### 18. Install `efibootmgr` and `os-prober`

```bash
pacman -S efibootmgr os-prober
```

### 19. Enable `cryptdisk` Support in GRUB

```bash
nano /etc/default/grub
```

- Add the following line (replace `{uuid of sdXY}`):

```bash
GRUB_CMDLINE_LINUX="cryptdevice=UUID={uuid of sdXY}:cryptroot"
```

- If using the `discard` option:

```bash
GRUB_CMDLINE_LINUX="cryptdevice=UUID={uuid of sdXY}:cryptroot:allow-discards"
```

- Uncomment:

```bash
GRUB_ENABLE_CRYPTODISK=y
```

- For NVIDIA drivers:

```bash
GRUB_CMDLINE_LINUX="cryptdevice=UUID={uuid of sdXY}:cryptroot nvidia_drm.modeset=1 rd.driver.blacklist=nouveau modprob.blacklist=nouveau"
```

### 20. Install GRUB

- For legacy setup:

```bash
grub-install /dev/sdX --recheck
```

- For UEFI setup:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub /dev/sdX --recheck
```

### 21. Generate `grub.cfg`

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

### 22. Create Users and Install Additional Packages

```bash
useradd -m foo
passwd foo
usermod -aG wheel foo
nano /etc/sudoers
```

- Uncomment the line for wheel group sudo privileges:

```plaintext
%wheel ALL=(ALL) ALL
```

```bash
pacman -S dhcpcd iwd bluez connman
systemctl enable dhcpcd
```

### 23. Exit Chroot and Reboot

```bash
exit
reboot now
```
