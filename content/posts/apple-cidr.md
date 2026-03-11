---
title: "Apple CIDR"
date: 2023-08-21
tags: ["bash","cidr","ipv4","jamf api","jamf nation","script"]
---

{{< notice info >}}

These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----
<br/>

It's been a while so here is something that you may never need, or, you have something clever that you already use for this & I am too dumb to know about it, or, maybe, someone will find it useful.

2 things actually:

1) I have arrived at what I feel is the bare minimum "boilerplate" code for a shell script intended to do something with a Jamf API token so, that's 1st. 

~~It's as short as I can make it,~~ it has error checking, it does not store a password in the script (it's interactive) & it has a function that can be called at any point to ensure the key is valid or invalid. Feel free to copy, it's been working well for me.

2) Remember that time you wanted to input new Network Segments & your fancy network guy gave you a single column of CIDR ranges? This converts them to integer IPv4 ranges with bitwise arithmetic.

- The input is assumed to be a `.csv` with the Jamf Network Segment index in column 1.
- Use `0` if it's a net-new Network Segment or the Network Segment index (id) if you are updating an existing record.
- Put the network name (label) in column 2 & the CIDR range in column 3.
- You can swap all of that around to make it match your `.csv` data if you prefer.

```bash
!/bin/bash
#shellcheck disable=SC2207



# This script:
#   - takes as input a .csv file with rows containing CIDR IPv4 address ranges
#   - converts them to integer ranges
# - creates or updates a Network Segment record for each .csv row via the Jamf API

# E.g., in the .csv:

# 0,foo,192.0.2.0/24
# 1,bar,10.0.0.0/18

# Column 1 is a Jamf Network Segment record index (or 0).
# Column 2 is the Network Name.
# Column 3 is a valid CIDR IPv4 range.
# Using 0 as the jamf network segment "index" means that a new network segment will be created.
# Any other index means a record already exists for the network segment. It will be updated with the name & range in the .csv row at that index.



### data ###

datafile='/private/tmp/building.subnet.data.csv' 

##########################
### DO NO MODIFY BELOW ###
##########################



### jamf API ###
jamfurl='https://some.company.com:8443' 
jamfapi="$jamfurl/api/v1" 
jamfchk="$jamfurl/healthCheck.html" 
jamfrsc="$jamfurl/JSSResource" 
osaauth='Jamf Pro Authentication' 
osapswd='Please enter API password:' 
osauser='Please enter API username:' 
usrcrnt="$(/usr/bin/stat -f %Su /dev/console)" 

tknchck()
{ 
  tknhttp="$(/usr/bin/curl -LSs -X GET -H 'Accept: application/json' -H "Authorization: Bearer $tkninit" "$jamfapi/auth" -o '/dev/null' -w "%{http_code}")"; /bin/sleep 1.5
  
  case "$tknhttp" in
    '200' ) tknvldt='OK' ;; 
    '401' ) tknvldt='Unauthorized' ;; 
        * ) tknvldt="$tknhttp" ;; 
  esac  
  >&2 printf "Jamf API Token Status: %s\n" "$tknvldt" 
}

>&2 printf "\nChecking Jamf availability & validating admin credentials...\n"
if /usr/bin/curl -LSs "$jamfchk" --connect-timeout 60 | /usr/bin/grep -q '\[\]'
then 
  apiuser="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript -e "display dialog \"$osauser\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\"" -e 'set theName to text returned of result')"
  apipswd="$(/usr/bin/sudo -u "$usrcrnt" /usr/bin/osascript -e "display dialog \"$osapswd\" default answer \"\" buttons {\"Ok\"} default button 1 with Title \"$osaauth\" with hidden answer" -e 'set thePswd to text returned of result')"
  b64auth="$(printf "%s" "$apiuser:$apipswd" | /usr/bin/iconv -t ISO-8859-1 | /usr/bin/base64)"
else 
  >&2 printf 'The Jamf server may not be available. Please try again. Exiting...\n'; exit 
fi 

>&2 printf "Obtaining Jamf API token...\n"
tkninit="$(/usr/bin/plutil -extract 'token' raw - <<< "$(/usr/bin/curl -LSs -X POST -H 'Accept: application/json' -H "Authorization: Basic $b64auth" "$jamfapi/auth/token")")"; /bin/sleep 1.5; echo; tknchck 

if [ "$tknvldt" != 'OK' ]
then
  >&2 printf 'Please ensure that your Jamf credentials are correct & try again. Exiting...\n'; exit 
fi 



### functions ###
cidrfnc()
{ 
  ntwkaddr="$(echo "$1" | /usr/bin/awk -F '/' '{print $1}')" 
  ntwkleng="$(echo "$1" | /usr/bin/awk -F '/' '{print $2}')" 
  IFS='.' read -r i1 i2 i3 i4 <<< "$ntwkaddr" 
  sbntmask=$(( 0xFFFFFFFF << (32 - ntwkleng) ))
  cnvrntwk=$(( (i1 << 24) + (i2 << 16) + (i3 << 8) + i4 ))
  cnvrfrst=$(( cnvrntwk & sbntmask ))
  cnvrlast=$(( cnvrfrst | (~sbntmask & 0xFFFFFFFF) ))
  ntwkfrst="$(printf '%d.%d.%d.%d' "$((cnvrfrst >> 24))" "$(( (cnvrfrst >> 16) & 255 ))" "$(( (cnvrfrst >> 8) & 255 ))" "$((cnvrfrst & 255))")"
  ntwklast="$(printf '%d.%d.%d.%d' "$((cnvrlast >> 24))" "$(( (cnvrlast >> 16) & 255 ))" "$(( (cnvrlast >> 8) & 255 ))" "$((cnvrlast & 255))")" 
  >&2 printf "%s - %s\n" "$ntwkfrst" "$ntwklast"
} 

datafnc()
{
  jamfindx="$(echo "$1" | /usr/bin/awk -F ',' '{print $1}')" 
  jamfname="$(echo "$1" | /usr/bin/awk -F ',' '{print $2}')" 
  jamfntwk="$(echo "$1" | /usr/bin/awk -F ',' '{print $3}')" 
  jamfstrg="$jamfrsc/networksegments/id/$jamfindx" 

  case "$jamfindx" in 
    0 ) httpmeth='POST'; >&2 printf "\nNew Network Segment!\n" ;; 
    * ) httpmeth='PUT'; >&2 printf "\nNetwork Segment Index: %s\n" "$jamfindx" ;; 
  esac 
  >&2 printf "Network Name: %s\nConverting Range: %s = " "$jamfname" "$jamfntwk"
} 



### operations ###
IFS=$'\n'
arrdata=($(/bin/cat "$datafile"))

>&2 printf "\nReading CIDR network ranges from %s\n" "$datafile"
for i in "${arrdata[@]}"
do 
  datafnc "$i"; cidrfnc "$jamfntwk"
  /usr/bin/curl -LSs -X "$httpmeth" -H 'Content-type: application/xml' -H "Authorization: Bearer $tkninit" -d "<network_segment><name>$jamfname</name><starting_address>$ntwkfrst</starting_address><ending_address>$ntwklast</ending_address></network_segment>" "$jamfstrg"; /bin/sleep 1.5; echo
  unset cnvrfrst cnvrlast cnvrntwk httpmeth jamfindx jamfname jamfntwk jamfstrg ntwkaddr ntwkfrst ntwklast ntwkleng sbntmask
done 

>&2 printf "\nOperations complete. Invalidating Jamf API token...\n"
/usr/bin/curl -LSs -X POST -H 'Accept: application/json' -H "Authorization: Bearer $tkninit" "$jamfapi/auth/invalidate-token"; /bin/sleep 1.5; echo; tknchck
```
