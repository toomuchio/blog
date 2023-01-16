# Building a small PXE recovery shell


Once again had to do something more sysadminisk / dev ops, which I found interesting and can actually write about freely unlike my usual sec work.


This does have an element of security in it as the existing recovery shell used was so out dated it was missing some nvme and nic drivers.

Which was creating some huge issues as the primary purpose behind this shell was to wipe drives on servers.


Now this should really fall to the person who last used the server to remove any sensitive data, but a catch all wipe is needed. If for no other reason than the fact existing partitions can break the next install.


Considered doing this a few ways:
- DBAN, outdated and you have to pay not the best automation options either
- initrd from Debian or other OS installer and burying all the needed modules in there + wipefs, sq3_tools etc. - Hacky and.... Yeah no
- Install Debian with all required packages snapshot it whittle down an initrd - Yuck
- Generate some initrd with initramfs-tools then slap in some rootfs - Eh

## What about Alpine?

A friend recommended I try Alpine, which has mechanisms to make custom ISOs (can just extract the required files out of that for PXE) already, https://wiki.alpinelinux.org/wiki/How_to_make_a_custom_ISO_image_with_mkimage

Note: I'm not covering the building commands and installing all these packages the wiki covers all that well enough.


Ticked all the boxes, thought it would be simple. Overall it was but there's a bunch of weird gotchas that weren't immediately apparent to me.
- Modloop and apkovl files can only be passed over http not tftp, wasted a bunch of time on that.
- For some reason release initrd refuses to boot at all, but edge is fine even when using the official mirrors
- Some of the kernel parameters were confusing - If you copy them out of the mkimage stuff, at one point the way I was building the initrd the only way I could get it to boot was including console= in my PXE parms which made it read only basically, uncommenting the shell on serial getty with aplovl wasn't really working either.
- If you include ip=dhcp in the kernel parameters then try and load networking or require net in openrc it's going to load twice.
- Alpine is missing a lot of basic packages in the main repo, have to enable community which wasn't as apparent as it should have been to me for aplovl to work.


Anyway I had to fight with all those stupid small issues, in-between other things duties, so it took me longer than it really should have.


## mkimg.recovery.sh
```
profile_recovery() {
        profile_standard
        kernel_cmdline="unionfs_size=512M console=tty0 console=ttyS0,115200"
        syslinux_serial="0 115200"
        apks="$apks syslinux util-linux coreutils curl"
        local _k _a
        for _k in $kernel_flavors; do
                apks="$apks linux-$_k"
                for _a in $kernel_addons; do
                        apks="$apks $_a-$_k"
                done
        done
        apks="$apks linux-firmware"
        initfs_features="$initfs_features network"
        hostname="recovery"
        apkovl="genapkovl-recovery.sh"
}
```

Important things to note here:
- Extending the standard profile which already includes nvme drivers and such in the initfs.
- Force including a few useful apks early
- Include network into the initfs_features, which means modloop and apkovl can actually be fetched.
- Add the generation of a custom apkovl


## genapkovl-recovery.sh
```
#!/bin/sh -e

HOSTNAME="$1"
if [ -z "$HOSTNAME" ]; then
        echo "usage: $0 hostname"
        exit 1
fi

cleanup() {
        rm -rf "$tmp"
}

makefile() {
        OWNER="$1"
        PERMS="$2"
        FILENAME="$3"
        cat > "$FILENAME"
        chown "$OWNER" "$FILENAME"
        chmod "$PERMS" "$FILENAME"
}

rc_add() {
        mkdir -p "$tmp"/etc/runlevels/"$2"
        ln -sf /etc/init.d/"$1" "$tmp"/etc/runlevels/"$2"/"$1"
}

tmp="$(mktemp -d)"
trap cleanup EXIT

mkdir -p "$tmp"/etc
makefile root:root 0644 "$tmp"/etc/hostname <<EOF
$HOSTNAME
EOF

makefile root:root 0644 "$tmp"/etc/motd <<EOF
Welcome to the Blah Recovery Shell
EOF

mkdir -p "$tmp"/etc/network
makefile root:root 0644 "$tmp"/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOF

mkdir -p "$tmp"/etc/apk

makefile root:root 0644 "$tmp"/etc/apk/repositories <<EOF
https://dl-cdn.alpinelinux.org/alpine/edge/main/
https://dl-cdn.alpinelinux.org/alpine/edge/community/
https://dl-cdn.alpinelinux.org/alpine/edge/testing/
EOF

makefile root:root 0644 "$tmp"/etc/apk/world <<EOF
alpine-base
curl
jq
mtr
wipefs
sg3_utils
nvme-cli
EOF

mkdir -p "$tmp"/etc/init.d
makefile root:root 0755 "$tmp"/etc/init.d/dowipe <<'EOF'
#!/sbin/openrc-run

depend() {
        after modules hostname
}

start () {
    set -o pipefail

    #Do your wipefs, sg_sanitize, blkdiscard, nvme format stuff here. Fire S.M.A.R.T data to an API and success info.
}
EOF

rc_add devfs sysinit
rc_add dmesg sysinit
rc_add mdev sysinit
rc_add hwdrivers sysinit

rc_add hwclock boot
rc_add modules boot
rc_add sysctl boot
rc_add hostname boot
rc_add bootmisc boot
rc_add syslog boot

rc_add dowipe default

rc_add mount-ro shutdown
rc_add killprocs shutdown
rc_add savecache shutdown

tar -c -C "$tmp" etc | gzip -9n > $HOSTNAME.apkovl.tar.gz
```


Import stuff to note to here
- Standard rubbish, set a hostname, motd and network but I'm not entirely sure that's actually needed but the genapkovl dhcp example included it
- Enable community repositories or we'll be missing a bunch of packages
- Install required packages, standard stuff
- Add init runner task to openrc, after modules and hostname should ensure it's run right at the end. Can't after modloop or require net, since we aren't using modloop and requiring net will start the network process twice. It's already running since that's how apkovl was grabbed and it's included in the initfs.
- rc_add no network or modloop, rest is standard add our runner task into default.


## PXE options
```
kernel alpine/boot/vmlinuz-lts
initrd alpine/boot/initramfs-lts
append ip=dhcp modules=squashfs,sd-mod,usb-storage apkovl=http://_MY_INTERNAL_CDN_IP_HERE_/recovery.apkovl.tar.gz
```

## Summary

Bang that's it, can even login with root blank password and use it as a quick shell to check things after the wipe has happened or disable that task and just use it as a quick shell.


Dispute all the random gotchas I ran into because things aren't documented and frankly I had no idea what I was doing here. Very happy with this, simple to build and thus update. Very small at 31MB for initfs, 8.4MB for kernel and a 2KB for the apkovl.

It **might** be nice to include the packages inside the apkovl or initfs somehow instead of installing them on the fly but at least this way the latest is always installed and the initial boot is very fast tftp is slow...

But if worried about fetching packages from an external source that can change or not wanting to give external network access, we are already running a web server to serve apkovl just put the packages there and alter the /etc/apk/repositories to fetch from your internal CDN.

Can't really do much better than that...
