---
title: "The date command is pretty hard to use..."
date: 2020-09-02
tags: ["date","jamf nation","macos"]
---

{{< notice info >}}

These posts are being recreated from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----

So, I don't know about you, but, when I use the `date` command it's usually to match some arcane format. This means I am then scrolling to the bottom of the `date` man page to find the name of the man page where all the format options are listed. Every time is just like the 1st time, but, not in a good way! (It's `strftime` by the way... good luck remembering it.)

It gets worse. Let's say you need multiple date formats in a script, for example, because you don't know exactly what date formats the sources you are matching against will use. The obvious thing to do would be to call the `date` command a few times in separate variables:

```bash
date_dd_mon_yyyy="$(/bin/date "+%d %b %Y")"
date_d_mon_yyyy="$(/bin/date "+%e %b %Y" | /usr/bin/sed 's/^[[:space:]]//')"
date_d_month_yyyy="$(/bin/date "+%e %B %Y" | /usr/bin/sed 's/^[[:space:]]//')"
```

Which yield the following (example) date strings:

```
01 Jun 2020
1 Jun 2020
1 June 2020
```

But, there is a problem crystallized in this nugget of wisdom from [Bash Pitfalls](https://mywiki.wooledge.org/BashPitfalls#month.3D.24.28date_.2B-.25m.29.3B_day.3D.24.28date_.2B-.25d.29).

So, what to do, what to do?

I think, perhaps, this:

```bash
eval "$(/bin/date +'century="%C" year_00_Mon="%g" year_00_Sun="%y" year_Mon="%G" year_Sun="%Y" month_01="%m" month_abrv="%b" month_full="%B" week_00_Mon="%W" week_00_Sun="%U" week_01_Mon="%V" weekday_0_Sun="%w" weekday_1_Mon="%u" day_001="%j" day_01="%d" day_1="%e" day_abrv="%a" day_full="%A" hour_00="%H" hour_01="%I" hour_0="%k" hour_1="%l" minute_00="%M" second_00="%S" second_epoch="%s" ampm="%p" timezone_alpha="%Z" timezone_utc="%z"')"
```

Using the Bash Pitfalls idiom for ensuring the `date` command is only called once, my version creates human-readable bash variables for all of the separate components that can make up a date string using the formats available in `strftime`. If I were being fancy I would maybe call it a "library" of possible options that you might want to include in a date string in your script. It's modular! I have intentionally excluded the `strftime` options that create combined strings like `%F`, `%D`, `%X`.

You might be thinking we've traded one opaque solution for another, but, my goal here is to NEVER HAVE TO LOOK AT OR REMEMBER THE NAME OF THE `strftime` MAN PAGE AGAIN. Before judging too quickly, pull up the definitions, take a look at my variable names, then, see if they make sense to you...

Using the `ampm` variable will print out the current "meridiem" for the time.

The `day_01` variable will print a 2-digit version of today's date while `day_1` will print a 1-digit version (one that doesn't use a leading 0.)

The `hour_00` variable prints the 24-hour 2-digit version of the hour, `hour_01` prints the 12-hour 2-digit version, `hour_0` prints a 24-hour 1-digit version, `hour_1` the 12-hour 1-digit version.

The `second_00` variable gives you a 2-digit version of the current time's seconds while `second_epoch` gives you current time in seconds since the epoch (unix time).

And so on... if I wanted to print the ISO 8601 time stamp string in a log, for example, I could do this:

```bash
echo "${year_Sun}-${month_01}-${day_01}T${hour_00}:${minute_00}:${second_00}"
```

which yields:

```
2020-09-02T18:47:25
```

Is the `eval` command super easy to read? Hmm, no. Is it easier to read than the `strftime` man page? Yes! Is it copy / paste-able so you can just put it somewhere in every script you write that calls `date`? Yes! Can you change the variable names I created to make them more "human-readable" to you? Yes! We all win.

As always, I hope some of you find this useful.

<br/><small>Source: https://community.jamf.com/t5/jamf-pro/the-date-command-is-pretty-hard-to-use/m-p/234571</small>

---

## Comments

<small>Bartlomiej Sojka</small>

> "Great and very tidy solution indeed! Although usually 'eval is evil', this one seems pretty much unexploitable. I've always avoided running plain `date` multiple times by using epoch as a source format for later conversion, i.e.: `EPOCH=$(date +%s); date -jf "%s" ${EPOCH} "+%T"` — This however doesn't solve the `strftime` 'pain' problem, so kudos."






