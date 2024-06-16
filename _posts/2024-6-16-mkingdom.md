---
layout: post
title: mKingdom TryHackMe writeup
---

mKingdom is advertised as an easy, beginner-friendly room, but I have some reservations about this claim. While standard methods will get you started, there are several rabbit holes that can mislead you. Although initial progress might seem straightforward, obtaining the user and especially the root flag can be quite challenging for beginners. I suspect this box may be reclassified as a medium difficulty in the future.

___

![alt text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom.png "MARO!")

_Note: I always use my own VM when pawning a box_

___

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

___

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

___

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)

We run Gobuster to find any directories. I like to start with a small wordlist first, and if it comes up empty, I do an extension search:

> gobuster dir -u http://mkingdom.thm:85 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt

We find one directory immediately. If you let the full scan run, it doesn't show anything else. 
 
> /app                  (Status: 301) [Size: 312] [--> http://mkingdom.thm:85/app/]

When we open this web folder, we only see a green button with "JUMP" on it. Clicking it will redirect us to the main page. (Don't forget to look at the source code as well! But still nothing interesting.)

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/toads_website.png)

___

<details>
   <summary>There are a few things we can check now. Can you think about them all? Click here if you want a hint</summary>
   In total, there are three things I checked:
   
   1. What you should already have checked twice in this room if you followed my instructions.
   2. That search bar is intressting.
   3. Checking every link on the page (dirbusting only led you down a rabbit hole).
</details>

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)


### 1. Source code!
   
When we look in the source code now, we can see the Content Management System (CMS) that was used, along with the version. This information will come in handy later.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/concrete.png)

___
### 2. Search bar might be vulnerable

There is a search bar at the top of the webpage. When we search for something, note how the URL changes to parameters with a search path or query. This might suggest the possibility of a directory traversal attack to gather information about the system running the webserver. Try it out, but don't spend too much time on it, as this version of Concrete CMS is not vulnerable to any parameter tampering attack.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/parameters.png)

___

<details>
 <summary>Click here if you want to see what I tried to tamper with these parameters</summary>
   I tried for about 30 minutes to see if I could trick the system into giving me information that it wasn’t supposed to. Maybe Toad saved his password somewhere in a hidden post, so I tried search queries like 'password', 'admin', and 'secret'. Everything came back empty.
   

 Secondly I tried to just search for /etc/passwd, but the results were empty. Then, I tried injecting it directly into the URL bar, but still without any result. I tried adding multiple ../ in front of it and ran a custom script to inject multiple payloads automatically. All results were empty.
 

I also tried to inject the search_path parameter. I was not sure what the [] symbols did in the parameter and could not find much information about it on the internet. I changed it to _search_path[/]=&query=/var/www/html/index.html_ to see if I could set the root of the search path to the root of the system. It did not work and was probably also not going to work like this if the search bar was vulnerable.


Then, I tried some common SQL injection (SQLi) commands, in case the search function was handled by a SQL database. No payloads worked. I captured the POST packet, saved it to a file, and ran it through sqlmap:


 > sqlmap -r request.txt


sqlmap gave back no parameters where injectable, so after all this I came to the comclusion the search bar could not be exploited.

</details>

___

### 3. Check out all paged
Manual enumeration is often overlooked but remains very powerful. I only glanced at the blog and contact pages, as well as their source code, and quickly concluded that they are not interesting. However, there is a Log in link at the bottom of the page:

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/log_in.png)

___

Clicking this link leads to a login form. Your hacker senses should be tingling right now. Getting past this login page seems crucial. When we search Google for 'concrete5 8.5.2 exploit', we find a very useful [manual for a Remote Code Execution exploit to get a reverse shell](https://vulners.com/hackerone/H1:768322)

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/login_form.png)

___

There is only one problem, though. The guide states we need to be an admin user to upload our PHP reverse shell, so we need to find a way in.

This part stumped me for quite some time! I eventually went dirbusting and found a link to browse all the Concrete CMS files on the server. There are a lot of directories to sift through! I was hoping to find a members folder to discover the admin username, which I could use to brute force passwords, but I couldn't find anything. Don't fall into this rabbit hole!

Now, it is time for you to think about how to get to the dashboard. Don't think too hard...

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)

___

I tried some default passwords with the username 'toad', because he is probably the admin of this webserver, right? When this didn't work immediately, I tried using Hydra to brute force it. However, I noticed that a 'ccm_token' was added to the POST request during login attempts. This token was different for different usernames, so I tried encoding my username and password to see if I could generate the same token. I failed to do this, and after some online searching, it seems this token is generated randomly on the server.

I decided to check if Hydra could determine when a failed login occurred, but it could not. The server even banned my IP after too many failed attempts, and I had to restart the machine. Hydra was not the way to go here.

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/ccm_token.png)

___

But then I remembered that this box is labeled as easy, and people in the Discord were getting past it with very little effort. I had to think less hard and fall back on the most basic form of login. I tried the most default of default credentials.

> admin admin.... WRONG!
> admin password.... CORRECT!

I hit my wall out of frustration and went on to upload my PHP shell.

# Getting a shell

Because the webserver is the only service running on the machine, the PHP shell will be my only way to control the machine. I generated a [PHP Pentestmonkey reverse shell](https://www.revshells.com/), saved it and looked again at [the guide from earlier](https://vulners.com/hackerone/H1:768322). It's not that hard to do. Just follow the guide, and you will have your shell up in no time (don't forget to whitelist the php extention under System & Setting > Files > Allowed File Types). When uploading the file, click on close when the bar under the uploaded shell turns green. Start a netcat listener and click the link after you uploaded your shell.

> ncat -lvnp 8888 (or any other Lport you want to use. Make sure it is the same as in your php shell script!)

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/mkingdom/shell_link.png)

___

Now that we have access, we need to stabilize the shell. In your Netcat listener, copy and paste the following lines:

> python3 -c 'import pty;pty.spawn("/bin/bash")'
>
> export TERM=xterm

Next, hit 'Ctrl + Z' to put the Netcat listener in the background (don't worry, it is still running, but you need to paste the following command in your own terminal on the same window that Netcat is running):

> stty raw -echo; fg

Hit 'Enter' twice and now we have a shell that mimics all the nice functionalities of a normal terminal window. Time to enumerate the system and see if we can escalate our privileges to a user. Good luck!

![alt_text](https://raw.githubusercontent.com/SpongySystems/spongysystems.github.io/master/images/think.png)

___

As a www-data user, we have very limited priviledges. I ran linpeas.sh to enumerate automaticly. First I uploaded the linpeas.sh script to the server the same way as we uploaded the shell. I named it linpeas.txt to bypass the filter this time. Then I ran the script, copied the output to a txt file and downloaded it on my own system. Instead of clicking the upload link, I copied it and moved to that folder in the terminal

> 

