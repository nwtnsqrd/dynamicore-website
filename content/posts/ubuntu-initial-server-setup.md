---
title: "Initial Ubuntu Server Setup with 2FA"
date: 2022-04-26T11:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["ubuntu", "ansible", "setup", "devops", "automation"]
author: "Pascal"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Learn how securely configure a fresh Ubuntu installation and automate the process with Ansible"
canonicalURL: "https://www.dynamicore.de/posts/ubuntu-initial-server-setup/"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "/img/ubuntu-logo.png" # image path/url
    alt: "Ubuntu Logo" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
---
[![DigitalOcean Referral Badge](https://web-platforms.sfo2.digitaloceanspaces.com/WWW/Badge%202.svg)](https://www.digitalocean.com/?refcode=59d9eefafa3c&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)

For this tutorial, we will be using **DigitalOcean** as our provider, although this tutorial is valid for any scenario where you have root access directly after the installation of your operating system.

Just be mindful to adapt to your individual circumstances where necessary (namely how to first connect to your server and the firewall configuration with your VPS provider).

## Install Ubuntu
This section assumes a noninteractive Ubuntu installation that lets you connect to the server as root after the installation process has finished. Please refer to the 
[official documentation](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview) if your installation requires manual intervention.

We are going to be using the latest available LTS version of Ubuntu, which at the time of writing is 22.04:
![Select the OS version](/img/ubuntu-tutorial-os.png)

If you have already saved SSH keys with your VPS provider, enable to use them with your root login. Otherwise, select password authentication.
![Select authentication method](/img/ubuntu-tutorial-root-auth.png)

## Connect to the new server
After the installation has completed, you can type `ssh root@<ip-addr>` to connect to your new server. If you have multiple hosts you regularly connect to, it is a good idea to create a SSH config
file where you organize your hosts. Also, a config file makes it easier to quickly change your connection parameters without having to remember the syntax of the `ssh` command.

If this file does not already exist, create a config file in your `.ssh/` folder.
```bash
touch ~/.ssh/config
```

Open the file with your favorite editor and add the following:

```bash
Host ubuntu-tutorial
	Hostname <ip-addr>
	User root
```

Don't forget to substitute `<ip-addr>` with the actual IP address of your server.

Now you can connect to your new server by typing `ssh ubuntu-tutorial`. Keep this file in mind, we are going to be making changes to it later on.

After you have accepted the fingerprint, you can access the shell:
```shell
The authenticity of host '<ip-addr>' cant be established.
ECDSA key fingerprint is SHA256:4jhQUjTGEoDczlAtHIlWY14A8ckhX9thRJE+oGzz1dw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '<ip-addr>' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.15.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Apr 26 12:50:27 UTC 2022

  System load:  0.0               Users logged in:       0
  Usage of /:   6.3% of 24.06GB   IPv4 address for eth0: <ip-addr>
  Memory usage: 20%               IPv4 address for eth0: 10.19.0.5
  Swap usage:   0%                IPv4 address for eth1: 10.114.0.2
  Processes:    94

10 updates can be applied immediately.
2 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@ubuntu-tutorial:~# 
```

The first thing you should after you connect to a new server is install pending updates.

```shell
root@ubuntu-tutorial:~# apt update && apt upgrade -y
```

## Create a sudo user

Let's create a user named `pascal`:
```shell
root@ubuntu-tutorial:~# adduser pascal
Adding user `pascal' ...
Adding new group `pascal' (1000) ...
Adding new user `pascal' (1000) with group `pascal' ...
Creating home directory `/home/pascal' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for pascal
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n]
```

This user has no special privileges yet. Add `pascal` to the `sudo` group and validate afterwards:
```shell
root@ubuntu-tutorial:~# usermod -aG sudo pascal
root@ubuntu-tutorial:~# groups pascal
pascal : pascal sudo
```
### SSH Key

Next up, `pascal` needs a public SSH key so we can authenticate. If your root user already has an SSH key you provided during installation, you can simply copy the `.ssh/` folder from root to the new user.

```shell
root@ubuntu-tutorial:~# rsync --archive ~/.ssh /home/pascal/
root@ubuntu-tutorial:~# chown -R pascal: /home/pascal/.ssh/
```

## Configure sshd

Open the file `/etc/ssh/sshd_config` with your favorite editor. We are going to make a couple of crucial changes.

First, find the line that says
```bash
#Port 22
```
Uncomment it and change the port to something else (be mindful of other ports being used by different services). 

Although it is very tempting to use port 69, we are going to stick to the general recommendation not to use custom ports below 1024, as they are reserved for the system.
```bash
Port 1069
```
Next, find the line that says
```bash
PermitRootLogin yes
```
and simply change it to
```bash
PermitRootLogin no
```
This next change is not necessary on every system. Sometimes your VPS provider already changes this for you during the installation if you have chosen to use SSH keys instead of a password
for the root login.

Check for the line that says
```bash
#PasswordAuthentication yes
```
and change it to
```bash
PasswordAuthentication no
```
Lastly, restart the ssh daemon for the changes to take effect
```shell
root@ubuntu-tutorial:~# systemctl restart sshd
```
## Enable 2FA
Using SSH to authenticate with a remote host is already very secure, compared to password authentication. For some machines though, adding an extra layer of security lets you sleep better at night.
I would recommend this for business critical data, such as backups.

We are going to be using `libpam-google-authenticator`, which is [open source](https://github.com/google/google-authenticator-libpam). To retrieve our OTP (one-time password) we will be using the Android app
`Authy` on our phone.

Let's install the package on our server with

```shell
root@ubuntu-tutorial:~# apt install libpam-google-authenticator
```

Next, we need to configure the PAM sshd settings in `/etc/pam.d/sshd`. Look for the line that says

```bash
@include common-auth
```

and comment it out like so:

```bash
# @include common-auth
```

At the end of the file, add this line:

```bash
auth required pam_google_authenticator.so
```

Save and close. Now open the sshd configuration again, `/etc/sshd/sshd_config`, and change

```bash
KbdInteractiveAuthentication no
```

to `yes`.


The next setting is probably already enabled by default, but make sure the line

```bash
UsePAM yes
```
exists, is not commented out or says `no`. Below the `UsePAM` line, add the following:

```bash
AuthenticationMethods publickey,keyboard-interactive
```

Save the file and exit. Now, let's restart the ssh daemon again:

```shell
root@ubuntu-tutorial:~# systemctl restart sshd
```

Last, but definitely not least, we need to launch the `google-authenticator` initialization script so we can retrieve our OTP. Switch to the user you want to authenticate with and run the script like so:

```shell
root@ubuntu-tutorial:~# su - pascal
pascal@ubuntu-tutorial:~$ google-authenticator 

Do you want authentication tokens to be time-based (y/n) y
Warning: pasting the following URL into your browser exposes the OTP secret to Google:
  https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/pascal@ubuntu-tutorial%3Fsecret%3DTYWBISKKQKN3EIH2DNN6QVUTAY%26issuer%3Dubuntu-tutorial
Your new secret key is: TYWBISKKQKN3EIH2DNN6QVUTAY
Enter code from app (-1 to skip): 050826
Code confirmed
Your emergency scratch codes are:
  28440763
  58054876
  87748414
  87026862
  64793459

Do you want me to update your "/home/pascal/.google_authenticator" file? (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y

```

After answering the first question, you get a QR code that you can scan with `Authy` . Do that, then proceed with the initialization. Read the follow-up questions to see if you agree with how I configured it in this tutorial or if you want different settings.

**(!) Leave your current terminal session to the remote host open!** If you close your current session and there is an error in your configuration, you are locked out of your system.

## Test SSH connection

We are ready to test! On your local system, open `~/.ssh/config` and make changes according to your configuration. For the configuration in this tutorial, it looks like that:

```bash
Host ubuntu-tutorial
        Hostname <ip-addr>
        User pascal
        Port 1069
```

Open a new terminal and connect to the remote host like so:

```shell
➜  ~ ssh ubuntu-tutorial
Verification code: 
Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.15.0-27-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Apr 27 07:48:10 UTC 2022

  System load:  0.0732421875       Users logged in:       1
  Usage of /:   10.2% of 24.76GB   IPv4 address for eth0: <ip-addr>
  Memory usage: 23%                IPv4 address for eth0: 10.19.0.5
  Swap usage:   0%                 IPv4 address for eth1: 10.114.0.2
  Processes:    99

0 updates can be applied immediately.


Last login: Wed Apr 27 07:15:17 2022 from 178.13.253.30
pascal@ubuntu-tutorial:~$ 
```

When you are prompted for the verification code, enter the current OTP from `Authy`.

## Disable password prompt for sudo (optional)

It's annoying. You will definitely not gain security by disabling password prompts, so do this on your own volition.

Open the sudoers file with the visudo utility:

```shell
pascal@ubuntu-tutorial:~$ sudo visudo
[sudo] password for pascal:
```
Towards the end of the file, change the line
```bash
%sudo   ALL=(ALL:ALL) ALL
```

to say

```bash
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
```

Save and close the file. The next time you log in with any sudo user you will not be prompted for your password when you execute a command with `sudo`.
## Configure tzdata

Your server time is important, especially if you communicate with other systems. Check the current time on your system like so:

```shell
root@ubuntu-tutorial:~# date
Wed Apr 27 08:03:35 UTC 2022
```
In my case, this is not the correct local time in my timezone `Europe/Berlin`. Set your timezone like this:

```shell
root@ubuntu-tutorial:~# dpkg-reconfigure tzdata

Current default time zone: 'Europe/Berlin'
Local time is now:      Wed Apr 27 10:06:11 CEST 2022.
Universal Time is now:  Wed Apr 27 08:06:11 UTC 2022.
```

After typing `dpkg-reconfigure tzdata` you will enter an interactive shell program that lets you choose your timezone.

## Automate the process
```yaml
become: true
```

## Configure Firewall
### ufw
### vps provider
