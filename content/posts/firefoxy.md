---
title: "Firefoxy"
date: 2022-05-30
tags: ["curl", "firefox", "jamf nation", "json", "jxa", "osascript"]
---

{{< notice info >}}

**NOTE:** These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

Jamf Nation post - https://community.jamf.com/t5/jamf-pro/firefoxy/m-p/266970

{{< /notice >}}

{{< notice note >}}

So,  added `jq` to macOS. I do have a funny story about this... The version I will tell here in public is: I was told that would never happen & then it did! I (& a lot of other people) were very glad about this. Point being, this technique is unnecessary.

{{< /notice >}}

<br/>

-----

<br/>

I am not a huge [Firefox](https://www.firefox.com/en-US/?redirect_source=mozilla-org) user, but, there are many things that are excellent about it & about how it's made. I sometimes get sad it isn't as popular as it used to be especially given the fact that Google is destroying the internet & Google Chrome has no real competition.

Here is a quick example...

Firefox publishes a URL to get information about updates & releases which returns a simple JSON object:

```sh
% curl -LsS 'https://product-details.mozilla.org/1.0/firefox_versions.json'
```
```json
{
  "FIREFOX_AURORA": "",
  "FIREFOX_DEVEDITION": "101.0b9",
  "FIREFOX_ESR": "91.9.1esr",
  "FIREFOX_ESR_NEXT": "",
  "FIREFOX_NIGHTLY": "102.0a1",
  "FIREFOX_PINEBUILD": "",
  "LAST_MERGE_DATE": "2022-05-02",
  "LAST_RELEASE_DATE": "2022-05-03",
  "LAST_SOFTFREEZE_DATE": "2022-04-28",
  "LATEST_FIREFOX_DEVEL_VERSION": "101.0b9",
  "LATEST_FIREFOX_OLDER_VERSION": "3.6.28",
  "LATEST_FIREFOX_RELEASED_DEVEL_VERSION": "101.0b9",
  "LATEST_FIREFOX_VERSION": "100.0.2",
  "NEXT_MERGE_DATE": "2022-05-30",
  "NEXT_RELEASE_DATE": "2022-05-31",
  "NEXT_SOFTFREEZE_DATE": "2022-05-26"
}
```

So easy & practical. Want the latest Firefox version? Here's one way that does not require anything other than commands available in the shell:

```sh
% jsonval(){
  JSON="$1"
  osascript -l 'JavaScript' \
    -e 'const env = $.NSProcessInfo.processInfo.environment.objectForKey("JSON").js' \
    -e "JSON.parse(env).$2"
}
% data="$(curl -LsS https://product-details.mozilla.org/1.0/firefox_versions.json)"
% key='LATEST_FIREFOX_VERSION'
% jsonval "$data" "$key"
100.0.2
```

Much of the credit for this code belongs to these posts:

https://paulgalow.com/how-to-work-with-json-api-data-in-macos-shell-scripts

https://www.macblog.org/parse-json-command-line-mac/

with assistance from @pico in the macadmins Slack #scripting & #bash channels.

My absolute favorite thing in the Firefox update data, however, is `NEXT_RELEASE_DATE` !!!!

```
% key='NEXT_RELEASE_DATE'
% jsonval "$data" "$key"
2022-05-31
```

If only all software publishers were so organized & conscientious...
