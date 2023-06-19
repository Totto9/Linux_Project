# Linux_Project
linux server
## Project context 
The local library in your little town has no funding for Windows licenses so the director is considering Linux. Some users are sceptical and ask for a demo. The local IT company where you work is taking up the project and you are in charge of setting up a server and a workstation.
To demonstrate this setup, you will use virtual machines and an internal virtual network (your DHCP must not interfere with the LAN).


You may propose any additional functionality you consider interesting.

## Must Have

Set up the following Linux infrastructure:

1. One server (no GUI) running the following services:
    - DHCP (one scope serving the local internal network)  isc-dhcp-server
    - DNS (resolve internal resources, a redirector is used for external resources) bind
    - HTTP+ mariadb (internal website running GLPI)
    - **Required**
        1. Weekly backup the configuration files for each service into one single compressed archive
        2. The server is remotely manageable (SSH)
    - **Optional**
        1. Backups are placed on a partition located on  separate disk, this partition must be mounted for the backup, then unmounted

2. One workstation running a desktop environment and the following apps:
    - LibreOffice
    - Gimp
    - Mullvad browser
    - **Required** 
        1. This workstation uses automatic addressing
        2. The /home folder is located on a separate partition, same disk 
    - **Optional**
        1. Propose and implement a solution to remotely help a user

## 1. Add new user

_Note: run `ssh root@server_ip` in a terminal if you are not connected. Replace server_ip with your VPS IP address._

To avoid using the root account on a regular basis, we need to first add a new user and grant it administrative privileges:

`adduser newuser`

Enter a strong password and hit Enter to skip optional fields.

`usermod -aG sudo newuser`

## 2. Install firewall

Update and install a basic firewall via apt:

`apt update`

`apt install ufw`

Set default policies:

`ufw default deny incoming`

`ufw default allow outgoing`

Allow ssh connections and enable the firewall:

`ufw allow SSH`

Verify the status with:

`ufw status verbose`

Only proceed if the output includes this line:

```
To             Action      From
--             ------      ----
22/tcp (SSH)   ALLOW IN    Anywhere
```

Enable the firewall:

`ufw enable`

_Note: later you will probably need to adjust the firewall settings to allow traffic in from other services that you install; example for Nginx: `sudo ufw allow 'Nginx Full'`._

You can now close the connection to the server with `exit`.

## 3. Harden SSH

### 3.1 Client config

On your local machine, open the `ssh_config` file:

`sudo nano /etc/ssh/ssh_config`

Add or modify these lines to allow public key authentication, disable IPv6 to reduce attack surface, and use the strongest cipher suites, message integrity codes, and key exchange algorithms available:

```
PubkeyAuthentication yes
AddressFamily inet
StrictHostKeyChecking ask
Ciphers chacha20-poly1305@openssh.com
MACs hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256
```

Save the file with `CTRL+X` > `Y` > press Enter.

### 3.2 Server config

Open two terminal tabs/windows and establish two separate SSH connections to the server, for backup purposes:

`ssh root@server_ip`

Restart the sshd daemon:

`systemctl restart sshd`

If both connections are still alive, try opening a third terminal window and test new connections:

`ssh root@server_ip`

Proceed if everything is working as expected.

In any terminal, make a backup of the original SSH configuration file with:

`cp /etc/ssh/sshd_config /etc/ssh/sshd_config_backup`

Now open the `sshd_config` file:

`nano /etc/ssh/sshd_config`

Add or modify these lines to enable SSH v2 for added protections against known vulnerabilities, disable IPv6, disallow root/pass logins, only allow logins from the user `newuser`, and use the strongest ciphers and algos available:

```
Protocol 2
AddressFamily inet
PermitRootLogin prohibit-password
PasswordAuthentication no
AllowUsers newuser
Ciphers chacha20-poly1305@openssh.com
MACs hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256
```

_Note: replace `newuser` with the actual username from step 1._

_Note: run `ssh -Q cipher`, `ssh -Q mac`, and `ssh -Q kex` for a list of supported ciphers and algorithms._

You can also change the default ssh daemon listen port from 22 to something else in an attempt to avoid open ports scans:

`Port 3489`

_Note: if you change the default ssh port, you need to update ufw rules; example: `ufw allow 3489`. Connect with `ssh newuser@server_ip -p 3489`._

Save the file with `CTRL+X` > `Y` > press Enter.

### 3.2 Test connection

Restart the sshd daemon:

`systemctl restart sshd`

Open another terminal tab/window and try to SSH to the server:

`ssh newuser@server_ip`

If it doesn’t work, check for configuration errors using the other terminal and then try again.

If successful, you will be able to ssh with `newuser` to the server from now on, and install any software you want.

