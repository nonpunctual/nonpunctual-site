---
title: "Automating the \"Mac Evaluation Utility\" App"
date: 2021-11-12
tags: ["applescript","jamf nation","macos","script"]
---

{{< notice info >}}

These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

Jamf Nation post - https://community.jamf.com/general-discussions-2/automating-the-mac-evaluation-utility-app-25934

{{< /notice >}}

<br/>

-----
<br/>

This is a (pretty dumb) script that will automate the execution of the "Mac Evaluation Utility" v. 4.0.1

This is a PoC & I plan to add more options & checking to it before I put it into production as part of a Jamf Pro Self Service policy. Right now, it sends the `.json` report output as an email attachment using the Microsoft Outlook app. It does have some tasty `osascript` / AppleScript bits for sending the email & reading an app's GUI elements that could be repurposed generically. Feel free to change it up for your needs!

```bash
#!/bin/bash
# compatible with Mac Evaluation Utility version 4.0.1 (2)

applpth='/Applications/Mac Evaluation Utility.app'
compnam="$(/usr/sbin/scutil --get ComputerName)"
crntusr="$(/usr/bin/stat -f %Su /dev/console)"
rprtend=''
rprtpth="/private/tmp/$compnam.json"
sprtpth="/Users/$crntusr/Library/Application Support/Mac Evaluation Utility/"
sqltpth="$sprtpth/MacEvalUtility.sqlite"

>&2 /bin/echo 'removing old files...'

/usr/bin/pkill -ai "Mac Evaluation"
/bin/sleep 1
/bin/rm -rf "$rprtpth" "$sprtpth"

>&2 /bin/echo 'executing Mac Eval Util...'

/usr/bin/open "$applpth"
/bin/sleep 3

/usr/bin/osascript \
    -e 'tell application "System Events"' \
    -e 'keystroke return' \
    -e 'delay 1' \
    -e 'keystroke "r" using command down' \
    -e 'end tell'

complete(){
    /usr/bin/osascript \
        -e 'tell application "System Events"' \
        -e 'activate application "Mac Evaluation Utility"' \
        -e 'get static text of group 1 of toolbar 1 of window "Mac Evaluation Utility" of application process "Mac Evaluation Utility" of application "System Events"' \
        -e 'end tell'
}

until echo "$rprtend" | /usr/bin/grep -q 'Report Complete'
do
    rprtend="$(complete)"
    >&2 /bin/echo 'waiting for Mac Eval Util...'
    /bin/sleep 3
done

>&2 /bin/echo 'report complete.'

if [ -n "$(/usr/bin/sqlite3 -ascii "$sqltpth" 'select ZEND from ZSESSION;' 2> /dev/null)" ]
then
    >&2 /bin/echo "attempting to send $compnam.json to IT Support..."
    /usr/bin/osascript \
        -e 'tell application "System Events"' \
        -e 'keystroke "s" using {shift down, command down}' \
        -e 'delay 2' \
        -e 'keystroke "/"' \
        -e 'delay 2' \
        -e 'keystroke "private"' \
        -e 'delay 2' \
        -e 'keystroke "/"' \
        -e 'delay 2' \
        -e 'keystroke "tmp"' \
        -e 'delay 2' \
        -e 'keystroke return' \
        -e 'delay 2' \
        -e 'click button "Save" of window "Save" of application process "Mac Evaluation Utility"' \
        -e 'end tell'
fi

/usr/bin/open '/Applications/Microsoft Outlook.app'
/bin/sleep 3

/usr/bin/osascript \
    -e 'tell application "Microsoft Outlook"' \
    -e "set theFile to \"$rprtpth\" as POSIX file" \
    -e "set theMessage to make new outgoing message with properties {subject:\"Mac Evaluation Utility $compnam\"}" \
    -e 'tell theMessage' \
    -e 'make new to recipient with properties {email address:{address:"someone@somewhere.com"}}' \
    -e 'tell theMessage' \
    -e 'make new attachment with properties {file:theFile}' \
    -e 'send theMessage' \
    -e 'end tell' \
    -e 'end tell' \
    -e 'end tell'

/usr/bin/pkill -ail "Mac Evaluation"
>&2 /bin/echo "exiting..."
```
