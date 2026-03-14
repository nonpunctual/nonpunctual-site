---
title: "Dynamic Token Of Static Appreciation"
date: 2023-05-08
tags: ["csv", "jamf api", "script", "static groups", "xml", "xmllint"]
---

{{< notice info >}}

**NOTE:** These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----

It's been a while since I posted anything, so, for (future) me & for you I am posting this script that dynamically updates Static Group membership.

You might be saying to yourself, "Dynamically updating a Static Group is impossibly dumb, because, that's what Smart Groups are for...", but, here is my dilemma:

*I need to assess an endpoint state for which there are no good client-side attributes to collect.*

- In this particular case (if you must know, Microsoft modern vs. basic authentication) there is no definitive data on endpoints in my environment that accurately reflects the sheer number of highly-variable user login behaviors. 

- It's not that it's too hard, it just can't be accurately done (yes, I've talked to a person I actually know at Microsoft... [Notes from the field: Does Outlook for Mac insist on using Basic Authentication?](https://techcommunity.microsoft.com/t5/exchange-team-blog/notes-from-the-field-does-outlook-for-mac-insist-on-using-basic/ba-p/3629510)).

- The user login data I need (i.e., email addresses that are using basic auth) is available, however, only in the server-side Exchange logs. 

Because our Messaging team can ship me that log data I can use it to create Static Groups. When that data is updated I can dynamically change the membership of my designated Static Group, thus, Dynamic Static Groups™

I think it's a pretty cool idea with lots of potential use cases. The best way to do this isn't necessarily with a bash script but with something like AWS Lambda or an orchestration tool like [Tines](https://www.tines.com/). 

Here's my script anyways which also happens to include the boilerplate stuff I intend to use in the future for Jamf API token authentication (which was created using ideas from Bill Smith's fantastic Jamf blog post on the topic: [How to Convert Classic API Scripts to Use Bearer Token](https://community.jamf.com/t5/tech-thoughts/how-to-convert-classic-api-scripts-to-use-bearer-token/ba-p/273910)).

Enjoy!

```bash
#!/bin/bash

# functions & strings & variables
authcsv='/Users/Shared/basic.auth.csv'
jamfurl='https://jss.blah.com:8443'
jamfapi="$jamfurl/api/v1"
jamfchk="$jamfurl/healthCheck.html"
jamfrsc="$jamfurl/JSSResource"
jamfstg="$jamfrsc/computergroups/id/1234"
osaauth='Jamf Pro Authentication'
osapswd='Please enter your Jamf API password:'
osauser='Please enter your Jamf API username:'
usrcrnt="$(/usr/bin/stat -f %Su /dev/console)"

tknchck(){
    tknhttp="$(/usr/bin/curl -LSs -X GET \
        -H 'Accept: application/json' \
        -H "Authorization: Bearer $tkninit" \
        "$jamfapi/auth" -o '/dev/null' -w "%{http_code}")"
    /bin/sleep 1
    case "$tknhttp" in
        '200' ) tknvldt='OK' ;;
        '401' ) tknvldt='Unauthorized' ;;
    esac
    printf "Jamf API Token Status: %s\n" "$tknvldt"
}

printf "\nChecking Jamf availability & validating admin credentials...\n"

if /usr/bin/curl -LSs "$jamfchk" --connect-timeout 60 | /usr/bin/grep -q '\[\]'
then
    apiuser="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript \
        -e "display dialog \"$osauser\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\"" \
        -e 'set theName to text returned of result')"
    apipswd="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript \
        -e "display dialog \"$osapswd\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\" with hidden answer" \
        -e 'set thePswd to text returned of result')"
    b64auth="$(/bin/echo -n "$apiuser:$apipswd" | /usr/bin/base64)"
else
    printf 'The Jamf server may not be available. Please try again. Exiting...\n'; exit
fi

printf "Obtaining Jamf API token...\n"
tkninit="$(/usr/bin/plutil -extract 'token' raw - \
    <<< "$(/usr/bin/curl -LSs -X POST \
        -H 'Accept: application/json' \
        -H "Authorization: Basic $b64auth" \
        "$jamfapi/auth/token")")"
/bin/sleep 1
tknchck

if [ "$tknvldt" != 'OK' ]
then
    printf 'Please ensure that your Jamf credentials are correct & try again. Exiting...\n'; exit
fi

printf "\nCollecting all computers for each email address...\n"
IFS=$'\n'
j=0
emladdr=($(/usr/bin/sort "$authcsv" | /usr/bin/uniq | /usr/bin/sed 's/@/%40/g'))

for i in "${emladdr[@]}"
do
    tnumarr+=($(/usr/bin/curl -LSs -X GET \
        -H 'Accept: application/xml' \
        -H "Authorization: Bearer $tkninit" \
        "$jamfrsc/users/email/$i" \
        | /usr/bin/xmllint --xpath "//computers/computer/name/text()" - 2> /dev/null))
    /bin/sleep 0.5
    if [ -z "${tnumarr[$j]}" ]
    then
        tnumerr+=($(echo "$i" | /usr/bin/sed 's/%40/@/g')); continue
    fi
    printf "$((j+1))) %s: \n%s\n" "$(echo "$i" | /usr/bin/sed 's/%40/@/g')" "${tnumarr[$j]}"
    ((j++))
done

for j in "${tnumarr[@]}"
do
    apidata+="$(/bin/echo -n "<computer><name>$j</name></computer>")"
done

printf "\nNo computers found for: %s\n\n" "${tnumerr[*]}"
read -p '> Pausing to validate output above. Press RETURN to continue or CONTROL + C to cancel...'

printf 'Deleting current Static Group members...\n'
/usr/bin/curl -LSs -X PUT \
    -H 'Content-type: application/xml' \
    -H "Authorization: Bearer $tkninit" \
    -d '<computer_group><computers></computers></computer_group>' \
    "$jamfstg"
/bin/sleep 2; echo

printf 'Verifying deletions...\n'
/usr/bin/curl -LSs -X GET \
    -H 'Accept: application/xml' \
    -H "Authorization: Bearer $tkninit" \
    "$jamfstg" | /usr/bin/xmllint -format -
/bin/sleep 2; echo

printf 'Updating Static Group membership...\n'
/usr/bin/curl -LSs -X PUT \
    -H 'Content-type: application/xml' \
    -H "Authorization: Bearer $tkninit" \
    -d "<computer_group><computers>$apidata</computers></computer_group>" \
    "$jamfstg"
/bin/sleep 2; echo

printf 'Verifying additions...\n'
/usr/bin/curl -LSs -X GET \
    -H 'Accept: application/xml' \
    -H "Authorization: Bearer $tkninit" \
    "$jamfstg" | /usr/bin/xmllint -format -
/bin/sleep 2; echo

printf "Operations complete. Invalidating Jamf API token...\n"
/usr/bin/curl -LSs -X POST \
    -H 'Accept: application/json' \
    -H "Authorization: Bearer $tkninit" \
    "$jamfapi/auth/invalidate-token"
/bin/sleep 1
tknchck
```

<br/><small>Source: https://community.jamf.com/t5/jamf-pro/dynamic-token-of-static-appreciation/m-p/290667</small>
