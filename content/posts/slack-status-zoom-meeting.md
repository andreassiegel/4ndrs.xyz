--- 
draft: false
date: 2020-11-28T10:15:03+01:00
title: "Using Hammerspoon to automatically set Slack Status when in Zoom Meeting"
description: ""
slug: "" 
tags:
- hammerspoon
- slack
- zoom
- chat
- call
- conferencing
- status
- automation
- mac
categories:
- Tips & Tricks
externalLink: ""
series: []
---

This year, online communication definitely is on a rise,
and there is a wide variety of tools for that. Two of them are Slack and Zoom.

[Slack](https://slack.com/) is probably the best tool for instant messaging.
Quite a while ago, they introduced a pretty cool feature already known from instant messaging:
In addition to just indicating your presence (active/online or away/offline),
you can set also set your status to whatever you like, including some icon or emoticon shown right next to your name.
Probably the best thing about that was introduced a little later, as far as I remember:

Whenever you are on a (Slack) call, a special icon is displayed to indicate exactly that.
Unfortunately, calls in Slack recently really became a pain, mostly because poor quality, and high resource and bandwidth consumption.
A colleague of mine recently noticed that every participant in a meeting took another 1 MBit.
So it was no surprise he was not able to join our team calls from home anymore.
Besides that, Slack has a lack of features as well, e.g.,
when in a call, you can either share your screen **or** use your webcam.

[Zoom](https://zoom.us/) is much better in all of that:
It has good quality in calls, and a lot of great features for online conferencing.

However, people in Slack will not see when you are in a meeting/call (like they do for Slack calls), and, therefore,
might not be able to respond to their messages in a timely manner -- and even worse:
They do not see you are not available for a quick call.

If just there was a way to update your Slack status when you are in a Zoom meeting...
Automation to the rescue!


## Automation via Hammerspoon

[Hammerspoon](https://www.hammerspoon.org/) is here to help with that, at least on MacOS.
It is a tool for powerful automation which is basically based on a bridge between the operating system and a Lua scripting engine.

Its power comes from extensions that expose specific pieces of system functionality to the user.

You can write Lua code to interact with all kinds of APIs of the operating system, e.g., for applications, windows, mouse pointers, filesystem objects, audio devices, batteries, screens, low-level keyboard/mouse events, clipboards, and more.

Hammerspoon can easily be installed via [Homebrew](https://brew.sh/):

```shell script
$ brew install hammerspoon
```

Afterwards, make sure to add it to the allowed tools for _Accessibility_ in the _Security & Privacy_ settings.
Usually, Hammerspoon shoould ask you to grant permission when you first launch it.
Also, it will ask you to grant access to documents so that it can read Lua scripts.

Now, let us find a way to make use of that to update the Slack status when we are in Zoom.

## Slack Status Updater

[Mark Harrison](https://www.mharrison.org/) created a script that can be used for that.
You find it on GitHub: [mivok/slack_status_updater](https://github.com/mivok/slack_status_updater)

It comes with batteries included and is ready to be used with Hammerspoon:
There already is a [Lua helper script to detect Zoom meetings](https://github.com/mivok/slack_status_updater/blob/master/helpers/zoom_detect.lua).

Use the below snippet to clone the repository, copy the Lua script to Hammerspoon and initialize it:

```shell script
$ git clone git@github.com:mivok/slack_status_updater.git
$ cp slack_status_updater/helpers/zoom_detect.lua ~/.hammerspoon/
$ echo "local zoom_detect = require(\"zoom_detect\")" >> ~/.hammerspoon/init.lua
```

In addition to that, make sure `slack_status.sh` is in your `$PATH`, e.g.,
by adding the cloned repository (assuming you are still in the same directory) to your `.zshrc` or `.bashrc` (or whatever you use):

```shell script
echo "export PATH="$(pwd)"/slack_status_updater:\$PATH" >> ~/.zshrc
```

Now, you are ready to configure the script:

1. [Create a new Slack app](https://api.slack.com/apps/new)
2. Set the name to "Slack Status Updater" and select your workspace
3. In the app configuration, go to _OAuth and Permissions_ and _User token scopes_, and add the scope `users.profile:write`
4. Click _Install App to Workspace_ at the top and allow the app to access your workspace in Slack
5. Copy the access token starting with `xoxp-` that will be shown next.

With that token, set up the status updater script via this command, and follow the prompts:

```shell script
$ slack_status.sh setup
```

This will create the configuration file `~/.slack_status.conf` for you.
This file will include your token and presets to update your status, for instance:

```
# vim: ft=sh
# Configuration file for slack_status
TOKEN=xoxp-11...

PRESET_EMOJI_test=":white_check_mark:"
PRESET_TEXT_test="Testing status updater"

PRESET_EMOJI_zoom=":zoom:"
PRESET_TEXT_zoom="In a zoom meeting"
```

As you can see, it will use the emoji `:zoom:` for your status by default.
However, you might not have such an icon in your workspace,
so you might have to add a custom emoji with that name, e.g., [this one](https://www.stickpng.com/img/icons-logos-emojis/video-conference-software-providers/zoom-icon-logo).

Afterwards, you can test this by executing this command:

```shell script
$ slack_status.sh test
```

If everything works, your status should show a check mark now.
If also the Hammerspoon configuration is loaded sucessfully,
your Slack status will be set accordingly.

Nevertheless, note that the status might not get updated immediately when you join a Zoom meeting.
By default, the script to detect Zoom meetings checks that every 20 seconds
so that it might take a few seconds to update the status in Slack.

## Webhook-based Alternative

Alternatively, you can use an alternative solution that might work on other systems than MacOS:

It uses also an app at Zoom's end to subscribe to events, and then use a Node.js application
to update the Slack status.

This solution requires a Vercel account.

Read more in Stefan Natter's article ["How to automatically update your Slack status when you join a Zoom meeting" on DEV.TO](https://dev.to/natterstefan/how-to-automatically-update-your-slack-status-when-you-join-a-zoom-meeting-28e0) or check out his [repository on GitHub](https://github.com/natterstefan/zoom-slack-status-updater).

## Downside of Automation

Nice, we saw two different approaches to updating the Slack status while being on a Zoom call.

However, keep in mind, that updating the Slack status via any kind of automation might interfere with other plugins that update your status, e.g., [Slack for Outlook](https://slack.com/apps/AFS3736H3-slack-for-outlook):

Usually, this add-on updates your status when you have meetings in your Outlook calendar,
i.e., it shows a calendar icon in your Slack status.
That's removed once the meeting (in the calendar) ends.

However, when you join a Zoom meeting, your Slack status is updated to reflect that.
When you leave the meeting, your Slack status is _cleared_ -- regardless of whether there was still an ongoing appointment in your Outlook calendar.

The same happens to the status you might have set manually in Slack before,
like "Vacationing" or "Working remotely".
