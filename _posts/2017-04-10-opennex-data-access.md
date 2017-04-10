---
layout: post
title: "The new OpenNEX/Planet OS data portal"
date: 2017-04-10
categories: data
tags: bash
---
{::comment}
The filename must be on this format:
2010-01-01-some-sort-of-title.md
the title in the header will be the one displayed on the web page.
{:/comment}
A while back, I was asked by a colleague to help download some scientific data.
What struck me as weird was that the procedure outlined on the web-page involved
deploying a docker image. The data was a subset from a global climate
simulation, which can be generated at the
[OpenNEX web page](http://opennex.planetos.com/).

My co-worker is probably not the only person who got stuck trying to download
the data (several people at the department had problems), so I'll outline the
important steps at the end of this post. But first, I need to raise a few issues
with the OpenNEX access tools.

## A word of warning
Creating a data-subset at the OpenNEX web-page is fairly easy, you select your
model (or models), a time range, the desired variables and the climate scenarios
(Representative Concentration Pathways, i.e. RCP) you wish to study. Then you
select a region, either by outlining it on their interactive map or by uploading
a shape file. Finally, scroll down and click the "create dataset" button. So far
so good, but here's were things get a bit shaky.

In order to download the data, you are asked to deploy a docker container, if
you don't have access to a "\*nix environment", you're out of luck (I do love
that the meaning of "\*nix" is considered common knowledge, by the way). 
At this stage the page asks you to run a given command, here's an example of
what that might look like:

~~~ bash
curl -sS http://opennex.planetos.com/p/eYjr6 | bash
~~~

Now, this is really frustrating for someone who knows just the tiniest bit about IT-security and safe practice. What this line of code does, is to download a script from some webpage (hopefully from OpenNEX) and execute that script immediately in your terminal.  I write *some* web page because, as you might have noticed, OpenNEX does not support SSL, all access to the web page is through http and any attempts to access through https redirects you to the http-site. This means that you cannot be certain that you are communicating directly with OpenNEX, if you don't think that this is important, take a look at
[what Google has to say about
it](https://developers.google.com/web/fundamentals/security/encrypt-in-transit/why-https).

The script you download makes this even worse as it will (by default) require
root access for some commands, so what you're basically asked to do in order to
be able to download the data is:

1. Download a shell script without any integrity checks and execute it.
2. Give the shell script super user privileges, allowing it make any changes to
   your system.

If someone would want to exploit this, it would be very easy, and you wouldn't
even know. Furthermore, even if there would be no security flaws with this
set-up, I really don't understand why the developers decided to use Docker in
this way. Docker is an amazing tool and you can use it in all kinds of clever
solutions, but it is intended as a sysadmin tool, not as something you hand out
to users in order to access your services.

With this said, I should point out that OpenNEX is currently in its early
stages, which is noted in the documentation. Hopefully, future development will
address at least some of these issues.

## How to access the data
These are the steps I used to download the data for my colleague.
(Optional: run this in a virtual machine so that you can simply reset to a
previous snapshot when done)

~~~ bash
# Install docker (assuming you're on Ubuntu):
sudo apt-get install docker.io

# Download the script
curl -sS http://opennex.planetos.com/p/eYjr6 > script.sh

# (Recommended: read through script to check if it does anything fishy)

# Run script
./script.sh
~~~

The script will now attempt to download and launch a docker image on your
system, asking you for your password in order to get root access if needed.
It will print a bunch of line to your terminal, what you're interested in is a
URL which you should  paste into your web browser.
The URL let's you download the data, you must keep the docker image running
until the download has finished!

When your data is downloaded, stop docker:

~~~ bash
sudo docker images  # This will print a code, 
sudo docker stop <code>
sudo service docker stop
~~~

Optionally: Uninstall docker from your system:

~~~ bash
sudo apt-get purge docker*
sudo apt-get autoremove
~~~

...and that's it, just hope that your system is still intact after being so
reckless ;-)
