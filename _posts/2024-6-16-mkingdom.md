---
layout: post
title: mKingdom TryHackMe writeup
---

mKingdom is advertised as an easy, beginner-friendly room, but I have some reservations about this claim. While standard methods will get you started, there are several rabbit holes that can mislead you. Although initial progress might seem straightforward, obtaining the user and especially the root flag can be quite challenging for beginners. I suspect this box may be reclassified as a medium difficulty in the future.

![alt text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom.png "MARO!")

_Note: I always use my own VM when pawning a box_

# Enumeration
This first step should be very obvious when you start. Start the machine and Nmap that boy! My target IP is 10.10.226.51, so change this IP to your target IP.

I always start by scanning all ports to avoid missing any services running on uncommon ports:
 
> nmap -sS -p 1-65535 10.10.226.51

![alt text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/nmap1.png)

We see only an HTTP server running on port 85, which is not the standard port 80. Thank you, Mario, for proving my point about not just checking standard ports! When I did this box yesterday, this port was open and not filtered. This means we cannot detect the specific service running this webserver, but that doesn’t matter much for this room.

If your machine returns an open port, you can still gather more information about this webserver with the following command:

> nmap -sC -sV -Pn 10.10.226.51 -p 85

**It will reveal that the server is running on *Apache httpd 2.4.7 ((Ubuntu))* and we get a suspicious http-title....

Let's-a-go to the website now!**

# Webserver Enumeration
I always add the server IP to my '_/etc/hosts_' file and I recomment you doing the same:

> sudo nano /etc/hosts

And add this line to your hosts (change the IP to your target machine address!)
> 10.10.226.51    mkingdom.thm

When navigating in your web browser to '_http://mkingdom.thm:85/_'. you should see the following page. It looks like the main page was defaced, so we need to find another way into the main website.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/defaced.png)

<details>
  <summary>We have reached our first decision point. What’s your first instinct about what to do next? Click here to see if you were right!</summary>
  If you said to do a directory scan, you are not wrong. However, it is always a good idea to look at the source code first, especially on a defaced website.

 
</details>

