---
layout: post
title: Skype Command Line Arguments
excerpt: "How to run two instances of skype using different accounts."
date: 2018-01-13
categories: [skype]
comments: false
share: true
firehose: false
---

## The Problem

I often find myself needing to run multiple instances of skype at the same time, e.g. personal account and work account. I like to keep these two things seperate. In fact for one company I worked for it was a requirement that we not use personal skype accounts for work related calls or chats and vice versa. On the face of it though skype only allows you to login to one account at any one time.

## The Solution

Well it turns out that the skype executable supports several command line options that can allow you to launch mutiple instances using different accounts. It's worth noting that this only applies to the desktop version of skype. There doesn't appear to a solution for the Windows 10 (or UWP) version of skype at the current time. It also does not seem to work with Lync or Skype for business.

The [skype support page](https://support.skype.com/en/faq/fa171/can-i-run-skype-for-windows-desktop-from-the-command-line) details the following command line parameters: -

| Command              | Description                                           |
| :------------------- | :---------------------------------------------------- |
| /nosplash            | Do not display splash screen when Skype starts.       |
| /minimized           | Skype is minimized to system tray when it starts.     |
| /callto:nameornumber | Call the specified Skype Name or phone number.        |
| /shutdown            | Close Skype.                                          |
| /secondary           | Allows you to start an additional skype.exe instance. |

Cool, so the _/secondary_ argument sounds like it would be helpful in solving the problem and in fact the skype support website [documents](https://support.skype.com/en/faq/FA829/how-can-i-run-multiple-skype-accounts-at-the-same-time) a pretty good way of creating a Windows shorcut that will start skype using this /secondary parameter.

We could also create a simple powershell script to help us launch a secondary instance.

```powershell
skype.exe /secondary
```

This works fine but it's a bit annoying that I have to keep typing in the username field each time.
After a quick search on ServerFault it seems skype also supports a _/username_ switch that will pre-populate the username field with whatever value you give it. So we can do this...

```powershell
skype.exe /secondary /username:joebloggs
```

Pretty handy! So we can now launch multiple instances of skype at the click of a button.
