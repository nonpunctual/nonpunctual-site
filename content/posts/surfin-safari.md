---
title: "Surfin' Safari"
date: 2022-06-03
tags: ["defaults", "macos", "plistbuddy", "safari"]
---

{{< notice info >}}

**NOTE:** These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----

<br/>

{{< figure src="/images/surfin-safari-cover.jpg" alt="" >}}

It is an objective fact that the Beach Boys are one of the 5 most overrated bands of all time, however, that discussion will have to wait for another time & place...

It is fun when you can still do something slightly interesting with `.plist` modification. 

To wit: a thing for customizing the toolbar in Safari. I am not going to make a script, I'll leave that to you, but, here are the raw ingredients:

```sh
crntusr=$(stat -f %Su /dev/console)
```
<div style="margin-left: 6rem;">- Current user (however you get that...) because Safari preferences are in the user domain.</div><br/>

```sh
sudo -u "$crntusr" defaults read /System/Volumes/Data/Users/"$crntusr"/Library/Containers/com.apple.Safari/Data/Library/Preferences com.apple.Safari.plist 'NSToolbar Configuration BrowserStandaloneTabBarToolbarIdentifier'
```
<div style="margin-left: 6rem;">- Read the .plist with defaults (you can also read it with PlistBuddy).</div><br/>

```sh
sudo pkill -ail Safari
sudo -u "$crntusr" /usr/libexec/PlistBuddy -c "add 'NSToolbar Configuration BrowserStandaloneTabBarToolbarIdentifier':'TB Item Identifiers':4 string HomeToolbarIdentifier" /System/Volumes/Data/Users/"$crntusr"/Library/Containers/com.apple.Safari/Data/Library/Preferences/com.apple.Safari.plist
```
<div style="margin-left: 6rem;">- The PlistBuddy syntax (as always) is a little challenging, but, what's happening is:</div>

- An entry is created in the array:
  - `'NSToolbar Configuration BrowserStandaloneTabBarToolbarIdentifier':'TB Item Identifiers'`
- In this example the Home page button is added
- An index is chosen for where the button should be positioned relative to what is already on the toolbar
- The index order is from left to right

<div style="margin-left: 6rem;">- Make sure Safari is not running when this change is made.</div><br/>


```sh
open /System/Applications/Safari.app
```
<div style="margin-left: 6rem;">- Open Safari & marvel at how computers work...</div><br/><br/>

{{< figure src="/images/safari1.webp" caption="Before" >}}

<br/>

{{< figure src="/images/safari2.webp" caption="After" >}}

<br/>

Enjoy!

<br/><small>Source: https://community.jamf.com/general-discussions-2/surfin-safari-33364</small>
