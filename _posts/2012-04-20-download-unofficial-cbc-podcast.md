---
layout: post
title: "Download CBC Radio stream "
category: Linux
tags: []
---

I listen to CBC Radio a lot. I often find the quality of the show
[Ideas](http://www.cbc.ca/ideas) superb. Recently there was a show
called "All in the Family" which introduced me to the
[ACE](http://www.cdc.gov/ace/index.htm) (Adverse Childhood
Experiences) study. The results of this study are worthy of another
blog post, but this post is about something technical. Sorry.

I wanted to download this show as a podcast so I could share it. I
went the Ideas website and the show is not available as a podcast but
there was a link to listen to the current show. This link brings up a
Flash Audio player which plays the show.

At this point I knew that since the audio is being played on my
computer I can capture it. First I looked in the web source for an
obvious link, but the code is so obfuscated I gave up quickly.

Next I considered recording the audio while it was playing, but I did
not want to tie up the computer for an hour.

After a few Google searches I stumbled across some posts that
mentioned UrlSnooper to figure out the location of a
stream. UrlSnooper is a Windows program, but there was a mention of a
Linux program called [ngrep](http://ngrep.sourceforge.net/). What a
great tool! I ran the following:

    sudo aptitude install ngrep
    sudo ngrep -W byline -qilw 'get' tcp dst port 80

Then I used FireFox to open the audio stream and saw the following in
a long stream of output in the console where I ran ngrep:

> T xxx.xxx.xxx.xxx:35064 -> 64.208.5.41:80
> GET /maven_legacy/thumbnails/ideas_20111213_27203_uploaded.mp3 HTTP/1.1.
> Host: thumbnails.cbc.ca.

An mp3 on thumbnails.cbc.ca?? I tried:

    wget http://thumbnails.cbc.ca//maven_legacy/thumbnails/ideas_20111213_27203_uploaded.mp3

Done! I had the mp3 of the Ideas show. I love Linux.
