# linpe
Linux Diskless Network Boot [PXE]

## Recognition
Inspired from [@espebra's](https://github.com/espebra/) pages:
- [centos-7-rootfs-on-tmpfs](http://www.espenbraastad.no/posts/centos-7-rootfs-on-tmpfs/)
- [zfs-nas-on-tmpfs](http://www.espenbraastad.no/posts/zfs-nas-on-tmpfs/)

## What's all this?
While troubleshooting servers, I need a way to boot them from a specific Linux image. 
You could use a USB stick or PXE; the latter scales better.

## What do you need:
- Build Server:
     - Centos/Redhat (verified working CentOS7,8 & RHEL8)
     
- Network Server:
     - DHCP server
     - HTTP server
     
- Client:
     - iPXE capable NIC

 ## Build server
 - Installed RHEL-latest (currently 8.5) minimal
 - Install packages
      ```
      dnf --assumeyes install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
      dnf --assumeyes upgrade
      dnf --assumeyes install pixz pigz
      ```
 - pick your destination
      ```
      work="/work"
      mkdir -p "$work/initramfs/bin/"
      mkdir -p "$work/newroot/"
      mkdir -p "$work/result/"
      ```
 - sync the OS
      ```
      cd /
      for i in bin boot dev etc home lib lib64 media mnt opt root sbin srv tmp usr var
      do  
        tar cf - "$i/" | (cd "$work/newroot/; tar xvf -) 
      done
      ```
 - install/remove packages from new root
      ```
      dnf --installroot="$work/newroot/" update
      dnf --installroot="$work/newroot/" erase zsh
      dnf --installroot="$work/newroot/" install ethtool fio hdparm iperf3
      ```
- clean some bits
     ```
     echo > "$work/newroot/etc/fstab"

     echo "SELINUX=disabled" > "$work/newroot/etc/selinux/config"

     mkdir "$work/newroot/etc/systemd/system/getty@.service.d"
     cat > "$work/newroot/etc/systemd/system/getty@.service.d/noclear.conf" << EOF
     [Service]
     TTYVTDisallocate=no
     EOF
     ```

- set up the initial init sequence
     ```
     wget -O "$work/initramfs/bin/busybox" https://www.busybox.net/downloads/binaries/1.28.1-defconfig-multiarch/busybox-x86_64
     
     chmod +x "$work/initramfs/bin/busybox"
     
     cat > "$work/initramfs/init" << EOF
     # Dump to sh if something fails
     error() {
        echo "Jumping into the shell..."
        setsid cttyhack sh
     }
     # Populate /bin with binaries from busybox
     /bin/busybox --install /bin
     mkdir -p /proc
     mount -t proc proc /proc
     mkdir -p /sys
     mount -t sysfs sysfs /sys
     mkdir -p /sys/dev
     mkdir -p /var/run
     mkdir -p /dev
     mkdir -p /dev/pts
     mount -t devpts devpts /dev/pts
     mkdir -p /dev/mqueue
     mount -t mqueue mqueue /dev/mqueue
     # Populate /dev
     echo /bin/mdev > /proc/sys/kernel/hotplug
     mdev -s
     mkdir -p /newroot
     mount -t tmpfs -o size=8192m tmpfs /newroot || error
     echo "Extracting rootfs... "
     xz -d -c -f rootfs.tar.xz | tar -x -f - -C /newroot || error
     mount --move /sys /newroot/sys
     mount --move /proc /newroot/proc
     mount --move /dev /newroot/dev
     exec switch_root /newroot /sbin/init || error 
     EOF
          
     chmod +x /work/initramfs/init
     
     cd "$work/newroot"
     tar cJf "$work/initramfs/rootfs.tar.xz" .
     
     cd "$work/initramfs"
     find . -print0 | cpio --null -ov --format=newc | gzip -9 > "$work/result/initramfs.gz"
     
     cp "$work"/newroot/boot/vmlinuz-*x86_64 "$work/result/vmlinuz"
     
     scp -r "$work"/result/* $networkserver:/var/www/html/boot/pxe-rh8/
     ```
     
 ## Network server
 
 ### DHCP Server (ISC-dhcp)
 Details how to set up are on the [iPXE](https://ipxe.org/howto/dhcpd) website.
 
      ```
      ```
 
 ### HTTP Server (apache)
 
 
 ## Client
  
