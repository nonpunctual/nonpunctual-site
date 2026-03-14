---
title: "I've Got the Power"
date: 2022-03-15
tags: ["apple silicon", "extension attribute", "intel", "jamf nation", "jamf pro", "macos", "plistbuddy", "script"]
---

{{< notice info >}}

**NOTE:** These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

Jamf Nation post - https://community.jamf.com/general-discussions-2/i-ve-got-the-power-27003

{{< /notice >}}

<br/>

-----

<br/>

An Extension Attribute script for Mac portable battery health:

```bash
#!/bin/bash
#shellcheck disable=SC2207

if ! /usr/sbin/sysctl -n hw.model | /usr/bin/grep -q -i 'book'
then
    echo "<result>no</result>"; exit
fi

IFS=$'\n'
pwrdata=($(/usr/libexec/PlistBuddy \
    -c "print 0:_items:0:sppower_battery_health_info:sppower_battery_health" \
    -c "print 0:_items:0:sppower_battery_health_info:sppower_battery_cycle_count" \
    -c "print 0:_items:0:sppower_battery_health_info:sppower_battery_health_maximum_capacity" \
    -c "print 0:_items:0:sppower_battery_charge_info:sppower_battery_max_capacity" \
    -c "print 0:_items:0:sppower_battery_model_info:sppower_battery_manufacturer" \
    -c "print 0:_items:0:sppower_battery_model_info:sppower_battery_firmware_version" \
    -c "print 0:_items:0:sppower_battery_model_info:sppower_battery_hardware_revision" \
    -c "print 0:_items:0:sppower_battery_model_info:sppower_battery_cell_revision" \
    /dev/stdin <<< "$(/usr/sbin/system_profiler -xml SPPowerDataType)" 2> /dev/null))

result(){
    echo "Health = ${pwrdata[0]}"
    echo "Cycle Count = ${pwrdata[1]}"
    
    if [[ "${pwrdata[2]}" == *'%'* ]]
    then
        echo "Max Capacity = ${pwrdata[2]}"
    else
        echo "Max Capacity = ${pwrdata[2]} mAh"
    fi

    if [ "$(/usr/sbin/sysctl -in hw.optional.arm64)" = 1 ] && /usr/sbin/sysctl -n machdep.cpu.brand_string | /usr/bin/grep -q 'Apple' && /usr/bin/uname -v | /usr/bin/grep -q 'ARM64'
    then
        i=5 # Apple Silicon
    else
        i=6 # Intel
        echo "Manufacturer = ${pwrdata[i-3]}"
    fi

    echo "Firmware Version = ${pwrdata[i-2]}"
    echo "Hardware Revision = ${pwrdata[i-1]}"
    echo "Cell Revision = ${pwrdata[i]}"
}

echo "<result>$(result)</result>"
```

Some notes:

- I want to separate Mac portables from non-portables without need for a Smart Group.
  - That's what the 1st section does.
  - Every Mac portable since the beginning of time has had "Book" in the name. 
  - This is how I like to find them. You may like another way.
- Intel Mac portables & Apple Silicon Mac portables have different battery reporting. 
  - This sucks & I found out only after version 1 of this EA.
- Not everyone is aware that you can pass multiple arguments to a single `PlistBuddy` command.
  - Although, `zsh` can be a little crybaby about how much can be parsed in 1 stream...
- I wanted the data to appear in a multi-line format.
    - I am a multi-line Extension Attribute fan.
    - Always have been, always will be. 
    - When some version of Jamf broke my multi-line EAs a few years back, I died a little inside.
- I also wanted customized, human-readable labels for the attributes.
  - This craziness with the conditionals at the end handles the labels, order & the cases (Intel vs. Apple Silicon).

Here is the result in Jamf. (See how one has `mAh` & the other has `%` capacity & a different number of lines?)

{{< figure src="/images/battery1.webp" alt="" >}}

{{< figure src="/images/battery2.webp" alt="" >}}

Anyway, that's all. I know there are fancier ways to do this. OSQuery, Fleet, etc. I like EAs.

-----

**PS. Update:**

Fleet (which uses OSQuery) was not reporting this correctly. 🙂 I believe they have addressed this based on my EA (fun!)

I am reporting this in Feedback Assistant — I don't understand why Apple stopped reporting battery manufacturer...
