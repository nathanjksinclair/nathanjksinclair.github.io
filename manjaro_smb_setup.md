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

## Security Notes

Avoid using guest shares unless you trust everyone on the network.

For better control, prefer authenticated shares with per-user access and tighter filesystem permissions.

If you expose Samba beyond a private LAN, also consider encrypting traffic, limiting allowed hosts, and using a stronger firewall policy.
