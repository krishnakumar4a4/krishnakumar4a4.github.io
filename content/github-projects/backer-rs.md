---
title: backer-rs
date: 2018-10-19
images: []
tags: ["rustlang"]
githubUrl: "https://github.com/krishnakumar4a4/backer-rs"
summary: "Never Loose data/notes - An efficient git based backup tool to save your changes periodically"
---
Never Loose data/notes - An efficient git based backup tool to save your changes periodically.

A git based backup tool.
Intelligent enough to trigger a commit, only, if there are any file changes in the folder it is pointed to.
Based on cross-platform file notification library.

## Example use cases:
- Can be pointed to storage locations of note applications like boostnote

## Build
- ``cargo build --release``
- ``backer-rs`` is the executable generated in ``target/release/`` folder.

### Usage

``backer-rs -p <path to the folder to backup> -f 2 -c 300 -n krishnakumar -e <email id of author> -d "Commiting all changes"``
- `-p` or `--path` path to monitor for changes, `Note:` This path will be converted to git repo `(Mandatory)`
- `-f` or `--ffreq` time delay(seconds) between monitoring file changes
- `-c` or `--cfreq` Wait time before making an automated commit after first file change
- `-n` or `--sname` Name of the author to be added as signature for commit `(Mandatory)`
- `-e` or `--semail` Email id of the author to be added as signature for commit `(Mandatory)`
- `-d` or `--defcommitmsg` default automated commit message

``backer-rs -p <path to the folder to backup> -n krishnakumar -e <email id of author>``
- Default file monitoring time is `2` seconds
- Default wait time to commit is `5` seconds
- Default commit message is `Committed all changes`
