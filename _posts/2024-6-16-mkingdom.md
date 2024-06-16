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
  If you said to do a directory scan, you are not wrong. However, it is always a good idea to look at the source code first, especially on a defaced website. You might find your powerstar just there.

  For this room, there isn't much in the source code. We see that the loaded image is located in the root of the webserver, which doesn't help much. So, we need to perform a directory busting (dirbusting) scan.

  ![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/sourcecode1.png)
 
</details>

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)

We run Gobuster to find any directories. I like to start with a small wordlist first, and if it comes up empty, I do an extension search:

> gobuster dir -u http://mkingdom.thm:85 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

We find one directory immediately. If you let the full scan run, it doesn't show anything else. 
 
> /app                  (Status: 301) [Size: 312] [--> http://mkingdom.thm:85/app/]

When we open this web folder, we only see a green button with "JUMP" on it. Clicking it will redirect us to the main page. (Don't forget to look at the source code as well! But still nothing interesting.)

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/toads_website.png)

<details>
   <summary>There are a few things we can check now. Can you think about them all? Click here if you want a hint</summary>
   In total, there are three things I checked:
   
   1. What you should already have checked twice in this room if you followed my instructions.
   2. That search bar is intressting.
   3. Checking every link on the page (dirbusting only led you down a rabbit hole).
</details>

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)

1. Source code!
   
When we look in the source code now, we can see the Content Management System (CMS) that was used, along with the version. This information will come in handy later.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/concrete.png)

2. Search bar might be vulnerable

There is a search bar at the top of the webpage. When we search for something, note how the URL changes to parameters with a search path or query. This might suggest the possibility of a directory traversal attack to gather information about the system running the webserver. Try it out, but don't spend too much time on it, as this version of Concrete CMS is not vulnerable to any parameter tampering attack.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/parameters.png)

<details>
 <summary>Click here if you want to see what I tried to tamper with these parameters</summary>
   I tried for about 30 minutes to see if I could trick the system into giving me information that it wasn’t supposed to.
First, I tried to just search for /etc/passwd, but the results were empty. Then, I tried injecting it directly into the URL bar, but still without any result. I tried adding multiple ../ in front of it and ran a custom script to inject multiple payloads automatically. All results were empty.

I also tried to inject the search_path parameter. I was not sure what the [] symbols did in the parameter and could not find much information about it on the internet. I changed it to _search_path[/]=&query=/var/www/html/index.html_ to see if I could set the root of the search path to the root of the system. It did not work and was probably also not going to work like this if the search bar was vulnerable.

Then, I tried some common SQL injection (SQLi) commands, in case the search function was handled by a SQL database. No payloads worked. I captured the POST packet, saved it to a file, and ran it through sqlmap:

 > sqlmap -r request.txt

sqlmap gave back no parameters where injectable, so after all this I came to the comclusion the search bar could not be exploited.
</details>

