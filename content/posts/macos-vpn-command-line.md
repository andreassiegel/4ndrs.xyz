--- 
draft: false
date: 2020-05-03T11:53:55+02:00
title: "Connect to a VPN from the Command Line on Mac OS"
description: ""
slug: "" 
tags:
- cli
- vpn
- keychain
- mac
categories:
- Tips & Tricks
externalLink: ""
series: []
---

During Corona time, I have to connect to a VPN most of the time in order to work.
I use two different VPNs depending on what I'm working on.
I can select the network to connect to from the status bar at the top of the screen
but there seems to be a bug: Sometimes there isn't anything to select,
so I have to go to the network preferences and connect from there.

This works, but it also bothers me. Most of what I do frequently can be done from the command line.
There must be a way to also connect to a VPN from there!

It turned out there are multiple ways, actually.
Prerequisite is that the VPN has been configured in the network preferences.

## Use `networksetup`

The `networksetup` command provides access to network settings that are usually configured in the _System Preferences_ application.

Use this command to connect the VPN configured with the name "myVPN":

```shell script
$ networksetup -connectpppoeservice "myVPN"
```

The following command will disconnect again:

```shell script
$ networksetup -disconnectpppoeservice "myVPN"
```

If you want to check the connection status, use:

```shell script
$ networksetup -showpppoestatus "myVPN"
```

## Use `scutil`

The "system configuration utility" or `scutil` command provides access to network configuration, too.

To connect to your VPN, use this command:

```shell script
$ scutil --nc start "myVPN"
```

Execute the following command to disconnect from the VPN:

```shell script
$ scutil --nc stop "myVPN"
```

If you want to check the connection status, use:

```shell script
$ scutil --nc status "myVPN"
```


## Connect with Password from Keychain

Both mentioned commands work with the built-in VPN client.
Unfortunately, it is required to enter the password every time I connect because the account password is not stored in the keychain.
People are discussing for years now whether this is actually a bug or a feature of the built-in VPN client in Mac OS.

Installing another VPN client might solve that problem but you can also use a script as a workaround.

A script or at least an alias is a good idea anyway because I'd like to have something as short as `vpn connect "myVPN"`.
It would be a great plus if that wouldn't even ask me to enter a password but retrieves it from the keychain instead.

Looking at the keychains in the _Keychain Access_ application, I see only entries of kind _IPSec Shared Secret_ in the system keychain and nothing in the login keychain or local items.
Thus, as of now, there isn't anything to retrieve the password from.

### Add the VPN Account Password to the Keychain

To fix that, we first of all add the account password to the login keychain:

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

The `-s` option should match the name of your configured VPN because we will use it to retrieve the account password from the keychain.

### Create a Utility Script

Next, we will create a utility script that uses the keychain entry we just created:

```shell script
$ echo '#!/bin/bash' > ~/vpnConnection.sh && chmod +x ~/vpnConnection.sh
```

This creates the `vpnConnection.sh` bash script in the home directory and makes it executable.

Open the script in an editor, and add this code:

{{< gist andreassiegel 4792195cf1c9425ae09be6b3f23c908c >}}

With that script, you can now use these commands:

Command                                 | Description
--------------------------------------- | ----------------------------------------------------------------
`~/vpnConnection.sh list`               | show all VPN connections
`~/vpnConnection.sh connect "myVPN"`    | connect to the VPN "myVPN" and automatically enter the password
`~/vpnConnection.sh disconnect "myVPN"` | disconnect to the VPN "myVPN"
`~/vpnConnection.sh status "myVPN"`     | show the connection status for VPN "myVPN"


### Configure Privacy

If you run the script now, you might get an error like this:

```
execution error: System Events got an error: osascript is not allowed to send keystrokes. (1002)
```

In this case, you will have to [configure your _Privacy_ settings](https://support.apple.com/guide/mac-help/allow-accessibility-apps-to-access-your-mac-mh43185/mac):

1. Go to _System Preferences > Security & Privacy_
2. Select the _Privacy_ tab
3. Select _Accessibility_
4. Unlock the configuration if necessary
5. Click the _+_ button
6. Press `Cmd+Shift+G` in the Finder window that opens
7. Enter `/usr/bin/osascript` and click _Go_
8. Click _Open_

This will allow `osascript` to control your computer, i.e.,
to automatically interact with the Mac OS user interface.

You might also have to add your terminal application (e.g., _iTerm_) to that list.

### Set an Alias

With that, it is already pretty convenient to manage VPN connections.
To make it even more convenient, we will also set an alias so that we can type `vpn` instead of `~/vpnConnection.sh`.

Add the following line to the configuration of your shell, e.g., `~/.zshrc` or `~/.bashrc`:

```shell script
alias vpn="~/vpnConnection.sh"
```

Now, using VPN connections is as easy as this:

```shell script
$ vpn list
$ vpn connect "myVPN"
$ vpn status "myVPN"
$ vpn disconnect "myVPN"
```
