---
title: "Unify Your macOS Terminal Command History"
date: 2020-05-18
tags: ["macos", "terminal", "zsh"]
---

{{< notice info >}}

These posts are being re-created from old Jamf Nation posts not because they are the best but because they may have useful things that were mangled in the [Jamf Nation](https://www.jamf.com/community/jamf-nation/) transition to its most recent hosting platform.

That, & they were always supposed to be blog posts anyway... I didn't have a blog then.

{{< /notice >}}

<br/>

-----

I'm embarrassed to not have known this until now, but, the joy in the discovery outweighs the pain of embarrassment, so, here goes.

How do you feel about having your Terminal history split into unique sessions in the `.bash_sessions` or `.zsh_sessions` file? Do you hate it? Does it drive you mad with rage that you can't get your entire command history in one place? Just me?

You can fix it.

1) Close any open Terminal windows other than the one you are working in.

2) Run the following command to delete the sessions folder in your home folder:

```zsh
rm -rf .zsh_sessions
```

This is fine. **NOTE:** if your default shell is still bash it will be called `.bash_sessions`

3) Run the following command:

```zsh
touch ~/.zsh_sessions_disable
```

If your default shell is still bash use:

```bash
touch ~/.bash_sessions_disable
```

4) Quit Terminal & open it again.

That's it. Now your Terminal command history is unified into the `.zsh_history` or `.bash_history` file (just like the days of yore...) If you want the sessions behavior back, delete the `sessions_disable` file. So simple.

The end.

Enjoy!

**P.S.** This is a bit recursive (because the link below references THIS POST...) but I wanted to update here with an additional configuration step:

Add

```zsh
export SHELL_SESSIONS_DISABLE=1
```

to your `.zprofile` on macOS Big Sur & later to prevent the sessions folder from recreating itself in every new shell / Terminal window.

See the post at this superuser.com link: [Disable zsh session folder completely](https://superuser.com/questions/1610587/disable-zsh-session-folder-completely) for the full discussion!

<br/><small>Source: https://community.jamf.com/general-discussions-2/unify-your-macos-terminal-command-history-21637</small>
