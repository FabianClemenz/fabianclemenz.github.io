+++
title = "Basic Server Security"
authors = ["Fabian Clemenz"]
description = "This article shows which basics should be applied to every sever installation, before even start installing software on it"
tags = ["server", "security", "ubuntu", "ssh", "firewall", "sudo"]
categories = ["infrastructure", "security", "101"]
series = ["server security 101"]
slug = "basic-server-security"
date = 2025-11-20T08:55:34+01:00
draft = false
+++
#### *or: The best door is useless if the key is left in it*

This article was first published on our company website. [https://devsuit.de/ueber-uns/aktuelles/basic-server-security](https://devsuit.de/ueber-uns/aktuelles/basic-server-security)

If you just want the commands - scroll to the [TL;DR](#tldr) section at the end of the artcile.
Most of the time, I would recommend using Docker and direct container hosting (such as DigitalOcean App Platform), but in some cases you still need a good old-fashioned server. Celery, to name just one example, cannot be used on the DigitalOcean App Platform (as of **November 7, 2025**).

When you reach the point where you need a server, or you're tasked with maintaining an old one, you should know a few basic steps to secure it. You can spend countless hours securing your applications, but if the server itself lacks basic security, it's like leaving the key in your front door.
Here are the basic steps we take on every server.

**Disclaimer**: This guide applies to Ubuntu Server starting from version 24.04. Most commands should also work on older versions and other operating systems.

## Create Users

Access to the server should always be done via dedicated users and never through centralized accounts or, even worse, the root user. There are two simple reasons for this:

1. If a user is compromised, that user's access can be revoked.
2. You can track which user made which changes.

A user is created using `sudo adduser <username> --force-badname`. You will be asked to set a password, which you definitely should do. The `--force-badname` flag is required when the username contains special characters such as dots.
Afterwards, add the user to the **sudo** group so they can execute commands as an administrator: `sudo usermod -aG sudo <username>`.

As a rule, access should only be possible via SSH keys. To do this, store the public SSH key in the user account. Switch to the newly created user with `sudo su <username>` and create the `.ssh` directory in their home folder (`mkdir ~/.ssh`).
Inside this directory, create the `authorized_keys` file, which will contain the public key. The command is: `cd ~/.ssh && nano authorized_keys`. Enter the public key there and save the file.
Finally, set the correct permissions on the `.ssh` folder and the `authorized_keys` file using: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`.

Repeat this process for every user who should have access to the server.

## Sudo command only with a password

To ensure that the **sudo** command can only be used with a password - which adds an extra layer of security - a configuration file must be created. This is done using: `sudo nano /etc/sudoers.d/10-default-user-config`. For each user account, add an entry following this pattern: `<username> ALL=(ALL) ALL`. This ensures that the user is asked for their password the first time they use the sudo command in a session.

**Important:** Only users who have SSH access may be included in this file.

## Secure SSH

After the users have been created, SSH itself is secured. As mentioned earlier, we want to prevent access for the root user while also allowing login only via keys. In addition, it's a good idea to change the SSH port (preferably to a port > 1024). This does not prevent attacks, but it makes them more difficult.

To do this, create a configuration file for SSH using: `cd /etc/ssh/sshd_config.d && sudo nano 10-custom-ssh.conf`.

We use the following configuration:

```bash
Port <SSH Port>
PermitRootLogin no # Disables root login
PasswordAuthentication no # Disables login with password
IgnoreRhosts yes # deactivates old RSH protocol
```

Since Ubuntu **22.04**, SSH uses sockets, so the port must be adjusted there as well. To do this, open the file with `sudo nano /lib/systemd/system/ssh.socket` and replace every instance of **22** with the previously chosen SSH port.

SSH shoudl then be restarted using: `sudo systemctl daemon-reload && sudo systemctl restart ssh`.

After logging out, you can only log in using the user accounts that were created.

## Deactivate the root shell

To prevent a user from switching to the root shell and executing commands there, the root shell is disabled. To do this, open the necessary configuration with `sudo nano /etc/passwd` and change `/bin/bash` to `/usr/sbin/nologin` in the line where `root` appears.

This prevents a user from switching to the root shell using `su root`. Any attempt to switch to the root shell will be logged.

## Firewall

In the final step, a firewall is activated so that only the ports that are actually needed remain open. This way, any access attempts to other ports are immediately blocked by the firewall. We use **ufw** as the firewall.
Using `sudo ufw default deny incoming` and s`udo ufw default allow outgoing`, we block all incoming traffic by default and allow outgoing traffic.

Next, the SSH port needs to be reopened with `sudo ufw allow <SSH Port>`. This must not be forgotten, otherwise SSH access will stop working as soon as the firewall is activated. Additional ports are opened following the same pattern.

Finally, the firewall is activated with `sudo ufw enable`.

## Conclusion

With this short guide, a server is fundamentally secured. Additional configurations and hardening measures must be evaluated and implemented based on specific requirements.

## TL;DR

For anyone who just wants a quick, simple guide.

### Create Users

1. `sudo adduser <username> --force-badname` → adds the account
2. `sudo usermod -aG sudo <username>` → adds the account to the sudo group
3. `sudo su <username> && mkdir ~/.ssh` → switches to the created account and creates an `.ssh` directory in its home directory
4. `cd ~/.ssh && nano authorized_keys` → switches to the `.ssh` directory and opens the `authorized_keys` file. The user's public key is entered in this file.
5. `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` → sets the appropriate permissions

Repeat steps 1–5 for every other user.

### Sudo command only with a password

1. `sudo nano /etc/sudoers.d/10-default-user-config` → creates a new sudoers configuration
2. `<username> ALL=(ALL) ALL` → add this line for each user

Only accounts which are allowed to use ssh may be included in this file!

### Secure SSH

1. `cd /etc/ssh/sshd_config.d && sudo nano 10-custom-ssh.conf` → creates an SSH configuration with the following content:

```bash
Port <SSH Port>
PermitRootLogin no # Disables root login
PasswordAuthentication no # Disables login with password
IgnoreRhosts yes # deactivates old RSH protocol
```

Since Ubuntu **22.04**, SSH uses sockets. To change the port there as well, do the following:

1. `sudo nano /lib/systemd/system/ssh.socket`
2. `Change ListenStream=0.0.0.0:22` to `ListenStream=0.0.0.0:<SSH Port>`
3. `Change ListenStream=[::]:22` to `ListenStream=[::]:<SSH Port>`

Restart SSH using: `sudo systemctl daemon-reload && sudo systemctl restart ssh`

Then log out and log back in with one of the created accounts to verify that everything works.

### Deactivate the root shell

1. `sudo nano /etc/passwd` -> opens the configuration
2. Change `/bin/bash` to `/usr/sbin/nologin` in the line which starts with root

### Firewall

1. `sudo ufw default deny incoming` → block incoming traffic by default
2. `sudo ufw default allow outgoing` → allow outgoing traffic by default
3. `sudo ufw allow <SSH Port>` → allow SSH
4. `sudo ufw allow <port>` → open additional ports (if necessary)
5. `sudo ufw enable` → activates the firewall. **IMPORTANT:** The SSH port must be opened beforehand, otherwise access to the server will no longer be possible.
