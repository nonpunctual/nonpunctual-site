---
title: "JSON & the Arg-nauts"
date: 2024-01-05
tags: ["ad","date","dscl","jamf nation","json","script"]
---

{{< notice info >}}

**NOTE:** These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

{{< notice note >}}

**UPDATE:** ~~Something I thought would never happen has happened. Since the beta features are under NDA, I guess I can't really say what it is, but, IYKYK, &, it makes everything in this post mostly obsolete. :) Cheers.~~ 

So,  added `jq` to macOS. I do have a funny story about this... The version I will tell here in public is: I was told that would never happen & then it did! I (& a lot of other people) were very glad about this.

{{< /notice >}}

<br/>

-----

What are we talking about? Handling json in the macOS shell of your choosing. 

Why? 

- ~~Because there still isn't a great way to do this natively,~~ (see the update above...)
- It's extremely useful given how many macOS binaries & logs output JSON-ish data
- There are ways of working around this limitation.

Arg-nauts is an unforgivable pun, however, there are some real explorers in this area. I rounded some of them up here:

https://community.jamf.com/t5/jamf-nation/firefoxy/td-p/266970

I will try to be a bit more exhaustive this time...

- From the crafty Joel Brunner (you may know him as @brunerd...)

  - `jpt`: https://www.brunerd.com/blog/2022/02/01/jpt-1-0-can-deal-with-multiple-json-texts/
  - `ljt`: https://www.brunerd.com/blog/2022/02/22/ljt-1-0-0-a-little-json-tool-for-your-shell-script/

  I have used `ljt` in production. It's lightweight & works great. Once you're on Joel's blog, check out ALL of his `json` related posts!

- The post below inspired much ado in the Mac Admins Slack channels regarding parsing `json` data:

  https://paulgalow.com/how-to-work-with-json-api-data-in-macos-shell-scripts/

- Resultant was the `JSON-Shell-Tools-for-macOS` created by the ever-grinding @pico on the Mac Admins Slack channels:

  https://github.com/RandomApplications/JSON-Shell-Tools-for-macOS/tree/main

- A somewhat recent update to the `sqlite3` binary included `json` parsing within tables, but, it can also just be called to process data on the command line:

  https://sqlite.org/json1.html

The fact that AppleScript / `osascript` supports JavaScript is nice, but, in the context of these discussions I am wary of how much support is actually put behind AppleScript or `JXA` & I fear that at some point they just might go away. This makes figuring out how to use the [SQLite JSON capabilities](https://sqlite.org/json1.html) more interesting? Critical? (Apple uses SQLite pretty much everywhere & I would bet on that binary sticking around for a while...)

- Very smart people like Bart Reardon (of [swiftDialog](https://swiftdialog.app/) fame!) are already playing around with this:

  https://github.com/bartreardon/macscripts/blob/master/json_via_sqlite.sh

Finally, this meager contribution...

In a modern environment you should not have to bind your Macs to AD. But, maybe your 802.1x authentication depends on it, or, something else does & you're stuck with it. One benefit of this is: you get to use `dscl` to access data directly from your domain.

In one of my environments AD lockouts were used as a security measure against malicious login attempts. So, it's good to have a way to know if a user's AD account is locked out independently of "Hello, IT? None of my logins work."

This script does some weird things:

- It collects dscl data from AD as a plist
- Because of this the data can be CONVERTED to `json`
- This is done by stripping off the `dscl` keys & parsing it through `plutil` (which is [definitely not my invention](https://scriptingosx.com/2018/07/parsing-dscl-output-in-scripts/))
- The script also converts the date / time formats AD uses (3 different formats!) to ultra-human-readable dates about which there are no questions

The script can be run interactively on someone's computer or automatically to make any of the AD attributes collected useful as Jamf extension attributes / management data (i.e., not just for resolving AD lockouts.) 

But, I am definitely not saying you should bind to AD to do this. More, it's a benefit of binding that you get this data from `dscl` & here are some ideas about how to get it.

Enjoy!

```bash
#!/bin/bash


# ad.lockout.status @2023
# author: brock_walters@blah.com
# modified: 2023-07-05
# version: 7


####################################
### data & functions & variables ###
####################################


NC='\033[0m\n'; RED='\033[1;31m'; GREEN='\033[1;32m'; BLUE='\033[1;94m'
arrlocl=(NFSHomeDirectory PrimaryGroupID RecordName UniqueID UserShell)
arrsyst=(accountExpires badPasswordTime badPwdCount distinguishedName DNSName lastLogonTimestamp logonCount name operatingSystem operatingSystemVersion RealName RecordName RecordType SMBLogoffTime SMBLogonTime SMBPasswordLastSet whenChanged whenCreated)
arruser=(accountExpires AltSecurityIdentities AppleMetaNodeLocation AppleMetaRecordName AuthenticationAuthority badPasswordTime badPwdCount City Country department distinguishedName EMailAddress employeeID extensionAttribute1 extensionAttribute5 extensionAttribute8 JobTitle lastLogon lastLogonTimestamp lockoutTime logonCount mailNickname manager memberOf msExchCoManagedObjectsBL msExchHideFromAddressLists name objectCategory physicalDeliveryOfficeName proxyAddresses PrimaryNTDomain publicDelegatesBL RealName RecordName RecordType sAMAccountName showInAddressBook SMBGroupRID SMBPasswordLastSet State targetAddress uid uidNumber userAccountControl userPrincipalName whenChanged whenCreated)
crntsys="$(/usr/libexec/PlistBuddy -c "print 'General Info':'Computer Account'" /dev/stdin <<< "$(/usr/sbin/dsconfigad -show -xml)")"
crntusr="$(echo "show State:/Users/ConsoleUser" | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}')"
domenvr='/Active Directory/blah/All Domains'
domlocl='.'

objcfnc(){ /usr/bin/dscl -plist "$1" -read "/$2/$3" objectClass 2>&1 /dev/null | /usr/bin/grep -q "<string>$4</string>" ; }

dsclfnc(){ /usr/bin/dscl -plist "$1" -read "/$2/$3" | /usr/bin/plutil -convert json -o - - | /usr/bin/sed 's/dsAttrTypeStandard://g;s/dsAttrTypeNative://g' ; }

jsonfnc(){
for i in "${arrkeys[@]}"
do
    if /usr/bin/plutil -extract "$i" json -o - - <<< "$1" > /dev/null 2> /dev/null
    then
        dscltxt+="\"$i\":$(/usr/bin/plutil -extract "$i" json -o - - <<< "$1"),"
    else
        dscltxt+="\"$i\":\"null\","
    fi
done
jsonfmt="$(echo "$dscltxt" | /usr/bin/sed 's/,$//;s/./{&/;s/.*/&}/')"
}

# function to convert ASN.1 Generalized-Time, Unix & Windows NT date formats to extra super ultra human-readable format
timefnc(){
case "$1" in
    0                                                    ) printf 0 ;;
    "$(/usr/bin/grep -Ei 'error|invalid|null' <<< "$1")" ) printf 'null' ;;
    "$(/usr/bin/grep -E '^\d{10}$' <<< "$1")"            ) /bin/date -j -f '%s' "$1" '+%A %B %e %Y %I:%M:%S%p' | /usr/bin/sed 's/  / /' ;;
    "$(/usr/bin/grep -E '^\d{18}$' <<< "$1")"            ) /bin/date -j -f '%s' "$(($1/10000000-11644473600))" '+%A %B %e %Y %I:%M:%S%p' | /usr/bin/sed 's/  / /' ;;
    "$(/usr/bin/grep -E '^2\d{13}\.0Z$' <<< "$1")"       ) TZ="$(/usr/sbin/systemsetup -gettimezone | /usr/bin/awk '{print $NF}')" /bin/date -j -f '%Y%m%d%H%M%S%z' "$(echo "$1" | /usr/bin/sed 's/.0Z/-0000/')" '+%A %B %e %Y %I:%M:%S%p' | /usr/bin/sed 's/  / /' ;;
    *                                                    ) printf "The input did not match the conversion formats.\n" ;;
esac
}

uimdfnc(){
lockout="$(/usr/bin/plutil -extract 'lockoutTime' json -o - - <<< "$jsonfmt" | /usr/bin/sed 's/"//g;s/\]//;s/\[//')"
mngrstr="$(/usr/bin/plutil -extract 'manager' json -o - - <<< "$jsonfmt" | /usr/bin/sed 's/"//g;s/\]//;s/\[//;s/^CN=\([^,]*\).*/\1/')"

if echo "$mngrstr" | /usr/bin/grep -Eq '^([Uu][0-9a-zA-Z]{3}$)'
then
    manager="$(/usr/libexec/PlistBuddy -c 'print dsAttrTypeStandard\:RealName:0' /dev/stdin <<< "$(/usr/bin/dscl -plist '/Active Directory/blah/All Domains' -read /Users/"$mngrstr")")"
elif echo "$mngrstr" | /usr/bin/grep -Eiq 'error|invalid|null'
then
    manager='null'
else
    manager="$mngrstr"
fi

for j in name RealName RecordName RecordType logonCount badPwdCount
do
    valname+=($(/usr/bin/plutil -extract "$j" json -o - - <<< "$jsonfmt" | /usr/bin/sed 's/"//g;s/\]//;s/\[//'))
done

for k in badPasswordTime whenCreated whenChanged SMBPasswordLastSet SMBLogonTime SMBLogoffTime lastLogon lastLogonTimestamp
do
    valtime+=($(timefnc "$(/usr/bin/plutil -extract "$k" json -o - - <<< "$jsonfmt" | /usr/bin/sed 's/"//g;s/\]//;s/\[//')"))
done

printf "\n%s ATTRIBUTES:\n" "$objclss"; echo "$jsonfmt" | /usr/bin/json_pp -t json
if [ "$objclss" = 'USER' ] && [ "$lockout" != 0 ] && [ "$lockout" != 'null' ]
then
    printf "\n====== AD PASSWORD LOCKOUT STATUS ======\n\n${RED}LOCKED: %s${NC}" "$(timefnc "$lockout")"
elif [ "$objclss" = 'USER' ] && [ "$lockout" = 0 ]
then
    printf "\n====== AD PASSWORD LOCKOUT STATUS ======\n\n${GREEN}UNLOCKED${NC}"
fi

printf "\nUser Name = ${BLUE}%s${NC}Real Name = ${BLUE}%s${NC}Record Name = ${BLUE}%s${NC}Record Type = ${BLUE}%s${NC}Manager = ${BLUE}%s${NC}Logon Count = ${BLUE}%s${NC}Bad Password Count = ${BLUE}%s${NC}Bad Password Time = ${BLUE}%s${NC}Created = ${BLUE}%s${NC}Changed = ${BLUE}%s${NC}SMB Password Set = ${BLUE}%s${NC}SMB Logon Time = ${BLUE}%s${NC}SMB Logoff Time = ${BLUE}%s${NC}Last Logon = ${BLUE}%s${NC}Last Logon Timestamp = ${BLUE}%s${NC}" "${valname[0]}" "${valname[1]}" "${valname[2]}" "${valname[3]}" "$manager" "${valname[4]}" "${valname[5]}" "${valtime[0]}" "${valtime[1]}" "${valtime[2]}" "${valtime[3]}" "${valtime[4]}" "${valtime[5]}" "${valtime[6]}" "${valtime[7]}"
}


##################
### operations ###
##################


# check for root execution

if [ "$EUID" != 0 ]
then
    printf "\nThis script must be executed by the root user. Exiting...\n"; exit
fi


# check for interactive mode execution & collect input

if /usr/bin/basename "$0" | /usr/bin/grep -Eq '^pol.'
then
    >&2 printf "%s executed in non-interactive mode..." "$(/usr/bin/basename "$0")"; autoexc='enabled'
    case "$crntusr" in
        '_mbsetupuser|daemon|jssblah|nobody|mgmt|root' ) input="$(/usr/sbin/scutil --get ComputerName)" ;;
        *                                              ) input="$crntusr" ;;
    esac
elif [ -n "$1" ]
then
    input="$1"
else
    while true
    do
        printf '\e[8;40;200t'; clear; printf "To get AD data & lockout status: enter a user identity at the prompt.\n\nTo convert an ASN.1 Generalized-Time, Unix Epoch or Windows NT timestamp to ultra human-readable format: enter a date string at the prompt.\n\n"; read -rp "> " input
        case "$input" in
            '' ) echo; read -rp "Let's try that again... Press return to continue or control+C to cancel."; continue ;;
             * ) break ;;
        esac
    done
fi


# check if AD is reachable

if ! /usr/bin/dscl /Search read /Computers/"$crntsys" > /dev/null
then
    printf "\nActive Directory cannot be reached. Exiting...\n\n"; exit
fi


# validate input by checking dscl object class, collect data, convert to human-readable date if input is an integer, exit if input is invalid
# check if the ebtered username string exists in AD
# if so, determine the record's "object class"

IFS=$'\n'

if objcfnc "$domenvr" 'Users' "$input" 'user'
then
    objclss='USER'
    if [ "$input" = "$crntusr" ]
    then
        arrkeys=("${arrlocl[@]}"); jsonfnc "$(dsclfnc "$domlocl" 'Users' "$input")"; unset arrkeys
    fi
    arrkeys=("${arruser[@]}"); jsonfnc "$(dsclfnc "$domenvr" 'Users' "$input")"
elif objcfnc "$domenvr" 'Computers' "${input}$" 'computer'
then
    objclss='COMPUTER'
    tstinpt="$(echo "${input}$" | /usr/bin/tr '[:upper:]' '[:lower:]')"
    tstsyst="$(echo "$crntsys" | /usr/bin/tr '[:upper:]' '[:lower:]')"
    if [ "$tstinpt" = "$tstsyst" ]
    then
        arrkeys=("${arrlocl[@]}"); jsonfnc "$(dsclfnc "$domlocl" 'Users' "$crntusr")"; unset arrkeys
    fi
    arrkeys=("${arrsyst[@]}"); jsonfnc "$(dsclfnc "$domenvr" 'Computers' "${input}$")"
elif echo "$input" | /usr/bin/grep -Eq '[[:digit:]]'
then
    echo; timefnc "$input"; echo; exit
else
    printf "\nThe input did not match any existing record. Exiting...\n\n"; exit
fi


# do some stuff in non-interactive mode, convert time strings & display data in human-readable output if executed in interactive (i.e., user) mode
# non-interactive mode: run on a local launch daemon at some intervsal until a lock out is flagged. run recon to update smart group membership / webhook / run policy to notify user, etc. run a swiftDialog message?

if [ "$autoexc" = 'enabled' ]
then
    /usr/local/jamf/bin/jamf recon
else
    uimdfnc
fi
```

<br/><small>Source: https://community.jamf.com/general-discussions-2/json-the-arg-nauts-31917</small>
