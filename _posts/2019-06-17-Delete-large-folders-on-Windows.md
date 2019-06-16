---
layout: post
title:  "Delete large folders on Windows"
date:   2019-06-15
categories: life-saver
tags: windows dos
---

* content
{:toc}

## Motivation

At times we may want to delete huge folders, e.g., when doing backups. Deleting large folders in Windows explorer could take quite a while because Windows would first calculate the folder's size, which is time-consuming. Therefore, it would be great to know how to quickly delete huge folders on Windows.

## Method

Suppose the folder to be deleted is called `foldername`.

1. Open cmd.
2. Type `cd C:\backup\foldername` to navigate to the folder to be deleted.
3. Type `del /f/q/s *.* > nul` to delete all files in `foldername`, while leaving the folder structure intact (for now).
4. Type `cd ..` to navigate to the parent folder of `foldername`.
5. Type `rmdir /q/s foldername` to delete `foldername` along with its subfolders.