---
title: SSH Reverse Tunnel
date: 2019-07-23
images: []
tags: ["golang"]
githubUrl: "https://github.com/krishnakumar4a4/go-ssh-reverse-tunnel"
summary: "SSH reverse tunnel for internet"
---
On corporate networks, we always have a problem of having restrictions on internet access to applications.

Of course, these restrictions are absolutely necessary in terms of security. But what if we need to give temporary internet access to these applications?

We can combine SSH reverse tunnel with a proxy server to do this.

# How to run

`./go-ssh-reverse-tunnel -u <username for SSH> -i <private key file path> -t <target ssh host name> -p <reverse tunnel listening port on target>`