---
title: "A Haiku On Regular Expressions"
date: 2023-06-18
tags: ["jamf nation","regex","script","zsh"]
---

{{< notice info >}}

These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----

For whatever reason, there was a day when the Mac admins slack was particularly goofy & there was a call for poems. I posted this haiku:

> I have a problem\
> Use regular expression\
> I have two problems

I can't take credit for the joke above, only for formulating it as a haiku. The origin goes back into the lore of computing. Here's a good summary: https://arstechnica.com/information-technology/2014/05/what-is-meant-by-now-you-have-two-problems/

The [regular expression syntax](https://en.wikipedia.org/wiki/Regular_expression) is an amazing human achievement. (Really!) 

The addition of `regex` values in Jamf Pro Smart Groups (which I have used a lot) probably made the idea of using this kind of matching more appealing, generally, to more people than ever. Eventually, most Mac admins encounter a situation where they believe `regex` may be the answer to their toughest problems. Often, it has something to do with date / time stamps, or, [version strings](https://gist.github.com/talkingmoose/2cf20236e665fcd7ec41311d50c89c0e).

This is wrong.

There are good ways of handling version string comparisons WITHOUT writing your own `regex`. You should take advantage of them. Don't introduce potential layers of failure into scripts that are difficult to troubleshoot or understand. Let the very experienced programmers who have done things like create shell environments & binaries for you handle the hard stuff...

I wrote about [one of these tools](/posts/googalogically-speaking) already: `sort --reverse --version-sort`.

Here's another I've been advised to use for a while now. I'm old & slow & stubborn, but, I finally got around to it...

```zsh
#!/bin/zsh


# notes:
# $4 Script Parameter in Jamf must be an application bundle id, e.g., com.apple.dt.Xcode
# $5 Script Parameter in Jamf must be an application bundle name, e.g., Xcode.app
# $6 Script Parameter in Jamf must be an application short version string, e.g., 14.2
# $7 Script Parameter in Jamf toggles a test mode. If populated with the string "test" the script will perform a "dry run" without removing files.


# variables (only populate these if Jamf Script Parameters are not used)
varbndl=''
varname=''
varvers=''


###############################
##### DO NOT MODIFY BELOW #####
###############################


# data
appbndl="${4:-$varbndl}"
appname="${5:-$varname}"
appvers="${6:-$varvers}"
tstmode="$7"

IFS=$'\n'

arrdata=($(/usr/bin/mdfind -0 "kMDItemCFBundleIdentifier = $appbndl" | /usr/bin/xargs -0 /usr/bin/mdls -name 'kMDItemVersion' -name 'kMDItemPath' | /usr/bin/sed 's/kMDItemPath = //g;s/kMDItemVersion = //g'))


# functions
appvr_ck(){ >&2 printf "\nChecking %s versions...\n" "$appname" ; }
appvr_no(){ >&2 printf "\nNo %s found. Exiting...\n" "$appname" ; }
appvr_ok(){ >&2 printf "\nFound %s\nversion = %s\nOk. Leaving %s in place...\n" "${arrdata[i-1]}" "${arrdata[i]}" "$appname" ; }
appvr_rm(){
    appkill="$(echo "${arrdata[i-1]}" | /usr/bin/sed 's/"//g')"
    >&2 printf "\nFound %s\nversion = %s\nInsecure version. Deleting %s...\n" "$appkill" "${arrdata[i]}" "$appkill"; /bin/rm -f -r "$appkill"
    >&2 printf "\nValidating deleted path: %s\n" "$appkill"; /bin/ls -ls "$appkill"
    exit 0
}
appvr_tm(){
    >&2 printf "\nFound %s\nversion = %s\nInsecure version. Deleting %s\n" "${arrdata[i-1]}" "${arrdata[i]}" "${arrdata[i-1]}"
    >&2 printf "\n!!! TEST MODE !!! Disabling test mode will remove: %s\n" "${arrdata[i-1]}"
}
autoload is-at-least
not_root(){ >&2 printf "\nThis script must be executed as the root user. Exiting...\n" ; }


# operations
if [ "$EUID" != 0 ]
then
    not_root; exit
fi

if [ -z "${arrdata[*]}" ]
then
    appvr_no; exit
fi

appvr_ck
for ((i=2;i<=${#arrdata[@]};i+=2))
{
    if is-at-least "$appvers" "${arrdata[i]}"
    then
        appvr_ok; continue
    else
        case "$tstmode" in
            'test' ) appvr_tm ;;
                 * ) appvr_rm ;;
        esac
    fi
}
```

This script is being used in my environment to handle removing older, insecure versions of installed apps.

There is nothing groundbreaking in it, but, it is taking advantage of a very, very nice `zsh` feature: `is-at-least` will do version string comparisons for you with its own built-in regex library (exactly what `sort --version-sort` is doing, by the way...)

If you enter `set -x` at the top of the script & execute it you will see the somewhat quaint `zsh` output & how it is breaking apart the version string into separate test commands for each "stanza".

Two things I will say about the script style:

1) The reason I separate my output text in functions from operations is because I hate seeing all that blah blah junking up the logic of the script.

2) The reason the Jamf Script Parameters (if you are using them) get renamed is because every function effectively runs as a subshell. Subshells get their own set of arguments / parameters. If you don't rename the Jamf Script Parameter variables, they would not be "portable" throughout the script, i.e., in the functions.

Most of the ideas in this script come from Tom Larkin's excellent blog post on using Spotlight data which you should definitely read: https://t-lark.github.io/posts/using-spotlight-macos/#caveats-with-custom-tagging

**PS.** Remember to execute this with something like `sudo zsh /path/to/script` or else you will be sad.

Enjoy!

<br/><small>Source: https://community.jamf.com/t5/jamf-pro/a-haiku-on-regular-expressions/td-p/293512</small>
