---
title: "A Light Unto My Xpath"
date: 2021-12-22
tags: ["curl", "jamf api", "jamf nation", "script", "xml", "xmllint", "xpath"]
---

{{< notice info >}}

**NOTE:** These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

{{< notice note >}}


**From the legendary [Armin Briegel](https://scriptingosx.com/about/) in the comments:**

>There seems to be a change to the `xmllint` command behavior in Ventura. Now this:
>
>```
>echo $xml | xmllint --xpath '/policies/policy/id/text()' -
>```
>
>returns a new line separated list of all the ids.

-----

**Reply:**

Hi Armin -

It looks like the `xmllint` version in macOS has indeed been updated:

macOS Monterey:

```sh
% sw_vers -productVersion 12.6.2 % xmllint --version xmllint: using libxml version 20904
```

macOS Ventura:

```sh
% sw_vers -productVersion 13.1 % xmllint --version xmllint: using libxml version 20913
```

The function for getting ids above still works the same (just tried it on Ventura).

But now it's unnecessary on macOS 13 & later because `xmllint` is no longer dumb about output & handles the counting + looping + concatenating behavior for you. This is a good thing!

So, for Monterey & prior (or in Ventura if you don't feel like changing anything...):

```sh
apiuser='someuser'                                                                                                                                 
apipswd='password'                                                                                                                                 
jamfurl='https://somejamfurl.whatever:8443/JSSResource' 

jamfids()
{                                                                                                                                        
    apidata=$(curl -LSs -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/$1")                                                  
    arrsize=$(echo "$apidata" | xmllint --xpath "//size/text()" -)
    
    for ((i=0;i<=$arrsize;i++))
    {
        echo "$apidata" | xmllint --xpath "concat(//$2[$i]/id/text(),' ')" - ;
    }
}

policy_ids=($(jamfids policies policy))
configuration_profile_ids=($(jamfids osxconfigurationprofiles os_x_configuration_profile))
```

For Ventura & beyond I still like the idea of a (simplified) generalized function for getting object ids:

```sh
jamfids()
{
    curl -LSs -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/$1" | xmllint --xpath "//$2/id/text()" - 
}

Thanks!
```

{{< /notice >}}

<br/>

-----

`xpath` is not super intuitive. Maybe I can clear up 1 tiny little thing here...

- First, a resource I go back to again & again: [Xpath cheatsheet](https://devhints.io/xpath)
- Second, `xpath` is a querying language. 

It can look complicated, but, a way of simplifying it is to think of the syntax as an analogy: exposing `xml` data as a "file system" & the thing you want to "get" as a "file". 

- Your `xpath` query is the "file path".
- Elements, in the analogy, are "folders" with stuff in them: one thing, many things, etc. 

E.g., you might want all IDs for some object in Jamf Pro (let's say Policies) so maybe you tried:

```bash
curl -Ss -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/policies" | xmllint --xpath "//id" - | tr '</id>' '\n'
```

Not great because you are stripping the `xml` data structure & `tr` often outputs strange results.

Then maybe you looked at the [Xpath cheatsheet](https://devhints.io/xpath) or took a Jamf course & discovered the magical `text()` command which strips `xml` tags!

```bash
curl -Ss -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/policies" | xmllint --xpath "//id/text()"
```

But you get output something like:

```
11971190119211931194119512676531279103458411133271180111483812706051283583632...
```

Why? Because `xpath` is simply doing what you asked: returning data where the tag name is equal to "id". Even though we thought we would get the "items" in this "file path", we did & we didn't.

So, then you tried this: [Jamf API xmllint --xpath Question](https://community.jamf.com/t5/jamf-pro/jamf-api-xmllint-xpath-question/m-p/243355) — smooshed output problem solved, but, it's still stripping the data structure.

Don't be sad. `xpath` doesn't iterate for you but it does understand the index of each member in each element in the data. The `xpath` command needed to get delimited output is (weirdly? not weirdly?) `concat`:

```bash
apidata=$(curl -sS -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/policies")
arrsize=$(echo "$apidata" | xmllint --xpath "//size/text()" -)
for ((i=0;i<=$arrsize;i++))
{
    echo "$apidata" | xmllint --xpath "concat(//policy[$i]/id/text(),' ')" -
}
```

- The 1st line gets all the policy data.
- The 2nd line returns the "size" of the array, i.e., the total number of policies (handy that pretty much every object in Jamf Pro has this...)
- The for loop uses the size as a counter & it uses `concat` to concatenate a space to each ID (i.e., after each loop iteration)

The result is a space-delimited set of object IDs. Now you can generalize this into a function for getting the IDs of any Jamf Pro object:

```
1197 1190 119 211 931 1941 195 1267 653 12...
```

- `$1` = the endpoint label (`"policies"`)
- `$2` = the xml tag name (`"policy"`)

```sh
apiuser='someuser'
apipswd='password'
jamfurl='https://somejamfurl.whatever:8443/JSSResource'

jamfids() {
    apidata=$(curl -sS -X GET -H 'accept: application/xml' -u "$apiuser:$apipswd" "$jamfurl/$1")
    arrsize=$(echo "$apidata" | xmllint --xpath "//size/text()" -)
    for ((i=0;i<=$arrsize;i++))
    {
        echo "$apidata" | xmllint --xpath "concat(//$2[$i]/id/text(),' ')" - ;
    }
}

policy_ids=($(jamfids policies policy))
configuration_profile_ids=($(jamfids osxconfigurationprofiles os_x_configuration_profile))
```

Enjoy!

<br/><small>Source: https://community.jamf.com/general-discussions-2/a-light-unto-my-xpath-26316</small>
