.. title: crontab and date
.. slug: crontab-and-date
.. date: 2016-01-18 22:42:39 UTC+01:00
.. tags: linux,cron,bash
.. category: linux
.. link: 
.. description: 
.. type: text

The other day, I wanted to add a script to the crontab and to redirect the
output to a file including the current date. Easy. I have used the
`date` command many times in bash script like that::

  current_date=$(date +"%Y%m%dT%H%M")

So I added the following to my crontab::

  0 1 * * * /usr/local/bin/foo > /tmp/foo.$(date +%Y%m%dT%H%M).log 2>&1


And... it didn't work...

I quickly identified that the script was working properly when run from the
crontab (it's easy to get a script working from the prompt, not running
from the crontab due to incorrect PATH). The problem was the redirection
but I couldn't see why.

I googled a bit but didn't find anything...

I finally looked at the man pages::

  $  man 5 crontab

       ...
       The  ``sixth''  field  (the  rest of the line) specifies the command to be run.  The entire command portion of the line, up to a
       newline or % character...


Here it was of course! `%` is a special character. It needs to be escaped::

  0 1 * * * /usr/local/bin/foo > /tmp/foo.$(date +\%Y\%m\%dT\%H\%M).log 2>&1

Lesson to remember: check the man pages before to google!
