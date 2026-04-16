---
layout: post
title: "One-Liners You'll Forget Without This Post"
category: serious-stuff
topic: CLI
---

## TL;DR

A running list of short, useful CLI one-liners for archives, logs, git, and daily ops — the ones you keep Googling because muscle memory refuses to stick.

## Why You Should Care

You've typed these before. You'll type them again. And next time, you'll waste 4 minutes on Stack Overflow looking for the exact flag you forgot. This post is the bookmark that saves those 4 minutes, every time.

## Archives — Because Passwords and Recursion Are Always Tricky

**Extract a password-protected 7z archive:**

```bash
7z x archive.7z -pYourPassword
```

No space between `-p` and the password. Forget that and you'll stare at a "wrong password" error for longer than you'd like to admit.

**Create a password-protected zip with recursion:**

```bash
zip -er archive.zip /path/to/folder
```

`-e` encrypts, `-r` recurses into directories. Without `-r`, you get an empty zip and a brief identity crisis.

**Extract a tar.gz (because nobody remembers tar flags):**

```bash
tar -xzf archive.tar.gz -C /destination
```

`x` extract, `z` gzip, `f` file, `-C` target directory. Think of `-xzf` as "eXtract Ze File."

## Git — Because You'll Absolutely Forget These

**Merge with history (not squash):**

```bash
git merge --no-ff branch-name
```

`--no-ff` creates a merge commit even if fast-forward is possible. You'll forget the flag, fast-forward anyway, then wonder where your branch history went. Add it now, thank yourself later.

**Full sequence: merge dev into main:**

```bash
git checkout main
git merge dev
git commit -m "Merge dev into main"
git push
```

Straightforward, but the commit step sometimes trips people up — it only runs if the merge needs a commit (i.e., dev has diverged from main). If dev is already merged, you'll see "Already up to date."

**Stash uncommitted changes before a merge:**

```bash
git stash push -m "Stash before merge for skipped validations update"
```

Named stashes actually get used later. Unnamed ones accumulate like browser tabs. The `-m` flag is your friend.

**Unstash after the merge, but revert specific files:**

```bash
git reset HEAD file1
git checkout -- file1
```

First command unstages `file1` from the merge (removes it from the index). Second command reverts it to its pre-merge state. Useful when the merge brought in changes you don't want.

## Windows Event Logs — Filtering the Noise

**Get Code Integrity events (WDAC enforcement logs):**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3076,3077}
```

`3076` = audit block (policy would have blocked). `3077` = enforced block (policy actually blocked). These are your two best friends when troubleshooting WDAC.

**Get Security log events by ID (e.g., logon events):**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624,4625}
```

`4624` = successful logon. `4625` = failed logon. Swap in any Event ID you need — the pattern is always the same.

**Get events from the last 24 hours:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddDays(-1)}
```

Adding `StartTime` prevents PowerShell from choking on a massive log. Your terminal will thank you.

**Quick export to CSV for further analysis:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} | 
    Select-Object TimeCreated, Id, Message | 
    Export-Csv -Path .\events.csv -NoTypeInformation
```

Pipe, select what matters, dump to CSV. Clean and portable.

## Networking — Quick Checks

**Test if a port is open (PowerShell):**

```powershell
Test-NetConnection -ComputerName server01 -Port 443
```

Faster than installing telnet. Again.

**Curl with timing (how slow is that endpoint really?):**

```bash
curl -o /dev/null -s -w "Connect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" https://example.com
```

Three numbers that tell you where the slowness lives.

## Takeaway

Bookmark this. Or don't — you'll just Google the same commands again tomorrow. But at least now they're all in one place, waiting for you.

This is a living post. It'll grow as more one-liners earn their spot.
