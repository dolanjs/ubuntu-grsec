Ubuntu kernel with grsecurity
=============================

This guide outlines the steps required to compile a kernel for [Ubuntu Server 12.04 LTS (Precise Pangolin)](http://releases.ubuntu.com/12.04/) with [Grsecurity](https://grsecurity.net/), specifically for use with [SecureDrop](https://pressfreedomfoundation.org/securedrop).

## Before you begin

The steps in this guide assume you have the following set up and running:

 * SecureDrop App and Monitor servers (see the [installation guide](https://github.com/freedomofpress/securedrop/blob/develop/docs/install.md))
 * An offline server running [Ubuntu Server 14.04 (Trusty Tahr)](http://releases.ubuntu.com/14.04/) that you use to compile the kernel
 * An online server running 12.04 or 14.04 that you use to download package dependencies

The idea is that you will use the online server to download package dependencies, put the files on a USB stick and transfer them to the offline server, then use the offline server to verify digital signatures and compile the new kernel.

The current version of this document assumes you are compiling Linux kernel version *3.2.61* and Grsecurity version *3.0-3.2.61-201407232156*. When running commands that include filenames and/or version numbers, make sure it all matches what you have on your server.

## Update packages on online and offline servers

Run the following set of commands to ensure all packages are up to date on the online and offine servers.


```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

## Install dependencies on the offline server

Run the following command to install the package dependencies required to compile the new kernel with Grsecurity.

```
sudo apt-get install libncurses5-dev build-essential kernel-package git-core gcc-4.8 gcc-4.8-plugin-dev make
```

Create a directory for Grsecurity and download the public keys that you
will later use to verify the Grsecurity and Linux kernel downloads.

```
mkdir grsec
cd grsec/
wget https://grsecurity.net/spender-gpg-key.asc
gpg --import spender-gpg-key.asc
gpg --recv-key 6092693E
```

At this point, you should disconnect this server from the Internet and treat it as an offline (air-gapped) server.

## Download kernel and Grsecurity on the online server

Create a directory for Grsecurity, the Linux kernel, and the other tools you will be downloading.

```
mkdir grsec
cd grsec/
```

Make a copy of the *kernel-package* directory on the online server.

```
cp -a /usr/share/kernel-package ubuntu-package
```

When downloading the Linux kernel and Grsecurity, make sure you get the long term stable versions and that the version numbers match up.

```
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.2.61.tar.xz
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.2.61.tar.sign
wget https://grsecurity.net/stable/grsecurity-3.0-3.2.61-201407232156.patch
wget https://grsecurity.net/stable/grsecurity-3.0-3.2.61-201407232156.patch.sig
```

Download the Ubuntu kernel overlay.

```
git clone git://kernel.ubuntu.com/ubuntu/ubuntu-precise.git
```

Transfer all the files in the *grsec* directory from the online server to the offline server using a USB stick.

## Before you compile on the offline server

After moving the files from the online server to the offline server, you should have the following in your *grsec* directory.

```
grsecurity-3.0-3.2.61-201407232156.patch	    spender-gpg-key.asc
grsecurity-3.0-3.2.61-201407232156.patch.sig	ubuntu-package/
linux-3.2.61.tar.sign				            ubuntu-precise/
linux-3.2.61.tar.xz
```

### Gather the required files for the Ubuntu kernel overlay

Copy the required directories from the Ubuntu kernel overlay directory to the correct *ubuntu-package* directory.

```
cp ubuntu-precise/debian/control-scripts/p* ubuntu-package/pkg/image/
cp ubuntu-precise/debian/control-scripts/headers-postinst ubuntu-package/pkg/headers/
```

### Verify the digital signatures

Verify the digital signature for Grsecurity.

```
gpg --verify grsecurity-3.0-3.2.61-201407232156.patch.sig
```

Verify the digital signature for the Linux kernel.

```
unxz linux-3.2.61.tar.xz
gpg --verify linux-3.2.61.tar.sign
```

Do not move on to the next step until you have successfully verified both
signatures. If either of the signatures fail to verify, go back to the online
server, re-download both the package and signature and try again.

### Apply Grsecurity patch to the Linux kernel

Extract the Linux kernel archive and apply the Grsecurity patch.

```
tar -xf linux-3.2.61.tar
cd linux-3.2.61/
patch -p1 < ../grsecurity-3.0-3.2.61-201407232156.patch
```

### Configure Grsecurity

Configure Grsecurity with the following command.

```
make menuconfig
```

You will want to follow the steps below to select and configure the correct options.

 * Navigate to *Security options*
   * Navigate to *Grsecurity*
     * Press *Y* to include it
     * Set *Configuration Method* to *Automatic*
     * Set *Usage Type* to *Server* (default)
     * Set *Virtualization Type* to *None* (default) 
     * Set *Required Priorities* to *Security*
     * Select *Exit*
   * Select *Exit* 
 * Select *Exit*
 * Select *Yes* to save

### Compile the kernel with Grsecurity

Use all available cores when compiling the kernel.


```
export CONCURRENCY_LEVEL="$(grep -c '^processor' /proc/cpuinfo)"
```

Compile the kernel with the Ubuntu overlay. Note that this step may fail if you
are using a small VPS/virtual machine.

```
make-kpkg clean  
sudo make-kpkg --initrd --overlay-dir=../ubuntu-package kernel_image kernel_headers 
```

When the build process is done, you will have the following Debian packages in the *grsec* directory:

```
linux-headers-3.2.61-grsec_3.2.61-grsec-10.00.Custom_amd64.deb
linux-image-3.2.61-grsec_3.2.61-grsec-10.00.Custom_amd64.deb
```

Put the packages on a USB stick and transfer them to the SecureDrop App and Monitor servers.

## Install and configure PaX on App and Monitor servers

Proceed with the following steps only if the SecureDrop App and Monitor servers are up and running.

Both servers need to have PaX installed and configured. PaX is part of common
security-enhancing kernel patches and secure distributions, such as Grsecurity.

```
sudo apt-get install paxctl
sudo paxctl -Cpm /usr/sbin/grub-probe  
sudo paxctl -Cpm /usr/sbin/grub-mkdevicemap  
sudo paxctl -Cpm /usr/sbin/grub-setup  
sudo paxctl -Cpm /usr/bin/grub-script-check  
sudo paxctl -Cpm /usr/bin/grub-mount  
```

### Ensure the web server can start on the App server

The following commands ensure the web server can start and should **only** be
run on the App server.

```
sudo paxctl -cm /var/chroot/source/usr/sbin/apache2
sudo paxctl -cm /var/chroot/document/usr/sbin/apache2
```

### Install new kernel on both App and Monitor servers

Install the new kernel with Grsecurity on both servers.

```
sudo dpkg -i *.deb
sudo update-grub
```

### Disable conflicting jail restrictions on the App server

The following commands disable conflicting jail restrictions and should **only**
be run on the App server.

```
sudo echo "kernel.grsecurity.chroot.caps = 0" >> /etc/sysctl.conf
sudo echo "kernel.grsecurity.chroot.deny.unix = 0" >> /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

### Set the new kernel to be the default on both App and Monitor servers

Set the new kernel to be the default on both servers. Start by finding the
exact menuentry name for the kernel.

```
grep menuentry /boot/grub/grub
```

Copy the output and use it in the *sed* command below to set this kernel as the default.

```
sudo sed -i "s/^GRUB_DEFAULT=.*$/GRUB_DEFAULT=2>Ubuntu, with Linux 3.2.61-grsec/" /etc default/grub
sudo update-grub
sudo reboot
```

### Test SecureDrop functionality

Before you move on to the final step, ensure that SecureDrop is working as expected by testing SSH, browsing to the .onion sites, completing the submission and reply process, and so on.

### Lock it down

Once you have confirmed that everything works, configure the Grsecurity lock in sysctl.conf.

```
sudo echo "kernel.grsecurity.grsec_lock = 1" >> /etc/sysctl.conf
```
