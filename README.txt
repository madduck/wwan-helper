wwan-helper — WWAN setup integration with ifupdown
==================================================

Copyright © 2013 Volker Grabsch <v@njh.eu> +
Copyright © 2014–2016 Martin F. Krafft <madduck@debian.org>

Released under the terms of the https://opensource.org/licenses/ISC[ISC
licence].

If the script turns out to be helpful to you, consider sending some Bitcoin to
+1L2R8jxPpbuZRyAth4sqSuXNpeks3ow6RH+

Introduction
------------
This hook started out as a helper for Volker to reliably establish a UMTS
connection with his ThinkPad T420s using the Ericsson F3507g Mobile Broadband
Module. If that hardware fails to get a WWAN connection, the modem often needs
a complete reset and reinitialization. Also, the network device needs some
reset during that phase, and so it was only logical to automate the process.

The wwan-helper is meant to run as hook script for Debian's `ifupdown`
mechanism, but should also work on other networking systems. We're looking
forward to your patches…

Martin's Thinkpad X240 came with a slightly different Ericsson card, but
a tiny modification made it work with Volker's script. However, the Thinkpad
T460s no longer permitted the use of these cards, forcing Martin to acquire
a Huawei LTE card, which required a slightly different approach than the
Ericsson cards.

So Martin factored out the card handling and the current framework hopefully
provides enough future-proof flexibility.

Supported cards
---------------

* Ericsson F3507g Mobile Broadband Module
* (Lenovo) Ericsson N5321 HSPA+/3G WWAN M.2 Module
* (Lenovo) Huawei ME906s

Please refer to <<adding_cards,the section on Adding Cards>> for information
on how to add support for other cards.

Getting started
---------------

The hook script uses the `chat` tool, contained in the `ppp` package, which
needs to be installed:

--------------------
sudo apt-get install ppp
--------------------

Then, clone the Git repository. You can place it wherever you want, e.g.
`/etc/wwan-helper` or `/usr/local/src/wwan-helper`. For the purpose of this
document, we'll assume the former. It's a good idea not to use `root` for
this:

----------------------
$ sudo install -d -o $LOGNAME /etc/wwan-helper
$ git clone https://github.com/vog/wwan-helper /etc/wwan-helper
----------------------

Next, install the `ifupdown` hooks using symlinks:

----------------------------
% cd /etc/network
% sudo ln -s /etc/wwan-helper/wwan-helper    if-pre-up.d/local-wwan-helper
% sudo ln -s /etc/wwan-helper/wwan-helper if-post-down.d/local-wwan-helper
----------------------------

Finally, configure your SIM card and carrier in a stanza in
`/etc/network/interfaces`:

.Minimal /etc/network/interfaces configuration of wwan-helper
----------------------------
iface wwan0 inet dhcp
  wwan-apn web.vodafone.de
  wwan-pin 1234
----------------------------

If you don't want other users of the system to be able to read the PIN,
protect the file (e.g. `chmod 660 /etc/network/interfaces`).

For more configuration options, please refer to <<_configuration_options,the
section on configurable options>>. The above two options are the required ones
(technically, the PIN is only required for locked SIM cards, but we advise
that you protect yours with a PIN, if you have not done so already).

To bring up the interface, now all you need to do is call `sudo ifup wwan0`
and wait — the Huawei card takes almost a full minute to initialise. You might
want to add `wwan_verbose 1` to your interface stanza to get some more verbose
output on the process.

Eventually, you'll see your DHCP client obtaining a lease, however:

.DHCP client output following initialisation of the WWAN card
-----------------
Internet Systems Consortium DHCP Client 4.3.5b1
Copyright 2004-2016 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/wwan0/02:1e:10:1f:00:00
Sending on   LPF/wwan0/02:1e:10:1f:00:00
Sending on   Socket/fallback
DHCPDISCOVER on wwan0 to 255.255.255.255 port 67 interval 3
DHCPDISCOVER on wwan0 to 255.255.255.255 port 67 interval 8
DHCPREQUEST of 10.7.133.178 on wwan0 to 255.255.255.255 port 67
DHCPOFFER of 10.7.133.178 from 10.7.133.177
DHCPACK of 10.7.133.178 from 10.7.133.177
bound to 10.7.133.178 -- renewal in 230821 seconds.
------------------

You can take the connection offline with `sudo ifdown wwan0`.
There's even a way to <<suspend_hook,automatically down the interface on
suspend with systemd>>, which is particularly useful with laptops.

Please note that your interfaces name might well differ. Common names include
`usb0`, as well as the unique naming schemes introduced by `systemd`, e.g.
`wwp0s29u1u4i6`. The output of `ip link list` will tell you which name's been
assigned to your card. You can always <<static_iface_names,configure `systemd`
to assign a static name>> if you so prefer, or use the `description`
configuration option for the interface (see `IFACE OPTIONS` in the
`interfaces(8)` manpage).

Configuration options
---------------------

The following options are available:

[options="header",width="50%"]
|==========================================================================
|Option            | Domain  | Description                 | Default
|wwan-apn          | text    | The carrier's APN           | (none)
|wwan-pin          | numeric | The SIM PIN (PIN1)          | (none)
|wwan-retries      | numeric | The number of retries       | 0
|wwan-usbid        | hex:hex | The USB ID of the WWAN card | (autodetected)
|wwan-verbose      | numeric | Script verbosity, see <<_debugging,Debugging>> | 0
|wwan-allow-roaming| boolean | Connect the carrier when roaming | false
|==========================================================================

Individual card handler scripts can add their own options. For instance, the
Ericsson handler also reads `wwan-enforce-umts` (boolean).

Debugging
---------

If you have any trouble, you can disable `wwan-helper` by appending
`wwan-helper-disable` to the stanza in `/etc/network/interfaces`

.Disabling wwan-helper in `/etc/network/interfaces
--------------
iface wwan0 inet dhcp
  […]
  wwan-helper-disable 1
--------------

To get more verbose output, add `wwan-verbose` to the stanza, using the
following integer values:

[options="header",width="25%"]
|===============================================
|`wwan-verbose` value | Effect
|0                    | Only warnings and errors
|1                    | + Informational messages
|2                    | + Chat dialog
|5                    | + Shell script trace (+x)
|10                   | + Shell script echo (+v)
|================================================

Advanced topics
---------------

[[adding_cards]]
Adding support for your WWAN card
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once you know the `AT` commands involved to set up and deconfigure your WWAN
card, it should require no more than a bit of trial and error to teach
`wwan-helper` how to handle it. These are the steps required:

. Start by copying the file `cards/TEMPLATE` to a meaningful filename in the
  same directory.

. Populate the functions therein with the appropriate commands. You can use
  whatever tools are required, including the helper functions defined in the
  `wwan-helper` file.

. Identify the USB ID(s) that your script can handle and provide a symlink to
  your handler from `usbids/abcd:1234`. The format of these link names is
  standardised to be two sets of 4 hexadecimal digits each (including leading
  zeroes if necessary), and limited to lower-case characters.

Once you create a new handler, please submit it upstream so we can include it
for others to use.

[[logical_ifaces]]
Using logical interfaces for multiple configurations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The `ifupdown` system allows one to separate logical configuration from
physical interfaces. This can come in particularly handy if you have multiple
SIM cards that you switch between.

The following excerpt should give you all the input you need to bootstrap your
imagination for how to make it work for your case:

.Multiple logical configurations in /etc/network/interfaces
--------------------------
iface wwan-de-vodafone inet dhcp
  wwan-apn web.vodafone.de
  wwan-pin 1234
  wwan-max-retries 1

iface wwan-at-aon
  wwan-apn apn.aon.at
  wwan-pin 4321
--------------------------

To tell `ifupdown` which logical configuration to use, append it to the name
of the physical interface, like so:

-------------------------
% sudo ifup wwan0=wwan-de-vodafone
-------------------------

It should also be possible to write a mapping script (cf. `interfaces(8)`) to
auto-select the configuration stanza based on SIM card features. Send us
patches when you're done.

[[static_iface_names]]
Configuring static interface names
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you prefer static interface names like `wwan0` over the unique naming
scheme introduced by `systemd`, you could create a file
`/etc/systemd/network/01-rename-wwan.link` with the following content — don't
forget to configure the MAC address of your card in the `[Match]` section:

./etc/systemd/network/local-rename-wwan.link
-----------------------
[Match]
MACAddress=02:1e:14:ef:05:6c

[Link]
Name=wwan0
-----------------------

Now `systemctl daemon-reload` and reinsert the module for your card… or just
reboot your machine, and the new, static name should be used instead.

[[suspend_hook]]
Deconfiguration on suspend
~~~~~~~~~~~~~~~~~~~~~~~~~~

Most cards won't handle the system being suspended and might not be usable
anymore after resume. The following `systemd` service takes the interface down
on suspend:

./etc/systemd/system/local-ifdown@.service
------------------
[Unit]
Description=Down the %I interface

[Service]
Type=oneshot
ExecStart=/sbin/ifdown --force %I

[Install]
WantedBy=sleep.target
------------------

Which you then install like so:

--------------------
# systemctl enable local-ifdown@wwan0.service
Created symlink /etc/systemd/system/sleep.target.wants/local-ifdown@wwan0.service → /etc/systemd/system/local-ifdown@.service.
# systemctl daemon-reload
--------------------


