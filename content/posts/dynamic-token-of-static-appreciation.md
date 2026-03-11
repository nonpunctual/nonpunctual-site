---
title: "Dynamic Token Of Static Appreciation"
date: 2023-12-21
tags: ["jamf api","jamf nation","logging","script","zsh"]
---

{{< notice info >}}

These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----
<br/>

As it is the holiday season, I wanted to offer up a little gratitude in the form of boilerplate code related to topics that come up in the mac admins #scripting & #bash channels again & again. I hope people find it here & find it useful.

1) Logging is always a hot topic. "How do I create a log for my script?" "Where should I write log output?" etc. Without saying this solution is definitive, the script below has a pattern that I have used over & over. In fact, I paste this block exactly into any script I create that needs logging. It uses `tee` along with `exec` & with a named pipe. Why? Well, I find this works in all situations to capture all foreground & background processes executed by the script & it creates a dedicated stream / file for the log output regardless of what the script is trying to do.

2) Jamf API. I said in an earlier post I had come up with the shortest, bestest version of making an API call with token auth. I was wrong. I made it shorter & betterer.

This script does not do anything remarkable - it removes Network Segments at their endpoint by name. This is also something that seems to come up repeatedly as the Jamf API docs could be a bit clearer on this subject. Using the object ID is straightforward but I think having a good example of using name as the identifier should be useful to some of you. Happy Whatever!

**PS.** The input should just be a single column `.csv` file of single quoted Network Segment names & you put the path to that file in the `network_data_csv` variable at the top of the script.

```zsh
#!/bin/zsh


# api.delete.network.segments @2023
# author: brock_walters@blah.com
# created: 2023-11-07
# version: 9

# NOTE: This script is currently set in "test mode". To execute actual deletions the line in the script below starting with:
#
# /usr/bin/curl -LSs -X DELETE ...
#
# must be uncommented.


# environment & data
network_data_csv='/Users/Shared/building.subnet.data.csv'
jamf_environment='https://server.blah.com:8443'


###########################
### DO NOT MODIFY BELOW ###
###########################


# logging
cpuname="$(/usr/sbin/scutil --get ComputerName)"
srlnmbr="$(/usr/libexec/PlistBuddy -c 'print 0:serial-number' /dev/stdin <<< "$(/usr/sbin/ioreg -ar -d 1 -k 'IOPlatformSerialNumber')")"
usrcrnt="$(/usr/bin/stat -f %Su /dev/console)"

logexec="$(/usr/bin/basename "$0")"
logpath="/private/tmp/${logexec%.*}.log"
logpipe="/private/tmp/${logexec%.*}.pipe"

/usr/bin/mkfifo "$logpipe"
/usr/bin/tee -a < "$logpipe" "$logpath" &
exec &> "$logpipe"
printf "$(/bin/date "+%Y-%m-%dT%H:%M:%S") [START] logging %s\ncomputer name: %s\nserial number: %s\ncurrent user: %s\n" "$logexec" "$cpuname" "$srlnmbr" "$usrcrnt" >> "$logpath"
logalrt(){ >&2 printf "$(/bin/date "+%Y-%m-%dT%H:%M:%S") [ALERT] %s\n" "$1" >> "$logpath" }
loginfo(){ >&2 printf "$(/bin/date "+%Y-%m-%dT%H:%M:%S") [INFO] %s\n" "$1" >> "$logpath" }
logexit(){ >&2 printf "$(/bin/date "+%Y-%m-%dT%H:%M:%S") [STOP] logging %s\n" "$logexec" >> "$logpath"; /bin/rm -rf "$logpipe"; /usr/bin/pkill -ail tee > /dev/null; exit }


# jamf api
jamfurl="$jamf_environment"
jamfapi="$jamfurl/api/v1"
jamfchk="$jamfurl/healthCheck.html"
jamfrsc="$jamfurl/JSSResource"
osaauth='Jamf Pro Authentication'
osapswd='Please enter your Jamf API password:'
osauser='Please enter your Jamf API username:'
apiuser="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript -e "display dialog \"$osauser\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\"" -e 'set theName to text returned of result')"
apipswd="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript -e "display dialog \"$osapswd\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\" with hidden answer" -e 'set thePswd to text returned of result')"
b64auth="$(printf "%s" "$apiuser:$apipswd" | /usr/bin/iconv -t ISO-8859-1 | /usr/bin/base64)"
tknauth="$(/usr/bin/plutil -extract 'token' raw - <<< "$(/usr/bin/curl -LSs -X POST -H 'Accept: application/json' -H "Authorization: Basic $b64auth" "$jamfapi/auth/token")")"; /bin/sleep 1.5

api_chk(){ printf "\nChecking jamf api...\n" ; }
api_no(){ printf "The jamf server may not be available. Please try again. Exiting...\n" ; }
api_yes(){ printf "OK.\n" ; }
auth_chk(){ printf "\nValidating jamf api credentials...\njamf url: %s\njamf api user: %s\n" "$jamfurl" "$apiuser" ; }
auth_no(){ printf "Please ensure that your credentials are correct & try again. Exiting...\n\n" ; }
delete_no(){ printf "%s Network Segment Not Found.\n\n" "$ntwkstr" ; }
delete_yes(){ printf "Deleting:\n" ; }
mtch_req(){ printf 'The server has not found anything matching the request.' ; }
ntwk_csv(){ printf "\nReading Network Segment names from %s\n\n" "$network_data_csv" ; }
ops_done(){ printf "Operations complete.\nInvalidating jamf api token...\n" ; }
token_no(){ printf "jamf api token status: UNAUTHORIZED\n" ; }
token_yes(){ printf "jamf api token status: OK\n" ; }
tknchk(){
  tknhttp="$(/usr/bin/curl -LSs -X GET -H 'Accept: application/json' -H "Authorization: Bearer $tknauth" "$jamfapi/auth" -o '/dev/null' -w "%{http_code}")"; /bin/sleep 1.5
  case "$tknhttp" in
    '200' ) token_yes ;;
    '401' ) token_no ;;
    * ) echo "$tknhttp" ;;
  esac
}

api_chk
if /usr/bin/curl -LSs "$jamfchk" --connect-timeout 60 | /usr/bin/grep -q '\[\]'
then
  api_yes
else
  api_no; logexit
fi

auth_chk; tknchk
if [ "$tknhttp" != '200' ]
then
  logalrt; auth_no; logexit
fi


# operations
IFS=$'\n'; arrdata=($(/bin/cat "$network_data_csv"))
ntwk_csv
for i in "${arrdata[@]}"
do
  ntwkstr="$(echo "$i" | /usr/bin/sed 's/,//g')"
  ntwkurl="$(echo "$i" | /usr/bin/sed 's/ /%20/g;s/,//g')"
  ntwkvar="$(/usr/bin/curl -LSs -X GET -H 'Accept: application/json' -H "Authorization: Bearer $tknauth" "$jamfrsc/networksegments/name/$ntwkurl"; /bin/sleep 1.5; echo)"
  if echo "$ntwkvar" | /usr/bin/grep -q "\"name\":\"$ntwkstr\""
  then
    logalrt; delete_yes; echo "$ntwkvar" | /usr/bin/json_pp -t json
    #/usr/bin/curl -LSs -X DELETE -H 'Accept: application/json' -H "Authorization: Bearer $tknauth" "$jamfrsc/networksegments/name/$ntwkurl"; /bin/sleep 1.5; echo >> "$logpath"
    unset ntwkstr ntwkurl ntwkvar
  elif echo "$ntwkvar" | /usr/bin/grep -q "$(mtch_req)"
  then
    loginfo; delete_no
    unset ntwkstr ntwkurl ntwkvar; continue
  else
    logalrt 'Error.'; echo >> "$logpath"
    unset ntwkstr ntwkurl ntwkvar; continue
  fi
done
ops_done
/usr/bin/curl -LSs -X POST -H 'Accept: application/json' -H "Authorization: Bearer $tknauth" "$jamfapi/auth/invalidate-token"; /bin/sleep 1.5; tknchk; echo; logexit
```
