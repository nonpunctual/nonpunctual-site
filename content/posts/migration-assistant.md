---
title: "Migration \"Assistant\""
date: 2024-05-15
tags: ["druva", "launchd", "macos", "script", "software"]
---

{{< notice info >}}

These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform. 

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

{{< notice note >}}

When I wrote this script, it was intentionally for blocking Migration Assistant.app - I re-wrote it so any app can be blocked this way just by setting the variables at the top. 

All you really need to know is the `CFBundleName` or `CFBundleDisplayName` (assuming that for most apps those match.) The "process identifier" (`prcidnt`) variable can really be anything (follow the given format & make it something unique.) Also, customize the messagaing.

{{< /notice >}}

<br/>

-----

So, here's a thing:

Have you ever wished your users didn't try to use Migration Assistant? Maybe you have something better (Alectrona Migrator, Druva - that's what I used to use & it was the best...) 

But, maybe, you just want to stop them. Yes, you can use Restricted Software. Maybe you've deployed [Santa](https://fleetdm.com/articles/deploy-santa-with-fleet-gitops-and-skip-the-sync-server). But, maybe, for some reason, the internet has led you here & ta da:

```bash
#!/bin/bash

# set -x
# trap read debug

# macos-block-app.sh @2024 Fleet Device Management
# Brock Walters (brock@fleetdm.com)


#####################################
### variables: populate as needed ###
#####################################

# The text that appears on the title bar of the AppleScript dialog window
apscttl='Migration Assistant Blocked'

# The text that appears in the AppleScript dialog window
apsctxt='Migration Assistant is blocked on this computer. Please contact your administrator for help.'

# The name of the app to block as it appears in Finder
blckapp='Migration Assistant'

# The 'identifier' for the 'block', e.g., blk.migr.asst
prcidnt='blk.migr.asst'

###########################
### DO NOT MODIFY BELOW ###
###########################


# paths
plstpth="/Library/LaunchDaemons/com.$prcidnt.plist"
scptpth="/opt/$prcidnt.sh"


# write out blocking script
/bin/cat << EOF > "$scptpth"
#!/bin/sh

if /usr/bin/pgrep -ail "$blckapp"
then
    /usr/bin/pkill -ail "$blckapp"
    /usr/bin/osascript -e "display dialog \"${apsctxt}\" buttons {\"OK\"} default button 1 with title \"${apscttl}\" with icon file \"System:Library:CoreServices:CoreTypes.bundle:Contents:Resources:AlertStopIcon.icns\""
fi
EOF
/bin/chmod 755 "$scptpth"
/usr/sbin/chown 0:0 "$scptpth"


# write out launch daemon
/bin/cat << EOF > "$plstpth"
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
      <string>$prcidnt</string>
    <key>ProgramArguments</key>
      <array>
        <string>/bin/sh</string>
        <string>$scptpth</string>
      </array>
    <key>RunAtLoad</key>
      <true/>
    <key>KeepAlive</key>
      <true/>
  </dict>
</plist>
EOF
/bin/chmod 644 "$plstpth"
/usr/sbin/chown 0:0 "$plstpth"
/usr/bin/plutil -convert binary1 "$plstpth"


# start launch daemon
/bin/launchctl bootstrap system "$plstpth"
```

<br/><small>Source: https://community.jamf.com/general-discussions-2/migration-assistant-32668</small>
