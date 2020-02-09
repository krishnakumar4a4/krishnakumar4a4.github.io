+++
authors = [
    "Krishna Kumar Thokala",
]
title = "“backer-rs” the missing versioned backup utility for boostnote/any note-taking app"
description = "“backer-rs” the missing versioned backup utility for boostnote/any note-taking app"
date = 2018-10-21T09:43:12+05:30
tags = [
    "rustlang",
    "announcements"
]
categories = [
    "rustlang",
]
series = ["rustlang"]
aliases = ["rustlang"]
images = []
+++
“Autosave notes”, it is a pretty important feature for any note taking app. Having autosave enabled and running in the background saves you from loosing any important notes you have taken. We would have felt autosave itself is a kind of backup feature for your notes.

Recently, I had been in a situation where the autosave feature of boostnote misbehaved and lost all my notes till date. It’s a big disappointment.

> I have finished my work, wrote a beautiful document on the work I have done on boostnote and left for the day. Next morning, when I logged back in, I noticed I am seeing an older version of my notes. After a while, some more content missing from the same notes. I restarted the boostnote app with a hope to restore everything back, but unfortunately ended up being a clean slate. I have literally gone back in time and all my notes are actually gone.

I never thought that I would loose notes just like that. I did some googling and realized that boostnote stores all the notes in simple text files. The moment I saw that it is not backed by any sort of version control. I lost hope that my notes can be recovered at all by any means.

> I should honestly agree, the user interface and markdown editor of boostnote is simply wonderful though. So, I could not give up on boostnote.

In spite of repenting(for not taking care of this before), I started writing a tool with the features that I felt would have been there in the first place if at all a tool exist.

It is a simple, auto commit application backed by git version control. It monitors for changes on a folder and makes a quick commit for you. It doesn’t commit, if there are no changes at all, thus preventing unnecessary commits in the history.

Initially I called it “boostnote-backup” and later felt that it can be a generic backup tool for any other note taking apps like orgzly etc.. and hence named it “backer-rs”.

It is actually opensourced out here. I would definitely love some pull requests, suggestions and feedback.
Currently, it is very basic version out there. We can add features like
- Making it run as a system service.
- Pushing to a remote repository after certain preconfigured time.
- Reading configuration from a file instead of just command line args.
- CI pipeline to setup Cross compiled binaries for windows, Mac, Linux and Android.
- Making it an installable package for the above platforms.
- Etc..

Long live note taker’s!!

> My notes are my prized possession. It has my wonderful thoughts for the future and hard to remember facts of the past. — I made this up.
