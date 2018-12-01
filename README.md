# www.lakesite.net #

JAMStack for lakesite.net using hugo.

## Setup

Ensure [Hugo](https://gohugo.io/) or similar static site publishing tool is installed on the remote end,
make sure you've set a git hook on the master branch, e.g.:

    $ mkdir ~/github && cd ~/github
    $ git clone https://github.com/lakesite/www.git
    $ git submodule update --init
    $ vim ~/github/www/.git/post-merge

Add the logic to run hugo e.g.;

    ```#!/bin/bash
 
       cd /home/lakesite/github/www && git submodule update --recursive
       /usr/bin/hugo -s /home/lakesite/github/www/ -d /var/www/vhosts/lakesite.net/www/
     '''

    $ chmod +x ~/github/www/.git/hooks/post-merge

Then create a cron job to periodically check e.g.

    $ crontab -e

* * * * * cd ~/github/www && git pull > /dev/null 2>&1

Which will issue git pull every minute, and run post-merge if there are any commits.

