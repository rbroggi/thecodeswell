---
title: "Too Many Kernels, Not Enough /boot: My Hilarious Journey Through a Linux Kernel Panic"
date: 2025-05-17
description: "A humorous and detailed account of diagnosing and fixing a Linux kernel panic triggered by a full /boot partition. Follow the journey through LUKS challenges, partition management, and the tools like Ventoy and BorgBackup that saved the day on an Arch Linux system."
tags:
  - "Linux"
  - "Kernel Panic"
  - "/boot"
  - "Arch Linux"
  - "GRUB"
  - "LUKS"
  - "Troubleshooting"
  - "System Rescue"
  - "Ventoy"
  - "BorgBackup"
  - "Disk Partitioning"
categories:
  - "Tech Story"
  - "Linux"
author: "Rodrigo Broggi" # Or your actual name/handle
toc: true # Set to false if you don't want a Table of Contents
draft: false # Set to true if it's not ready to be published
# You can add other Hextra-specific or Hugo fields here if needed,
# for example, a path to a featured image:
# image: "/images/my-kernel-panic-adventure.jpg" 
---

Alright, buckle up, buttercups, because I've got a tale of digital woe, unexpected heroism from trusty tools, and a Linux box that decided to give me the silent treatment – or rather, the very LOUD "Kernel Panic" treatment. This is the story of how my `/boot` partition went on a diet, choked, and how I (eventually) nursed it back to health, learning a ton along the way.

## The Day My Linux Desktop Decided to Play Dead

It all started with a seemingly innocent desire: to add the LTS (Long Term Support) kernel to my Arch Linux setup. You know, for stability, for options, for the sheer geeky joy of having more than one kernel. Arch Linux, being the wonderfully flexible beast it is, makes this pretty straightforward. Or so I thought.

After the `pacman -S linux-lts linux-lts-headers` command did its thing, I rebooted, anticipating a new entry in my GRUB menu. Instead, I was greeted by a cascade of text, culminating in the dreaded **Kernel Panic**. My screen froze, my heart sank. "Not an OS crash!" I lamented, "This is a *Linux* screen of death!"

### Why Bother With Multiple Kernels Anyway?

Now, you might be asking, "Why complicate life with more than one kernel?" Great question! On a rolling release distro like Arch, having multiple kernels is like having a safety net and a playground rolled into one:

1.  **The Stable Workhorse:** This is your main kernel (e.g., `linux`). It's got the latest features and drivers. Usually, it's rock solid.
2.  **The LTS (Long Term Support) Kernel:** This one (e.g., `linux-lts`) is updated less frequently, focusing on stability and security fixes over a longer period. If a new mainline kernel introduces a bug that affects your hardware, you can boot into the LTS kernel and keep working while the issue gets sorted. It's your "old reliable."
3.  **Testing/Specific Kernels (Optional):** Sometimes you might want to try a testing kernel, a kernel optimized for a specific task (like `linux-zen` for desktop responsiveness), or even compile your own!

Essentially, it's about choice, resilience, and sometimes, troubleshooting. My mistake? Not giving these kernels enough room to, well, *exist*.

## The Culprit: A Cramped `/boot` Partition

After some head-scratching and booting into a live USB, the villain of our story became clear: my `/boot` partition was stuffed to the gills. I had allocated a measly 200MB to it during the initial installation. "Plenty," I'd thought. Famous last words.

### What's Eating All That Space in `/boot`?

The `/boot` partition is critical. It holds the files necessary to get your system up and running before your main root filesystem is even touched. The main space hogs are typically:

* **Kernel Images (`vmlinuz-linux`, `vmlinuz-linux-lts`, etc.):** These are the actual Linux kernels. They're compressed, but they still take up space (tens of MBs each).
* **Initramfs Images (`initramfs-linux.img`, `initramfs-linux-lts.img`):** This is the **Initial RAM Filesystem**. It's a small, temporary root filesystem loaded into memory. Its job is crucial: it contains modules and scripts needed to mount your *actual* root filesystem, especially if it's encrypted (like mine with LUKS) or on a complex storage setup (RAID, LVM). These can be surprisingly chunky, often larger than the kernel image itself, especially if you have lots of hardware or features enabled. Adding a second kernel means adding a second, equally large, initramfs image.
* **Microcode Updates (`intel-ucode.img`, `amd-ucode.img`):** Processor microcode that gets loaded early. Smaller, but they add up.
* **GRUB Configuration and EFI files:** If `/boot` is also your EFI System Partition (ESP) as is common, it will hold your bootloader files (e.g., `EFI/GRUB/grubx64.efi`) and its configuration (`grub/grub.cfg`).

With two kernels, two initramfs images, microcode, and GRUB paraphernalia, my 200MB `/boot` threw in the towel. It simply couldn't handle the new `initramfs-linux-lts.img`.

**How to manage them?** Normally, your package manager handles installing and removing kernel packages. `mkinitcpio` (on Arch) generates the initramfs images, and `grub-mkconfig` updates GRUB. The main "management" is ensuring you have enough space in the first place, or occasionally cleaning up *old, unused* kernel versions (though that wasn't my issue here).

## The Great Disk Shuffle: A Tale of LUKS and KDE Partition Manager

My disk layout was simple: a 200MB `/boot` partition at the start of the disk, and then one massive partition taking up the rest, encrypted with **LUKS (Linux Unified Key Setup)**, holding both my root (`/`) and home (`/home`) directories.

**A Quick Detour into LUKS:** LUKS is fantastic. It encrypts entire block devices (like a partition), meaning all your data is scrambled and unreadable without the passphrase. Peace of mind, achieved! However, encrypted partitions, especially LUKS ones, can be a bit more stubborn when you want to resize or move them.

My initial plan:
1.  Boot from a live USB.
2.  Use a partition manager to shrink the big LUKS partition from its end.
3.  Move the LUKS partition to the right (towards the end of the disk).
4.  This would free up space *after* the original `/boot` partition, allowing me to merge that space and expand `/boot`.

I fired up a Xubuntu live session (more on why Xubuntu later, thanks to Ventoy!) and launched **KDE Partition Manager**. It's a pretty slick tool and, to its credit, it handled shrinking the LUKS partition like a champ! I carefully reduced its size from the right-hand side, freeing up about 1.5GB of unallocated space.

The problem? KDE Partition Manager (at least in the version I was using) didn't seem to want to *move* the LUKS partition. The option to shift its starting point to the right, which would have been necessary to merge the newly created unallocated space with the space *next to* my original `/boot` partition (which was at the very beginning of the disk), just wasn't playing ball for LUKS. Moving partitions, especially encrypted ones, is a delicate operation, and not all tools support all scenarios.

## Plan B: A Brand New `/boot` and a Fond Farewell to 200MB

Okay, so moving the LUKS behemoth wasn't an option. What now? I had this lovely 1.5GB of unallocated space sitting *after* my LUKS partition. The original 200MB `/boot` was still at the very beginning of the disk.

The new plan: Forget the original `/boot`. I'd lose that 200MB (a small sacrifice for a working system and my sanity), and create a *brand new* `/boot` partition in the 1.5GB space I'd just freed up. This new `/boot` would be at the end of my LUKS partition. More than enough room for many kernels to come!

So, the layout would change from:
`[ /boot (200MB) | LUKS (root + home) ]`
to something like:
`[ unused_space (200MB) | LUKS (root + home, shrunk) | /boot_new (1.5GB) ]`

This meant I'd have to reconfigure GRUB EFI to point to this new partition and tell my Arch system where its new boot files lived. A bit more involved, but definitely doable.

## My Trusty Sidekicks: Ventoy and BorgBackup

Before I dived into surgery, a word about the tools that made this whole ordeal manageable:

1.  **Ventoy - The Multi-Boot USB Wizard:**
    If you're not using Ventoy, you're missing out. This magical open-source tool lets you create a bootable USB drive where you can simply copy multiple ISO files (Linux distros, Windows installers, rescue tools, etc.) onto it, and Ventoy presents you with a menu to boot whichever one you want. No more re-flashing USBs for different distros!
    For this adventure, my Ventoy USB had:
    * **Xubuntu:** It comes with KDE Partition Manager (and GParted) out of the box, which I used for the initial LUKS resize.
    * **Arch Linux ISO:** Crucial for the next step – chrooting into my existing system to reconfigure GRUB and the boot setup.

2.  **BorgBackup - My Data Guardian Angel:**
    Before messing with partitions, especially encrypted ones, **BACKUPS ARE NON-NEGOTIABLE.** I use BorgBackup, an amazing open-source deduplicating backup program. It supports compression, client-side encryption (so your backups are safe even if stored remotely), and incremental backups that are fast and space-efficient. Knowing my precious data was safely tucked away with Borg gave me the confidence to proceed. If you value your data, find a good backup solution. Borg is a stellar choice.
    * **Key features:** Deduplication (saves tons of space), compression, encryption.
    * Website: [borgbackup.readthedocs.io](https://borgbackup.readthedocs.io)

## Rebuilding Boot Camp: The Command-Line Ballet

With my new 1.5GB space ready and my Arch ISO booted via Ventoy, it was time for some command-line action. Here's a rundown of the steps, which largely involve chrooting into your installed system from the live environment.

> **Disclaimer:** Device names like `/dev/sda1`, `/dev/sda2` will vary based on your system. Use `lsblk` or `fdisk -l` to identify your partitions correctly. Double-check everything before hitting Enter!

1.  **Create and Format the New Boot Partition:**
    From the Arch live environment, I used `fdisk` to create a new partition in the 1.5GB unallocated space. Let's say this new partition became `/dev/sda3`.
    Since my system uses EFI, and I wanted my `/boot` to be the EFI System Partition (ESP), I formatted it as FAT32:
    ```bash
    sudo mkfs.fat -F32 /dev/sda3
    ```
    If you're not using EFI or prefer a different filesystem for `/boot` (like `ext4`, though FAT32 is required for the ESP itself), adjust accordingly (e.g., `sudo mkfs.ext4 /dev/sda3`). I also set the partition type to "EFI System Partition" using `fdisk` (type code `ef00` or select the appropriate type in `gdisk`).

2.  **Unlock the LUKS Partition:**
    My root filesystem was on, say, `/dev/sda2`.
    ```bash
    sudo cryptsetup luksOpen /dev/sda2 my_encrypted_root
    ```
    This unlocks the LUKS partition and makes it available at `/dev/mapper/my_encrypted_root`.

3.  **Mount Everything:**
    I needed to create a temporary mount structure to chroot into my system.
    ```bash
    # Mount the root filesystem
    sudo mount /dev/mapper/my_encrypted_root /mnt

    # Mount the NEW boot partition inside the mounted root
    # First, create the mount point if it doesn't exist (it should from the original install)
    sudo mkdir -p /mnt/boot 
    sudo mount /dev/sda3 /mnt/boot
    ```
    If you have a separate EFI partition that isn't `/boot` itself (e.g., `/boot/efi`), you'd mount that too: `sudo mount /dev/sdaX /mnt/boot/efi`. In my case, `/boot` *is* the ESP.

4.  **Chroot into the System:**
    This is where the magic happens. We're basically taking over the installed system from the live environment.
    ```bash
    sudo arch-chroot /mnt
    ```
    (Arch-chroot conveniently handles binding `/dev`, `/proc`, `/sys`, etc. If you're on a different live distro, you might need to do these manually:
    `sudo mount --bind /dev /mnt/dev`
    `sudo mount --bind /proc /mnt/proc`
    `sudo mount --bind /sys /mnt/sys`
    `sudo mount --bind /run /mnt/run` (sometimes needed)
    `sudo chroot /mnt /bin/bash` )

5.  **Update `/etc/fstab`:**
    My system needed to know where the new `/boot` partition lived. I first got its UUID:
    ```bash
    # Still inside the chroot
    lsblk -f
    # or
    blkid /dev/sda3 
    ```
    Note down the UUID of `/dev/sda3`. Then, edit `/etc/fstab` (e.g., `nano /etc/fstab`) and update the line for `/boot` with the new UUID and ensure the filesystem type is correct (e.g., `vfat` for FAT32).

    Example old line:
    `UUID=OLD_BOOT_UUID /boot vfat defaults 0 2`

    Example new line:
    `UUID=NEW_BOOT_UUID /boot vfat defaults,umask=0077 0 2` (added umask for FAT ESP)

6.  **Reinstall Kernels (or ensure they are there):**
    Since my old `/boot` was tiny and now abandoned, I needed to make sure my kernels and initramfs images were correctly placed on the *new* `/boot`.
    A simple way to ensure everything is set up correctly is to reinstall the kernel packages. The package manager's hooks will copy the `vmlinuz` files to `/boot`, generate new `initramfs` images in `/boot` using `mkinitcpio`, and update GRUB.
    ```bash
    # Still inside chroot
    pacman -S linux linux-headers linux-lts linux-lts-headers
    ```
    This will also trigger `mkinitcpio -P` to regenerate all initramfs images. If not, you can run it manually:
    ```bash
    mkinitcpio -P
    ```

7.  **Reinstall GRUB:**
    This is the crucial step to make the system bootable again from the new `/boot` partition. Since `/boot` is my ESP, the `--efi-directory` will be `/boot`.
    ```bash
    # Still inside chroot
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCHLINUX --recheck
    ```
    (Replace `ARCHLINUX` with whatever you want your bootloader to be named in the EFI boot menu. `--recheck` can help GRUB find all necessary files).

8.  **Regenerate GRUB Configuration:**
    ```bash
    # Still inside chroot
    grub-mkconfig -o /boot/grub/grub.cfg
    ```
    This scans for kernels and creates the `grub.cfg` file that gives you the boot menu.

9.  **Exit and Reboot:**
    If all commands ran without angry error messages:
    ```bash
    # Still inside chroot
    exit

    # Back in the live USB environment
    sudo umount -R /mnt 
    sudo cryptsetup luksClose my_encrypted_root
    sudo reboot
    ```
    Remove the Ventoy USB and cross your fingers!

## Lessons Learned and a Happy, Booting System

Success! My Arch Linux system booted up, GRUB presented me with both my mainline and LTS kernels, and the kernel panic was but a distant, slightly amusing memory. My new 1.5GB `/boot` partition has plenty of space for future kernel adventures.

What did I learn from this escapade?
* **Generosity is Key (for `/boot`):** Don't skimp on `/boot` space! 200MB is clearly not enough for comfort if you plan on having more than one kernel. These days, 512MB is a safer minimum, and 1GB to 1.5GB (like I have now) is generous and worry-free, especially if `/boot` is also your ESP.
* **LUKS is Awesome, But Plan Ahead:** Full-disk encryption is great for security. Just be aware that resizing and moving LUKS partitions can be trickier. Having a good strategy (and backups!) is essential.
* **Live USBs are Lifesavers:** A well-equipped live USB (thanks, Ventoy!) is indispensable for system rescue and repair.
* **Embrace the Command Line:** While GUIs like KDE Partition Manager are helpful, sometimes the command line offers more power and control, especially for complex tasks like bootloader recovery.
* **Backups, Backups, Backups!** Did I mention BorgBackup? Seriously, before any major disk operation, ensure your data is safe.
* **Every "Oops" is a Learning Opportunity:** While frustrating at the moment, solving these kinds of problems is how you truly learn the ins and outs of your system.

So, the next time your Linux box acts up, don't panic (too much). With a bit of patience, the right tools, and the amazing resources available in the Linux community (Arch Wiki, I'm looking at you!), you can conquer almost any digital beast.

Happy tinkering!

---

### Further Reading & References:

* **Arch Wiki - Kernel:** [https://wiki.archlinux.org/title/Kernel](https://wiki.archlinux.org/title/Kernel)
* **Arch Wiki - GRUB:** [https://wiki.archlinux.org/title/GRUB](https://wiki.archlinux.org/title/GRUB)
* **Arch Wiki - EFI System Partition:** [https://wiki.archlinux.org/title/EFI_system_partition](https://wiki.archlinux.org/title/EFI_system_partition)
* **Arch Wiki - mkinitcpio:** [https://wiki.archlinux.org/title/Mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio)
* **Arch Wiki - LUKS (dm-crypt):** [https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
* **Ventoy:** [https://www.ventoy.net/](https://www.ventoy.net/)
* **BorgBackup:** [https://www.borgbackup.org/](https://www.borgbackup.org/) or [https://borgbackup.readthedocs.io/](https://borgbackup.readthedocs.io/)
* **KDE Partition Manager:** [https://kde.org/applications/system/org.kde.partitionmanager/](https://kde.org/applications/system/org.kde.partitionmanager/)