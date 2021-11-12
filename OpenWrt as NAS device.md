OpenWrt as NAS device
=====================

Guide to setup an OpenWrt device as NAS device using SMB protocol, optionally
accessible from different locations using WireGuard as point-to-point
connection between different subnets. The guide assumes at least one device
in all of the locations has USB3 port for disk connection and that all
locations run OpenWrt 21.02 or newer software with Luci installed.

Features we want:

- multiple shares support
- multiple users support
- each user has own place to store files
- users can access and write to common share locations
- no issues with directories with many files
- optionally linked with WireGuard to other locations
- optionally allow remote network to access local shares

Step 1: Install necessary packages
----------------------------------

This guide assumes minimal Linux knowledge and that SSH access to the router
is already resolved, so *ssh* into it first.

1. Install USB and EXT family of filesystems support:
```
opkg update
opkg install block-mount lsblk blkid
opkg install kmod-usb-storage kmod-usb2 kmod-usb3 usbutils
opkg install kmod-usb-storage-uas
opkg install kmod-fs-ext4 e2fsprogs
```

2. Optionally install support for F2FS and ExFAT filesystems:
```
opkg install kmod-fs-f2fs f2fs-tools
opkg install kmod-fs-exfat libblkid1
```

3. Optionally install WireGuard and some useful tools:
```
opkg install luci-proto-wireguard luci-app-wireguard 
opkg install iperf3 htop coreutils procps-ng-watch prlimit
```

4. Reboot router to ensure all kernel modules work properly:
```
reboot
```

Now wait until the device is back and *ssh* into it again.

Step 2: Setup an USB storage device
-----------------------------------

We'll assume you have an USB device with one partition on it, so connect the
USB storage device and check if the partition is detected:
```
lsblk
```

Output should include:
```
NAME      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda         8:0    0  1.8T  0 disk
└─sda1      8:1    0  1.8T  0 part /mnt/sda1
```

If the `sda1` partition above has no data on it, we should ideally format it
using EXT4 filesystem to ensure maximum stability. Alternative would be to
use F2FS as flash-friendly storage, but it's not certain how stable it is and
how good its *fsck* is at the moment. Since on our devices we could lose power
at any time, it's better to use the most resilient filesystem. So again, **only
if there is no data on the partition**, format it like this:
```
mkfs.ext4 /dev/sda1
```

The above **will wipe all data from the partition**. We can check the
information on the newly formatted partition above with:
```
blkid
```

Output will look similar to this:
```
/dev/mtdblock7: TYPE="squashfs"
/dev/sda1: UUID="dc51a7be-78cf-4e63-a6cb-2514ed208c19" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8ab2bc86-01"
```

One could have multiple partitions above, but we'll work with `sda1` in this
guide. We need to ensure each time this drive is connected, it's mounted to the
same place. We can import this to UCI's *fstab* configuration file like this:
```
block detect | uci import fstab
```

The above should have created `global` section and also added one `mount`
section to the UCI's *fstab* configuration file. Assuming we are only dealing
with one partition and that our *fstab* configuration only has one mount point
configured, this will ensure our USB device's partition is always mounted and
enabled by default:
```
uci set fstab.@mount[0].enabled='1'
uci set fstab.@global[0].anon_mount='1'
uci commit fstab
```

Now confirm that we have a section in `/etc/config/fstab` which looks like
this (with your uuid):
```
config mount
        option target '/mnt/sda1'
        option uuid 'dc51a7be-78cf-4e63-a6cb-2514ed208c19'
        option enabled '1'
```

Adjust the configuration manually if needed (eg. if you have multiple devices or
partitions). Let's now enable the configuration:
```
/etc/init.d/fstab boot
```

Login to your OpenWrt router's web interface and you should be able to see the
mounted device in the newly added Luci section under *System > Mount points*.
There is an option to safely unmount your USB storage device there as well,
should you need this.

> **_Tip:_** Setting up multiple USB drives is just a variation on the steps
above. You can use an USB3 hub to connect multiple devices to OpenWrt routers,
but beware of power requirements (maybe you will need a powered USB3 hub).

#### Optional considerations for the storage device

As an additional consideration here, be advised that some USB3 storage devices
support (or require) new UAS protocol, which is especially well suited for SSD
devices and allowes them to reach their maximum performance. We have installed
package `kmod-usb-storage-uas` above to hopefully enable this mode for
supported drives, but not all host controllers support this mode. You can run
`dmesg | grep "UAS"` command to check if the kernel complains about this. If it
does, your device will most likely work with *usb-storage* driver instead of
*uas* driver. You can check the used driver with `lsusb -t` command. If your
setup ends up not being able to use the new driver, it should not be that big
of a deal, since final performance will usually be limited by other factors.

Finally, to monitor the health of your drive, you can install package
`smartmontools` and, with our setup above, use command `smartctl -a /dev/sda`
to get the health data. Please get more information on this package online
since you might want to manually provide Drive Database *drivedb.h* file to it,
so that it knows what to make of all the SMART attributes your drive exposes.
To trigger the actual SMART test, you can use one of the following commands:

- `smartctl -t short /dev/sda` (for short test, ~2 minutes)
- `smartctl -t long /dev/sda` (for long test, few hours)

The tests above run in background mode, so you can still use the drive. To
check status of the ongoing test, as well as the result of previous ones, you
can run `smartctl -l selftest /dev/sda` command. Note that some drives do not
show ongoing tests here, so you could also try running `smartctl -a /dev/sda`
command and look for *Self-test execution status* section there.

> **_Tip:_** If you are adventurous enough, you could even schedule regular
cronjobs do preform these checks and notify you in case the drive starts
developing issues that it can detect itself.

Step 3: Setup shared directories
--------------------------------

On most OpenWrt devices, it's not possible to use *Samba 3* or *Samba 4* due to
limited storage space available. However, we can use *ksmbd* implementation of
SMB protocol which is built into the kernel and which requires relatively tiny
amount of space. It's not as advanced as Samba, but it's good enough. Let's
install it and then restart Luci:
```
opkg update
opkg install luci-app-ksmbd ksmbd-utils shadow-useradd
/etc/init.d/uhttpd restart
```

> **_Tip:_** You can skip the next section of adding users to Linux and *ksmbd*
in case you have just upgraded your OpenWrt and already had them setup before.
But to make sure all settings are applied, run `/etc/init.d/ksmbd restart` once.
Also, if you are a Windows user and have issues reconnecting to your network
shares after the upgrade, try restarating service *Workstation* or simply
restart your computer.

We need to add users that will be used to connect to our SMB service. We can
have multiple ones, so in this example we'll set three of them. We will use the
`useradd` command to add Linux users which we will then add as SMB users using
`ksmbd.adduser` command (with different password for SMB). Adjust this to your
needs:
```
useradd joe
useradd jane
useradd share
```

The users above now have account on our system, and you might (or not) want
to give them passwords to be able to login to the system. We will not be
doing that, but instead only setup their access to SMB service. The access
and passwords used to access SMB are different and will be configured with
this command:
```
ksmbd.adduser -a joe
ksmbd.adduser -a jane
ksmbd.adduser -a share
```

> **_Tip:_** To update user's password for SMB, use `-u` parameter, and to
remove users from it, use `-d` parameter with the same command above. To delete
Linux users themselves, you can install and use package `shadow-userdel`. To
manage user groups, you can also install `shadow-groupadd` and `shadow-groupdel`
packages.

Let's make some directories that we want to share over the network:
```
cd /mnt/sda1

mkdir joe
chown joe:joe joe/

mkdir jane
chown jane:jane jane/

mkdir share
chown share:share share/
```

We'll make it so that *joe* and *jane* have their private place to store files
and *share* account can be used for other members. We will also make it so
that *joe* and *jane* can use their own credentials to access *share* directory
and ensure any new files there are created as *share* user in this case (instead
of being created as and owned by *joe* or *jane*).

By now, you should see a new Luci section under *Services > Network Shares*. In
the *Shared Directories* section, add three items. For network share names, we
can use any names we want, but to avoid confusion, we will name them according
to the users that own them. So use names `joe`, `jane` and `share` and enter
their directory paths. Enable *Browsable* and *Hide dot files* options, keep
others unchecked. In the allowed users, put `joe` for *joe*, `jane` for *jane*
and then for *share* directory, add `share joe jane` to allow *joe* and *jane*
to access this network share as well.

Your setup should look like this:

```
joe       /mnt/sda1/joe      []  []  []   joe              []  []  []
jane      /mnt/sda1/jane     []  []  []   jane             []  []  []
share     /mnt/sda1/share    []  []  []   share joe jane   []  []  []
```

Now, to make sure any files created by *joe* and *jane* in the *share* directory
are owned by the *share* user, one simple option is to go to *Edit Template*
tab and add this at the bottom:
```
[joe]
        force user = joe
[jane]
        force user = jane
[share]
        force user = share
```

> **_Tip:_** The section `[joe]` above applies to network share with the same
name. Force user option will make it so that the provided Linux user owns the
newly created files in the directory path of this network share.

Also, while here, consider changing some SMB2 parameters in the global section
like this:
```
[global]
        smb2 max read = 2M
        smb2 max write = 2M
        smb2 max trans = 8K
        max open files = 20000

```

Keep other options as they are. Setting `smb2 max trans` to this low value can
resolve some issues with some clients when browsing directories with many files.
Play with this if you want or if you experience issues when browsing certain
directories.

Click *Save & Apply* now. Note that in order to change the template
configuration, as of now you also have to change any checkbox on the screen.
Otherwise, you will get message *"There are no changes to apply"* and your
changes will not be applied. So a trick when tweaking this configuration is to
also change some checkbox, apply changes and then restore checkbox and apply
changes again.

You should now be able to connect to SMB service from your client devices.
Assuming your device is named *openwrt*, try accessing the router with
`smb://openwrt` or `\\openwrt\` and you should see all three shares. You can
also try replacing *openwrt* with the IP address of the router. Depening with
which user you connect, you might not have access to all directories here (eg.
if you connect as *share* user, you will not have access to *joe* and *jane*
directories). Try making some files if you have connected as *joe* in the
*share* directory, then ensure correct ownership in the shell:
```
cd /mnt/sda1
ls -al share/
```

All files there should have user and group ownership like this:
```
drwxr-xr-x    2 share    share         4096 .
drwxr-xr-x    6 root     root          4096 ..
-rw-rw-rw-    1 share    share         4096 somefile.txt
-rw-rw-rw-    1 share    share         4096 anotherfile.txt
```

> **_Tip:_** There might be other ways to achieve correct file ownership, eg.
using parameter `inherit owner = yes` in the `[share]` section, so if you prefer
using that approach, play with that option instead of `force user` approach that
we used. To force different group ownership, you can also use `force group`
parameter. If you want to know all parameters you can change in *ksmbd*
configuration, search online for term *ksmbd configuration* and you should find
a repository with all options. Additionally, if you are playing with parameters
above, it's advised to tail system logs in the console as you apply them, to
make sure you are not having issues with the *ksmbd* daemon with some of their
values. You can tail logs with `logread -f` command.

Step 4: Setup a WireGuard tunnel
--------------------------------

This is an optional step, in case you want to access the LAN network above and
any services on it (including our shares) from another location. We will be
setting up a simple two location network like this:
```
       ____________ (internet) ____________
      |     ..........................     |
     wan    .                        .    wan
usb [ A ] wire                      wire [ B ]
     lan                                  lan
```

For this to work, at least one router should be visible via the Internet to the
other one. If you want to have more than two locations, this is considered at
the end of this step, but for now we will work with two locations only.

#### Interface setup

In Luci, go to *Network > Interfaces* page on both routers. For proper routing
over the tunnel, ensure that two different LAN networks are on different subnet,
so check the LAN interface and ensure that *IPv4 address* values look similar to
this:
```
On Router A: LAN is 192.168.10.1

On Router B: LAN is 192.168.20.1
```

We will assume you are using `255.255.255.0` as netmask, which is the same as
`/24` for our routing needs further along. Adjust this if needed. Routing will
not work properly if both networks are in the the same subnet, so eg. with
subnet `/24`, the private IP addresses above must use different third octet in
the address.

On both routers, create a new WireGuard interface named `wire`. Note that when
we create this interface, we should use lowercase name, but in Luci, this
interface is usually displayed in uppercase, so we will also be using uppercase
to refer to it in this guide.

On both routers, edit the new WIRE interface. First thing is deciding on a
*Private Key*, which is unique for each router. It is very important to keep
this key secret, so don't expose it even to yourself while on this screen.
Assuming you have generated it using the provided button there, we can continue
with the *IP addresses* configuration.

For the interface IP address, we need to use a new dedicated subnet, different
to both LAN networks. You can use these values as *IP Addresses* on both.
```
On Router A: WIRE is 192.168.222.1/28

On Router B: WIRE is 192.168.222.2/28
```

> **_Tip:_** Notice how both interfaces are on the same subnet with capacity for
14 hosts in this example, due to /28 maskbit. You can of course use a larger
subnet if you want, eg. also /24 if you plan to have up to 254 remote locations.
To better understand subnet capacities, you can search online for *subnet
calculator* term.

Decide on the *Listen port* here, since you will need to open firewall to allow
external connection to it. You will also need it to when seting up peer on the
other router. We will assume you will use port `10000` for this on both routers.

On the *Advanced Settings* tab, disable *Use default gateway* option, *Use DNS
servers advertised from peers*, *IPv6* and any other checkbox which assumes you
want to route all of your traffic via tunnel (we don't want this).

On the *Firewall Settings* tab, make sure to add it to its own dedicated
Firewall zone (name it eg. *wgzone*) on both routers. In the zone name dropdown,
there should be a dedicated input field to enter new zone name. This will make
it easier to come up with special firewall rules later on, should you want to be
specific about this.

Save the interfaces now on both routers and then click on their individual
*Restart* buttons. We now have both interfaces ready, but we still have to make
them able to establish a connection between them.

#### Firewall for WireGuard incoming connection

Before we proceed with setting up peers, let's open a firewall hole for the
incoming WireGuard connection. Assuming you will use `10000` as listen port for
both router interfaces, on both routers go to *Network > Firewall > Traffic
Rules* and add a new rule here:
```
Name: Allow WireGuard from wan
Protocol: UDP
Source zone: wan
Destination zone: Device (input)
Destination port: 10000
Action: Accept
```

Save it and move it above any firewall rule that might be dropping all inbound
traffic from *wan* zone. Note that this rule is just to allow incoming WireGuard
connection and is not related to the traffic that will be flowing inside the
tunnel. In theory, you can do this only on one router, but if both of them are
visible from the Internet, consider doing this on both.

Now, we also need to allow the routing in firewall rules from *lan* zone to
*wgzone* zone and vice versa, on both routers. Go to *Network > Firewall* page
and edit *wgzone*. *Input*, *Output* and *Forward* should be set to `accept`.
*Masquerading* and *MSS Clamping* options should be disabled. And in the
*Allowed forward to/from destination zones* dropdowns, make sure that *lan* zone
is selected in both boxes (be careful **not** to allow *wan* here). Save
changes to the zone. Now also *Save & Apply* changes.

#### Setting up WireGuard peers

Next step is adding Router A as peer on Router B's WIRE interface, and adding
Router B as peer on Router's A WIRE interface. On both routers, go back to
*Network > Interfaces* and edit WIRE interface. Go to *Peers* tab.

On Router A, add a peer and put description as `Link-2-B`, enter public key from
Router B (in the ssh shell on Router B, type command `wg` to see public key --
do not by mistake put any private key here). For *Allowed IPs*:

- add the Router B WIRE interface's IP address: `192.168.222.2`
- add the Router B LAN network address with subnet: `192.168.20.1/24`  

With the setup above, we should be able to route traffic to Router B LAN network
via the tunnel. While here, ensure that *Route Allowed IPs* option is toggled
on and for *Persistent Keep Alive* option you might want to put value `25`.

Now we have to decide on the *Endpoint Host* and *Endpoint Port* fields. You
only need to define these if it's possible for our router to connect to this
peer. For our assumptions to work, at least one of the routers has to be
connectable, so if we are on the Router A and editing its peer connection to
Router B, then Router B should be connectable. If location where Router B is has
a public static IP, use that. If your provider gives you a different IP every
day or so, you should setup a dynamic DNS name and ensure it's updated with the
provided public IP address in some automated manner. OpenWrt has packages for
this, so search online how to set this up, because it's out of scope of this
guide.

> **_Tip:_** If your Router B is behind another router (eg. ISP provided), then
you must open and forward port `10000` from this ISP router to your Router B local
IP address (be it WAN or LAN one from Router B's perspective). Also, if your
dynamic IP address changes and the connection is lost for longer time, the only
way for WireGuard client on the other end to be aware of new the IP address your
DDNS points to is to restart its interface (since it resolves DNS names only
once when its being brought up). This can also be automated one way or another.
Also note that WireGuard supports clients to hop between IP networks and it will
just work, as long as the connection can reliably be established in at least one
direction. This means your phone for example can jump between WiFi and mobile
connections, and any transfers going on at that moment through the tunnel will
not break (or even almost notice the interruption). More details on this can be
read in the WireGuard whitepaper under *Endpoints & Roaming* section.

Repeat the same procedure of adding a peer on Router B: Use description
`Link-2-A`, enter public key from Router A (get it from the ssh shell on Router
A using the same `wg` command). For *Allowed IPs*, add the IP address
`192.168.222.1` and also LAN network from Router A's LAN interface
`192.168.10.1/24`, so we can route traffic to it as well. And if you know its
network public IP address, add it here as *Endpoint Host* together with the
suggested *Endpoint Port* value.

> **_Tip:_** For additional security, you could setup an optional *Preshared
Key* which is usually unique for each peer-to-peer connection. So eg. when
setting up a tunnel between Router A and B, one can put the same value here in
their peer setup screens. To get an appropriate key, you can generate it with
`wg genpsk` on one of the routers (or your computer) and use the same value for
both peers. If you plan to add a third location to the mix, when setting it up,
make sure to a use a different preshared key for better security.

Save both interfaces and wait a minute or so. The connection should be
established and you should see this in the *Status > WireGuard* page. You can
also observe this on both of your router shells with the command `watch wg`.
Look for the *latest handshake* value, it should be defined.

#### Enabling remote location to access local shares

One final thing we have to do now is to configure *ksmbd* daemon to also listen
on the WIRE interface, because by default it's only listening on the LAN
interface.

On the router where we have the USB storage device, open the *Services > Network
Shares* page. Since at the moment it's not possible to select more than one
interface in the *General Settings* page, go to *Edit Template* tab and change
interfaces line to look like this:
```
[global]
	interfaces = |INTERFACES| wire
```

This should make it so that the daemon listens on the interface seleced on the
previous tab as well as on the WIRE interface. Now *Save & Apply* changes (do
the double toggle thing) and you should be able to connect to your shares from
both networks!

#### Adding more locations

Adding more locations to the mix is just a variation on the above. However, with
WireGuard, you have two options how to set this up. In the best scenario, your
locations would be visible to each other, so you would have network like this:

```
  A -- B
  |  /
  | /
  |/
  C
```

So each router would have two peers configured, and IP routing would be
configured in the same way as in this guide. Each peer is allowed to route its
own WIRE IP and its own LAN network subnet.

However, if only one of your locations has a connectable public IP address, you
could also setup your network like this:
```
  A -- B
  |
  |
  |
  C
```

In this scenario, Router A can route any traffic between between B and C
networks. Router A would be the only one having two peers configured (in the
same way as in the setup above). However, both Router B and C would have only
Router A as peer, and they would configure it to route an additional network to
its own ones. In particular:

- on Router C, Router A peer configuration (Link-2-A) would allow it to
additionally route the Router B LAN subnet
- on Router B, Router A peer configuration (Link-2-A) would allow it to
additionally route the Router C LAN subnet

Adding more and more locations now is just a matter of careful peer
configuration, in particular, which networks should they be allowed to route
traffic for. Take into the account that when Router A has to route traffic
between Routers B and C, the transfer speed will be limited by its WAN interface
download and upload link speeds (whichever is lower) as well as its own CPU
capacity, because it has to decrypt traffic from one WireGuard tunnel and then
encrypt it before sending it to another.
