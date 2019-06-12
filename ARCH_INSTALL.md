# Installer Archlinux avec chiffrement des partitions

## Sources

- https://github.com/FredBezies/arch-tuto-installation/blob/master/install.md
- https://driikolu.fr/comment-installer-archlinux-en-chiffre/

## Processus (version rapide)

L'installation est prévu pour un ordinateur n'ayant qu'un seul SSD de 128Go, sans dual-boot et en mode UEFI. La configuration des paritions est la suivante:

| Référence  | Point de montage | Taille | Système de fichiers |
| ---------- | ---------------- | ------ | ------------------- |
| /dev/sda1  | / 	              | 40Go   | ext4                |
| /dev/sda2  | /boot/efi        |	128 Mo | fat32               |
| /dev/sda3  |                  | 8Go    | swap                |
| /dev/sda4  | /home            |	Reste  | ext4                |

```bash
# Clavier en azerty
loadkeys fr-pc # loqdkeys fr)pc

# Voir l'état des disques
fdisk -l

# Créer les partitions comme prévus (avec /dev/sda)
parted /dev/sda
mklabel gpt
mkpart primary ext4 2MiB 40G
mkpart primary fat32 40G 40.256G
mkpart primary linux-swap 40.256G 45G
mkpart primary ext4 45G 100%
q

# Chiffrer les partitions / et /home (il faut répondre "YES", puis donner une passphrase)
cryptsetup --verbose --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random luksFormat /dev/sda1
cryptsetup --verbose --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random luksFormat /dev/sda4

# Ouverture des partitions chiffrées
cryptsetup open --type luks /dev/sda1 root
cryptsetup open --type luks /dev/sda4 home

# Formatage des partitions
mkfs.ext4 /dev/mapper/root
mkfs.fat -F32 /dev/sda2
mkfs.ext4 /dev/mapper/home
mkswap /dev/sda3
swapon /dev/sda3

# Montage des partitions
mount /dev/mapper/root /mnt
mkdir /mnt/{boot,boot/efi,home}
mount /dev/sda2 /mnt/boot/efi
mount /dev/mapper/home /mnt/home

# Connexion en wifi
wifi-menu -o

# Garder les miroirs les plus proches en commentant les autres (mir.archlinux.fr / archlinux.polymorf.fr)
vim /etc/pacman.d/mirrorlist

# Installer les paquets de base
pacstrap /mnt base base-devel pacman-contrib
pacstrap /mnt zip unzip p7zip vim mc alsa-utils syslog-ng mtools dosfstools lsb-release ntfs-3g exfat-utils bash-completion tlp

# Génération du fstab qui liste les partitions présentes
genfstab -U -p /mnt >> /mnt/etc/fstab

# Installation de grub pour le boot
pacstrap /mnt grub os-prober efibootmgr

# On entre dans l'OS qu'on vient d'installer
arch-chroot /mnt

# Configuration du clavier
vim /etc/vconsole.conf 
# KEYMAP=fr-latin9
# FONT=eurlatgr

# Configuration localisation
vim /etc/locale.conf
# LANG=fr_FR.UTF-8
# LC_COLLATE=C

# Vérifier que fr_FR.UTF-8 / en_US.UTF-8 dans /etc/locale.gen n'est pas commenté
vim /etc/locale.gen # puis décommenter les bonnes lignes

# Générer les traductions
locale-gen

# Spécifier la langue de la session courante
export LANG=fr_FR.UTF-8

# Donner un nom à la machine
echo "Arch-KaYo" > /etc/hostname

# Installation du fuseau horaire
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc --utc

# Modifier /etc/default/grub pour ajouter le déchiffrement de la partition root
vim /etc/default/grub
# GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda1:root"
# GRUB_ENABLE_CRYPTODISK=y

# Ajouter les hooks keymap et encrypt dans /etc/mkinitcpio.conf
vim /etc/mkinitcpio.conf
# HOOKS="base udev autodetect modconf block filesystems keyboard fsck keymap encrypt"

# Modifier /etc/fstab pour pointer vers la version chiffré
# Il faut commenter la ligne de /dev/mapper/home et remplacer par
# /dev/mapper/home /home ext4 rw,relatime,data=ordered 0 2

# Modifier /etc/crypttab et ajouter:
# home /dev/sda4

# Génération du noyau linux
mkinitcpio -p linux

# Vérification du point de montage, et activation si besoin + installation de Grub
mount | grep efivars &> /dev/null || mount -t efivarfs efivarfs /sys/firmware/efi/efivars
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch_grub --recheck

# Petit bonus pour éviter certains bug (apparemment)
mkdir /boot/efi/EFI/boot
cp /boot/efi/EFI/arch_grub/grubx64.efi /boot/efi/EFI/boot/bootx64.efi

# Génération du fichier de config grub
grub-mkconfig -o /boot/grub/grub.cfg

# Création du mot de passe root
passwd root

# Installation de NetworkManager pour le réseau
pacman -Syy networkmanager
systemctl enable NetworkManager

# Pour aussi avoir accès à des lib 32bits comme Skype
# Décommenter la ligne:
# [multilib]
# Include = /etc/pacman.d/mirrorlist
vim /etc/pacman.conf

# On quitte et on redémarre
exit
umount -R /mnt
reboot


```
