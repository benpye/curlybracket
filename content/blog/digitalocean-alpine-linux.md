---
date: "2018-10-07T18:07:00Z"
title: "Running Alpine Linux on Digital Ocean"
---

As you may have seen, Digital Ocean have recently added support for custom images [^1]. I have been using Digital Ocean for a long time to run various services for myself and friends and during this time I have been using Ubuntu. I have no issue with running Ubuntu however most services I run now are running within their own container and using a heavy distro for the host OS makes little sense. I looked into running Alpine Linux on a VPS a while ago and whilst Scaleway [^2] have supported Alpine Linux for some time I have often seen criticism of their network performance and never ended up trying it myself. When I recently saw Digital Ocean's announcement I thought it'd be a good time to try getting Alpine Linux running on one of their droplets, without any of the hacks needed in the past [^3]. If you want to skip right ahead and get an Alpine Linux droplet running then check out the scripts on GitHub at https://github.com/benpye/alpine-droplet , if you want to know how this works read on.

Digital Ocean set out a bunch of requirements for a custom image:
- they must support `ext3` or `ext4`
- they must be a disk image they support (we will use `qcow2`)
- the image must have `cloud-init`

We will see later that we can get away with bending the rules a little for the last point.

The Alpine Linux project helpfully provides a tool [^4] to generate an Alpine Linux disk image suitable for a virtual machine, this gets us most of the way to a custom image and actually, just using this tool will provide an image that boots on Digital Ocean however it isn't really usable. The default image will have no networking, no `openssh` server and it will not install your public keys or set it's hostname correctly. For the sake of example, to use the script to generate a basic image invoke the following, we will see later how to add the additional configuration:


```
# ./alpine-make-vm-image -f qcow2 ~/alpine-3.8-virt.qcow2
```

With a typical distribution we could solve these issues by installing `cloud-init` and enabling it on first boot, this doesn't quite work with Alpine Linux however. `cloud-init` appears to assume that your Linux distribution uses `systemd`, it requires `udev`, a fully featured `ip` and maybe other commands that do not exist on Alpine Linux. This is because Alpine Linux does not use `systemd`, it instead uses OpenRC. Additionally Alpine Linux doesn't even have the tradition GNU tools, instead opting to use `busybox`, this pretty much makes `cloud-init` a no-go. Instead, let's look at what `cloud-init` should be doing:

1. Configure networking
2. Set the hostname
3. Configure the `authorized_keys` file for `root`
4. Run any additional user steps specified in `user-data`

We can easily address 1, whilst it would be nice to have a static IP setup it suffices to enable DHCP negotiation. Digital Ocean will respond to a DHCP request from our server and give us the correct public IP. `alpine-make-vm-image` lets us specify a script to run once the rest of the image is configured, within the `chroot`, using this we can enable networking, the following will work to enable the networking service and `udhcpcd`. We will also enable the `sshd` daemon though `authorized_keys` hasn't been configured so it will not yet be possible to actually connect.

```
#!/bin/sh

rc-update add sshd default

cat > /etc/network/interfaces <<-EOF
iface lo inet loopback
iface eth0 inet dhcp
EOF

ln -s networking /etc/init.d/net.lo
ln -s networking /etc/init.d/net.eth0

rc-update add net.eth0 default
rc-update add net.lo boot
```

To address 2 and 3 we need to look at how `cloud-init` would have gotten this configuration, on a Digital Ocean droplet this is provided by their internal metadata service [^5]. To query this service we simply need to send a `HTTP` request at the link local address `169.254.169.254`. Looking through the API we see two endpoints that look to provide the information we require, `http://169.254.169.254/metadata/v1/hostname` and `http://169.254.169.254/metadata/v1/public-keys`, providing the hostname and `authorized_keys` respectively. These endpoints helpfully do not mangle the data in any way and it is sufficient for us to simply `wget` the data from these endpoints to the appropriate config files on first boot. We now need to extend the earlier script to generate a service that we can then enable for the first boot. OpenRC is quite simple and provides a script guide with everything we need to get a basic target working [^6]. All our service does is run a shell script, the shell script grabs the necessary data from the metadata endpoints and correctly configure the permissions before disabling the service preventing it from being executed on subsequent boots. Updating our previous script we get something like the following:

```
#!/bin/sh

rc-update add sshd default

cat > /etc/network/interfaces <<-EOF
iface lo inet loopback
iface eth0 inet dhcp
EOF

ln -s networking /etc/init.d/net.lo
ln -s networking /etc/init.d/net.eth0

rc-update add net.eth0 default
rc-update add net.lo boot

mkdir -p /root/.ssh
chmod 700 /root/.ssh

cat > /bin/do-init <<-EOF
#!/bin/sh
wget -T 5 http://169.254.169.254/metadata/v1/hostname -q -O /etc/hostname\nhostname -F /etc/hostname
wget -T 5 http://169.254.169.254/metadata/v1/public-keys -O /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
rc-update del do-init default
exit 0
EOF

cat > /etc/init.d/do-init <<-EOF
#!/sbin/openrc-run
depend() {
    need net.eth0
}
command="/bin/do-init"
command_args=""
pidfile="/tmp/do-init.pid"
EOF

chmod +x /etc/init.d/do-init
chmod +x /bin/do-init

rc-update add do-init default
```

Putting it all together we use this with the `alpine-make-vm-image` tool as follows:

```
sudo ./alpine-make-vm-image -p openssh -c -f qcow2 ~/alpine-3.8-virt.qcow2 ./setup.sh
```

This will produce an image that will correctly enable networking, get a DHCP lease, update it's hostname and configure the `authorized_keys` file for root. Our script does miss one large chunk of `cloud-init` functionality and that is the ability to parse `user-data`, whilst it would be easy enough to extend this for shell script `user-data`, there is also a `yaml` configuration that `cloud-init` supports which would be quite complex to fully support here, however for me this is functional enough and lets me have a usable Alpine Linux droplet.

[^1]: https://blog.digitalocean.com/custom-images/
[^2]: https://www.scaleway.com/imagehub/alpine-linux/
[^3]: https://github.com/bontibon/digitalocean-alpine
[^4]: https://github.com/alpinelinux/alpine-make-vm-image
[^5]: https://developers.digitalocean.com/documentation/metadata/
[^6]: https://github.com/OpenRC/openrc/blob/master/service-script-guide.md
