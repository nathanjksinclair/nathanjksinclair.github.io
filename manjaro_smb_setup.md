# Manjaro SMB Share Setup Guide

## Overview

This guide covers how to create and publish **SMB** (Samba) network shares from **Manjaro Linux** so that other devices on your local network can access them.

It includes package installation, Samba service setup, share configuration, user authentication, and a simple way to test the share from another machine.

## Prerequisites

- A **Manjaro** system with administrator access.
- A folder you want to share over the network.
- A local user account that will be allowed to access the share.
- A trusted private network. Do not expose a basic file share directly to the public internet.

## 1. Install Samba

Install the Samba server package from Manjaro's repositories:

```bash
sudo pacman -S samba
```

If you want command-line client tools for testing shares from the Manjaro machine itself, also install the Samba client utilities:

```bash
sudo pacman -S smbclient
```

## 2. Prepare the Shared Folder

Choose or create a directory to share. For example:

```bash
sudo mkdir -p /srv/samba/share
sudo chown -R root:users /srv/samba/share
sudo chmod -R 2775 /srv/samba/share
```

If you want a specific user to own the files instead of the `users` group, adjust the owner and group to match your local setup.

## 3. Add a Samba User

Samba uses its own password database. Add the Linux user to Samba with:

```bash
sudo smbpasswd -a yourusername
```

Then enable that account:

```bash
sudo smbpasswd -e yourusername
```

## 4. Configure the Share

Edit the Samba configuration file:

```bash
sudo nano /etc/samba/smb.conf
```

Add a share definition like this near the end of the file:

```ini
[SharedFolder]
    path = /srv/samba/share
    browseable = yes
    read only = no
    guest ok = no
    valid users = yourusername
    create mask = 0664
    directory mask = 2775
```

### Optional Guest Share

If you want guest access on a trusted local network, you can use a separate share definition like this:

```ini
[GuestShare]
    path = /srv/samba/share
    browseable = yes
    read only = no
    guest ok = yes
    force user = nobody
```

Guest access is less secure, so use it only when you really want unauthenticated network access.

## 5. Validate the Configuration

Check the Samba configuration for syntax errors:

```bash
testparm
```

If `testparm` reports no errors, the configuration is ready to use.

## 6. Start and Enable Samba Services

Enable the Samba services so they start automatically at boot:

```bash
sudo systemctl enable --now smb nmb
```

If you only need modern SMB file sharing and not legacy NetBIOS browsing, `smb` is usually the most important service.

## 7. Open the Firewall

If you use a firewall on Manjaro, allow SMB traffic on the local network.

For `ufw`:

```bash
sudo ufw allow samba
```

For `firewalld`:

```bash
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload
```

If you do not use a firewall, you can skip this step.

## 8. Test the Share

From another computer, connect to the share using the Manjaro machine's IP address or hostname.

Example from Linux:

```bash
smbclient //manjaro-hostname/SharedFolder -U yourusername
```

Example from Windows:

```text
\\manjaro-hostname\SharedFolder
```

Enter the Samba username and password when prompted.

## 9. Host Bare Git Repositories on the Server

If you want the Manjaro machine to act as a Git host, create a dedicated folder for bare repositories on the server itself. Other computers should access those repositories with `git clone`, usually over SSH.

Bare repositories do not contain a working copy. They are meant to be the central server-side copy that clients clone from and push to.

### Create a Repository Folder

```bash
sudo mkdir -p /srv/git
sudo chown -R yourusername:users /srv/git
sudo chmod -R 2775 /srv/git
```

If you want the repository folder to be owned by a service account instead of your personal account, use that user and group consistently for all repo operations.

### Create or Clone a Bare Repository

If you want to mirror an existing GitHub repository into the server folder, use a bare clone:

```bash
git clone --bare https://github.com/owner/example-project.git /srv/git/example-project.git
```

You can repeat this for each project you want to host on the server.

### Periodically Fetch Updates From GitHub

If the server copy should stay in sync with GitHub, run `git fetch` inside each bare repository. A bare clone from GitHub already has a remote named `origin`, so it can be updated with:

```bash
cd /srv/git/example-project.git
git fetch origin --prune --tags
```

If you created the repository first and added GitHub later, add the remote once:

```bash
cd /srv/git/example-project.git
git remote add github https://github.com/owner/example-project.git
git fetch github --prune --tags
```

To update every bare repository on a schedule, use a small script such as:

```bash
#!/bin/bash
for repo in /srv/git/*.git; do
    [ -d "$repo" ] || continue
    git -C "$repo" fetch origin --prune --tags
done
```

Save it as something like `/usr/local/bin/update-git-mirrors.sh`, make it executable, and run it from `cron` or a systemd timer if you want the fetch to happen automatically.

### Enable SSH Access

Install and start the SSH server so other machines can reach the repository folder:

```bash
sudo pacman -S openssh
sudo systemctl enable --now sshd
```

If you use a firewall, allow SSH traffic on the local network:

```bash
sudo ufw allow ssh
```

or:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### Clone From Another Computer

From another Linux machine, clone the repository with an SSH path:

```bash
git clone yourusername@manjaro-hostname:/srv/git/example-project.git
```

If SSH key authentication is set up, the clone can proceed without prompting for a password.

After cloning, work in your local repository and push changes back to the server with normal Git commands.

## Troubleshooting

### The Share Does Not Appear

- Confirm `smb` is running with `systemctl status smb`.
- Make sure the configuration passes `testparm`.
- Check that the network allows SMB traffic.

### Access Denied

- Confirm the Linux folder permissions allow the user access.
- Make sure the Samba user was added with `smbpasswd -a`.
- Check that `valid users` matches the account you are using.

### Guest Access Does Not Work

- Confirm `guest ok = yes` is present in the share definition.
- Check that the folder permissions allow the guest mapping user to read or write.
- Make sure client-side security settings are not blocking guest access.

### Git Operations Fail Over the Network

- Confirm the repository is a bare repository created with `git init --bare`.
- Make sure the user that is pushing has write access to the repository folder and its files.
- Confirm the SSH service is running and port 22 is allowed through the firewall.
- If you cloned from GitHub into the server with `git clone --bare`, verify the remote URL and repository path before pushing from clients.
- If performance is poor, keep the repository on local disk and use SSH for Git transport instead of mounting the folder over SMB.

## Security Notes

Avoid using guest shares unless you trust everyone on the network.

For better control, prefer authenticated shares with per-user access and tighter filesystem permissions.

If you expose Samba beyond a private LAN, also consider encrypting traffic, limiting allowed hosts, and using a stronger firewall policy.
