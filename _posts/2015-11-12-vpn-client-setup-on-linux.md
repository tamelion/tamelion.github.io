---
title: Setting up a VPN client on Arch Linux
tags: [vpn, arch, linux]
---
There are many uses for VPNs, from creating a secure connection to a business netork to accessing your home server to accessing online shows in your home country. When I'm travelling away from home and want to prevent the local network of "free WiFi" users sniffing my web traffic, I use [Private Internet Access](http://privateinternetaccess.com). They provide OpenVPN access (which is more secure) and offer plenty of automated setup methods for various operating systems. With Arch, however, we like to do things the manual way...

## Basic setup
Once we've installed the `openvpn` package, the basic VPN setup is very straight forward.

### Downloading the config
Assuming you have already signed up for a PIA account, simply download the [OpenVPN config files](https://www.privateinternetaccess.com/openvpn/openvpn.zip), unzip and move everything to `/etc/openvpn/`.

``` bash
unzip openvpn.zip && cd openvpn
mv * /etc/openvpn
```

### Connecting to the VPN
At this point you could simply `sudo openvpn --config /etc/openvpn/UK\ London.conf` and enter your username and password, but why not add some security steps and also make connecting even easier.

## Security

### Limiting read/write access

We should change the owner/group and read/write access to root for extra security.

``` bash
sudo chown -R root:root /etc/openvpn
sudo chmod -R 600 /etc/openvpn/*
```

### Preventing DNS leaks

Make sure that you edit your `/etc/resolv.conf` with some nameservers you trust, so that you don't get any DNS leaks (where the lookup is performed on your local connection instead on the remote VPN). You could also set this to your VPN service provider's DNS servers in the connection script.

## Optional extras for an easy life

### Automatic credentials

``` bash
sudo vim /etc/openvpn/creds.conf
```

Edit the file with username on the first line and password on the second, then `sudo chmod 600` it and follow either the relative file paths or dmenu section below, according to taste.

### Relative file paths

If you'd rather keep your configs and certificates in separate directories, you can do some batch renaming on the config files. First let's create a temporary environment variable to make the substitutions easy:

``` bash
OPENVPN_DIR=/etc/openvpn
```

Now we can use this with the `sed` tool to replace various things in all the config files. `sed` can use any delimiter for replacement (not just /) so we'll use % to make things play well with paths. Also, we'll use double quotes so the environment variable is read properly.

``` bash
# If you're not there already:
cd /etc/openvpn
# Use file for user/password instead of being prompted
sed -i "s%auth-user-pass%auth-user-pass $OPENVPN_DIR/creds.conf%" *

# Point to correct certificate locations
sed -i "s%ca.crt%$OPENVPN_DIR/ca.crt%" *
sed -i "s%crl.pem%$OPENVPN_DIR/crl.pem%" *

```

## Using dmenu

I love to use dmenu for everything. It makes my life very easy. First let's rename the config files so we don't have to worry about those spaces in the filenames:

```bash
sudo rename ' ' '-' /etc/openvpn/*
```

You can use this dmenu script or modify as required. In this case I'm also telling openvpn to write a pid too so I can tell when VPN is connected (using i3bar etc.)

``` bash
#!/bin/sh

VPNDIR=/etc/openvpn/
VTERM="urxvtc -hold -e"

cd $VPNDIR

vpn=`ls $VPNDIR | grep -Ev "ca.crt|crl.pem|creds.conf" | dmenu -p 'select vpn:'`&& eval "$VTERM sudo openvpn --config $vpn --writepid /var/run/vpn.pid --auth-user-pass creds.conf"
```
