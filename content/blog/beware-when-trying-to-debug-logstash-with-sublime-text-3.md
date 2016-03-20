+++
date = "2014-11-12T10:02:00+01:00"
title = "Beware when trying to debug logstash with Sublime Text 3"

+++

The last days I've spent working on extending the logging mechanism of one of our applications to additionally log in JSON format which then can be used with logstash directly without the need of filtering the data.

The log file generation took place in a vagrant box and was synced to the host system with a synced folder where logstash was watching for changes. The problem was that, although logstash found changes, nothing was parsed. I thought there are likely issues with vagrant synced folders so I started to edit the log file directly with Sublime Text 3 on my host machine. The same behaviour occured.

I've spent hours trying to figure out what was wrong. Sometimes I've had the wrong encoding, sometimes it worked when restarting logstash, other times not.

My last idea was to append a new JSON line via command line on OS X. It suddenly worked perfectly. After that I inspected the log in Sublime again. There was a new line at the end of the file. I thought that has to be the problem. So I tried inserting another JSON line with a following new line. You may guess: It stopped working.

After some googling about how Sublime saves files I found a thread on [Stackoverflow](http://stackoverflow.com/questions/20634684/what-is-sublime-text-doing-when-i-save-a-file) which just explains theses strange behaviour perfectly.

It looks like Sublime Text 3 uses "atomic save" by default which creates a temp file on save and then overwrites the original file resulting in other tools not recognising that the file has changed or even loosing reference to that file. I quickly jumped to the settings and disabled "atomic save" (How To? -> explained in the above thread)

Now it works like a charm. Nevertheless a new line at the end of the file is required to get logstash to recognise the change. I hope you won't fall into the same trap.
