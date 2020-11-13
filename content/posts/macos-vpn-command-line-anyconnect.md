--- 
draft: false
date: 2020-11-13T18:39:03+01:00
title: "Use Cisco AnyConnect to Connect to a VPN from the Command Line on Mac OS"
description: ""
slug: "" 
tags:
- cli
- vpn
- keychain
- mac
- cisco
- anyconnect
categories:
- Tips & Tricks
externalLink: ""
series:
- Command Line VPN
---

After moving to a new office location, a lot changed -- including the network setup.
Different than before, we have to use [Cisco AnyConnect](https://www.cisco.com/c/de_de/products/security/anyconnect-secure-mobility-client) now to connect to the VPN.
Still, there are different VPNs depending on what I'm working on but everything else is quite different.

First of all, this means I can neither use `networksetup` nor `scutil` which I have used in [my previous script]({{< ref "/posts/macos-vpn-command-line" >}} "Connect to a VPN from the Command Line on Mac OS") to manage my VPN any longer.
For obvious reasons, this means my script suddenly became useless.

My personal requirement is still the same, though: I want to be able to connect to any of the VPNs from the command line (and also disconnect again),
without having to enter a password.
The password should be retrieved from the keychain.

The good news: The Cisco AnyConnect client comes with a command line interface: `/opt/cisco/anyconnect/bin/vpn`
Nevertheless, I want it to work exactly as before.
This means: I want using the VPN to be as easy as this:

```shell script
$ vpn list
$ vpn connect "myVPN"
$ vpn status "myVPN"
$ vpn disconnect "myVPN"
```

## Drawback using Cisco AnyConnect

The AnyConnect CLI requires quite some arguments to connect to the VPN:

- host
- group ID
- user name
- password

Unfortunately, this does not align with what previously was either configured for the VPN or stored in the keychain,
i.e., we will not be able to connect using just some name of a configured VPN.

For instance, the following command could be used to connect to a VPN:

```shell script
$ /opt/cisco/anyconnect/bin/vpn -s connect myvpn.example.com <<"EOF"
1
myUserAccount@myVPN
mySuperStrongPassword
y
exit
EOF
```

**Important Note:**
Be aware that, when used in a script, this will print your password in clear text to the terminal.
I found no way around this, in some way the passwort, etc. was always printed.
So it is really recommended to eventually hide the output from the command by passing it on to `/dev/null`, which is, for sure, less than ideal.

In order to make this work with some name like `myVPN` as before,
we either need to add some hardcoded stuff to our script, or we add a configuration file similar to the `~/.ssh/config` for instance.

## Set up a VPN Configuration File

Let's really use the SSH config at `~/.ssh/config` as an example, and create a config file for our VPNs at `~/.vpn/config`:

```shell script
$ mkdir ~/.vpn
$ touch ~/.vpn/config
```

Then, add something like the following to this file:

```
VPN myVPN
    GroupId 1
    User myUserAccount@myVPN
    Host myvpn.example.com
```

You will find the `GroupId` values to use when you start to connect to a VPN via AnyConnect CLI manually:

```shell script
$ /opt/cisco/anyconnect/bin/vpn connect myvpn.example.com
Cisco AnyConnect Secure Mobility Client (version 4.6.00362) .

Copyright (c) 2004 - 2018 Cisco Systems, Inc.  All Rights Reserved.


  >> state: Disconnected
  >> state: Disconnected
  >> notice: Ready to connect.
  >> registered with local VPN subsystem.
  >> contacting host (myvpn.example.com) for login information...
  >> notice: Contacting myvpn.example.com.

  >> Please enter your username and password.
    0) anycon
    1) myVPN
Group: [myVPN]
```

You should see some similar output as in the snippet above.

With the config file at `~/.vpn/config` we have everything in place to create the VPN connection utility we aim for.
The only missing piece is the password which we will add to the Keychain next.


## Add the VPN Account Password to the Keychain

At first, we will make sure to have the new credentials available in the login keychain:

```shell script
$ security add-generic-password \
  -D "VPN account password" \
  -s "myVPN" \
  -a "myUserAccount@myVPN" \
  -w "mySuperStrongPassword" \
  -T "/usr/bin/security"
```

- `-a` specifies your VPN account name (required)
- `-D` specifies the kind of the keychain entry (optional, default is "application password")
- `-s` specifies the service name (the name of the keychain entry, required)
- `-T` specifies an application which may access the keychain entry (multiple `-T` options are possible)
- `-w` specified your VPN account password

The `-s` option should match the VPN name from your configuration file because we will use it to retrieve the account password from the keychain.

## Create a new Utility Script

Next, we will create a utility script that uses the keychain entry we just created:

```shell script
$ echo '#!/bin/bash' > ~/.vpn/anyVpnConnection.sh && chmod +x ~/.vpn/anyVpnConnection.sh
```

Open the script in an editor, and add this code:

{{< gist andreassiegel 72f9521d7d6a1e58b4754e38531d136e >}}

With that script, you can particularly use a convenient command to connect to a VPN identified by a configured name and reading the password from the keychain:

```shell script
$ ~/.vpn/anyVpnConnection.sh connect "myVPN"
```

Besides that, the `stats` command from AnyConnect basically is aliased as `info`.
Every other command will just be passed on to the original AnyConnect CLI.

Thus, the capabilities are a bit different than with the previous utility script,
and the only actual addition is the more convenient way to connect to a VPN.

In my mind, that is still worth it, the basic requirements are fulfilled.

### Set an Alias

In order to make it yet another bit nicer and more convenient,
we also add an alias so that we can type `vpn` instead of `~/.vpn/anyVpnConnection.sh`.

Add the following line to the configuration of your shell, e.g., `~/.zshrc` or `~/.bashrc`:

```shell script
alias vpn="~/.vpn/anyVpnConnection.sh"
```

Now, using VPN connections is as easy as this:

```shell script
$ vpn connect "myVPN"
$ vpn status
$ vpn info
$ vpn disconnect
```
