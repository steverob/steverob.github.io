---
layout: post
title: "RackRewindableInput lock file not found"
date: 2015-03-11 17:05:41 +0530
comments: true
categories: ops
---

We were doing some crazy things with our staging server and it ran out of space which we did not notice because of the lack of monitoring on this instance. When our Q/A tried accessing the server later on the app was inaccessible and was throwing 500s all over.

The exception tracker we are using sent us the stacktrace and the error message was incredibly puzzling - `Errno::ENOENT: No such file or directory /tmp/RackRewindableInput20150311-13259-cvqkcr.lock)`. I was not sure what to make of this and I had no idea that we had ran out of space.
<!--more-->
Using the stacktrace I figured out that the exception was thrown from a class - `RewindableInput` in the Passenger project. There was note saying this was a modified version of `Rack::RewindableInput`. So this is actually a part of the Rack codebase as well.

After doing some research I learned that this RewindableInput class is used to make a the request body `rewindable` which apparently means that you can bring it back to the original state whenever you want to (if I understand correctly).

The exception was thrown at [this line](https://github.com/phusion/passenger/blob/stable-3.0/lib/phusion_passenger/utils/rewindable_input.rb#L86) where a `tempfile` is being created and since we did not have any space on the disk this could not happen and thus we got the error.

Even then, since the error message was `Errno::ENOENT: No such file or directory` and me being not aware of how [Tempfile](https://www.omniref.com/ruby/2.2.1/symbols/Tempfile) works could not figure out that the reason was the diskspace. Then I ran into a StackOverflow answer where the OP had reported a similar problem with JRuby and the reccomended solution was the check if the `/tmp` directory is writable and has proper permissions. But the OP later reported that he had actually ran out of space.

Why did I not think of that? Anyway, I then ran `df` and found out the disk had no space left. I immediately cleared the junk that was created because of the **crazy stuff** I had been doing earlier and everything went back to normal.

So! If you ever come across this exception - `Errno::ENOENT: No such file or directory /tmp/RackRewindableInput<timestamp>-<random>.lock)` check your disk space first! :)