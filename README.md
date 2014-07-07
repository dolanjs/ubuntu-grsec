ubuntu-grsec
============
### Create airgapped ubuntu 14.04 host to compile grsec on ###
Update the host, install dependencies and create dir structure

```
apt-get update
apt-get upgrade -y
dependencies="libncurses5-dev build-essential kernel-package git-core gcc-4.8 gcc-4.8-plugin-dev make"
apt-get install $dependencies -y
mkdir ~/grsec
cd ~/grsec
```

Gather required public keys used to verify downloads later

```
wget http://grsecurity.net/spender-gpg-key.asc
gpg --import spender-gpg-key.asc
gpg --recv-keys 6092693E
```
At this point treat the machine as an airgapped host

### Gather required file from networked computer ###

```
mkdir ~/grsec
cd ~/grsec
```

Get kernel-package directory from a precise 12.04 server install

```
cp -a /usr/share/kernel-package ubuntu-package
```

Download and Verify files

```
kernel_ver="3.14.10"
grsec_patch_ver="3.0-3.14.10-201407052031"

wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-${kernel_ver}.tar.gz
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-${kernel_ver}.tar.sign
wget http://grsecurity.net/stable/grsecurity-${grsec_patch_ver}.patch
wget http://grsecurity.net/stable/grsecurity-${grsec_patch_ver}.patch.sig
```

Download ubuntu overlay

```
git clone git://kernel.ubuntu.com/ubuntu/ubuntu-precise.git  
```

Transfer files to airgapped host's ~/grsec directory

### On the airgapped host

```
mkdir ~/grsec
cd ~/grsec
```

Copy the required directories from ubuntu overlay to the correct ubuntu-package folder
```
cp ubuntu-precise/debian/control-scripts/p* ubuntu-package/pkg/image/  
cp ubuntu-precise/debian/control-scripts/headers-postinst ubuntu-package/pkg/headers/
```

Verify files

```
kernel_ver="3.14.10"
grsec_patch_ver="3.0-3.14.10-201407052031"

gpg --verify grsecurity-${grsec-patch-ver}.patch.sig  
gunzip linux-${kernel_ver}.tar.gz
gpg --verify linux-${kernel_ver}.tar.sign
```

Apply patch to kernel
```
tar -xf linux-${kernel_ver}.tar  
cd linux-${kernel_ver}
patch -p1 <../grsecurity-${grsec_patch_ver}.patch
```

Select and configure grsec options

```
make menuconfig
```
* navigate to 'Security options'
* navigate to 'Grsecurity'
* enable the ‘Grsecurity’ option
* Set ‘Configuration Method’ to ‘Automatic’
* Set ‘Usage Type’ to ‘Server’
* Set ‘Virtualization Type’ to ‘None’
* Set ‘Required Priorities’ to ‘Security’
* exit and save

Use all the available cores to compile the kernel

```
export CONCURRENCY_LEVEL="$(grep -c '^processor' /proc/cpuinfo)"
```

Make a clean kernel
Will sometimes fail on small VPSs

```
make-kpkg clean  
make-kpkg --initrd --overlay-dir=../ubuntu-package kernel_image kernel_headers 
```

Distribute the deb packages to the app and monitor servers

### On the app and monitor servers
Resolve grub pax

```
apt-get install paxctl -y  
paxctl -Cpm /usr/sbin/grub-probe  
paxctl -Cpm /usr/sbin/grub-mkdevicemap  
paxctl -Cpm /usr/sbin/grub-setup  
paxctl -Cpm /usr/bin/grub-script-check  
paxctl -Cpm /usr/bin/grub-mount  
```

Resolve apache pax only on app server

```
paxctl -cm /var/chroot/source/usr/bin/apache2
paxctl -cm /var/chroot/document/usr/bin/apache2
```

Install grsec kernel

```
dpkg -i *.deb
update-grub
```

Disable conflicting chroot restriction in sysctl.conf only on app server

```
echo "kernel.grsecurity.chroot.caps = 0" >> /etc/sysctl.conf
echo "kernel.grsecurity.chroot.deny.unix = 0" >> /etc/sysctl.conf

sysctl -p /etc/sysctl.conf
```

Configure grsec kernel to be default


Check for exact grsec kernel menuentry name

```
grep menuentry /boot/grub/grub
```

Copy the menuentry and use it in the sed command to set the default grub kernel

```
kernel_ver="3.14.10"
sed -i "s/^GRUB_DEFAULT=.*$/GRUB_DEFAULT=2>Ubuntu, with Linux ${kernel_ver}-grsec/" /etc/default/grub
update-grub
reboot
```

Test functionality

Configure grsec lock in sysctl.conf

```
echo "kernel.grsecurity.grsec_lock = 1" >> /etc/sysctl.conf
```
