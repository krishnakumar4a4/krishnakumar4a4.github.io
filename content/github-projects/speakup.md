---
title: SpeakUp
date: 2018-05-29
images: []
tags: ["erlang", "elixir", "python", "node"]
githubUrl: "https://github.com/krishnakumar4a4/speakup"
summary: "Make discussions less cumbersome"
---
### Meetings and discussions get less cumbersome and more lively
Ever occurred to your mind, why is it so problematic getting your ears on the most important session you had just now. You have some questions, perhaps someone might have asked the same and you wouldn't want to repeat. Oh hell, can someone give him a microphone to talk!! OMG, Microphone is 5 rows behind the audience and it's pretty hard circulating that in time. Isn't there any other way to make these aftermath effective? Yes there is a way. These days everyone carry a mobile phone and mostly smart phone everywhere. It has inbuilt mic and wifi connectivity. Leveraging the most commonly available smart phone resources to create a more effective participant interaction seemed to be a cool idea. Why not? So I started working on it.

### Is it a mobile app?
No it's not, I hate installing apps for every need.It just works with your mobile browser by logging to an url. I deliberately avoided building a mobile app just to eliminate the need for installing an app if at all it could work on a browser.

### How does it work?
You come to an event, connect to the event wifi and then open a link in your browser. You can sign up as participant and you have the ability to speak and your voice instantly comes over the speaker.

### Couldn't this be misused?
Yes there is a possibility. Hence it's introduced a moderator control which a piece of software runs on a machine in the same event network. A moderator can login and view all the available participants and their current status i.e muted or speaking. Moderator has the capability to mute someone, for the benefit of making the interaction smoother or allow everyone to get chance if someone is hogging the questionnaire with extremely long questions. Moderator interaction is not always necessary and an automated handover mechanism is implemented to allow sequential talk.

### It's a simple feature, anything more you are thinking!
Yes I do, live audio transcribing to all the participants, chat feature with all the participants, live feedback to organizers, feedback form, live polling, send questions to speaker. Generate a transcript for the event towards the end. Append audio data to live video streaming so that all the questions can also be heard. Big feature is discussion mode.

### Why within a network?
To prevent outsiders spoiling the event who are not physically present at the venue.

### Can this be a hosted service on internet?
It can be, but will be laggy and I wouldn't prefer that. I would prefer to download the software and host my own.

### What about certificates?(to be done)
We use let's encrypt to generate free certs.

### What is the current software composition?
Elixir and Phoenix for the webapp. Erlang for moderation of audio data to node-speaker. Speech transcription using python .