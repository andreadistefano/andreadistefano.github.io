---
title: "Make Windows Search Fast Again"
date: 2024-06-12T08:19:15+02:00
draft: false
showToc: true
tags: ["windows", "how-to"]
---

To me, one of the most useful features of Windows is the Search you get by hitting the Windows key on your keyboard and then typing whatever you want. This is the way to make it work for you, instead of earning money for Microsoft.
<!--more-->

## The problem

Letting us open programs or files on our own PC's doesn't earn Microsoft any money, so they gradually built ads and web searches into their search, making it slow and extremely unreliable.

## The solution

* Open `regedit`.
* Head to `Computer\HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Search`.
* Edit (or create) the DWORD value `BingSearchEnabled` and set its value to 0.
* Reboot (maybe a logout would be sufficient but just to be sure).
* Enjoy lightning-fast search on your local files, folders, and applications.