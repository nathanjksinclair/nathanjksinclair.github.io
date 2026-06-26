# Windows 11 SMB Network Share Configuration Guide

## Overview

This document details the PowerShell configurations needed to resolve **Access Denied** errors when connecting **Windows 11** clients to **Linux** **SMB** (Samba) network shares.

Recent **Windows 11** updates, especially version 24H2, enforce stricter security defaults that can block compatibility with standard **Linux** share configurations.

## Prerequisites

- **Administrator access**: Run **PowerShell** with elevated privileges.
- **Target system**: Apply the commands on the **Windows 11** client machine that is trying to access the share.
- **Network context**: These steps are intended for trusted private networks where **Linux** shares may use guest access or may not require mandatory **SMB** signing.

## Configuration Steps

### 1. Open PowerShell as Administrator

1. Right-click the **Start** button.
2. Select **Terminal (Admin)** or **Windows PowerShell (Admin)**.
3. Confirm the User Account Control (UAC) prompt if it appears.

### 2. Disable Mandatory SMB Security Signatures

**Windows 11** now mandates **SMB** signing by default. Most **Linux** **Samba** servers do not support this requirement, which can lead to connection rejections. Run the following command to disable the mandatory requirement on the client:

```powershell
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
```

If the **Windows** machine is also hosting shares, you may need to disable this on the server side as well:

```powershell
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force
```

### 3. Enable Insecure Guest Logons

By default, **Windows 11** blocks unauthenticated guest access to network shares. If your **Linux** share does not require a username and password, explicitly allow this behavior:

```powershell
Set-SmbClientConfiguration -EnableInsecureGuestLogons $true -Force
```

### 4. Verify Configuration

Confirm that the settings were applied correctly by running:

```powershell
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableInsecureGuestLogons
```

Expected output:

- `RequireSecuritySignature`: `False`
- `EnableInsecureGuestLogons`: `True`

## Troubleshooting Persistent Access Denied Errors

If the error persists after applying the PowerShell configurations, check the following common authentication conflicts.

### Credential Conflicts

**Windows** may try to use cached credentials that do not match the **Linux** server's expectations.

1. Clear cached credentials:

	```cmd
	cmdkey /list
	cmdkey /delete:TargetName
	```

2. Reconnect to the share and explicitly enter the **Linux** username and password.

	Note: You must use the actual **Linux** user password. **Windows PINs** and Windows Hello biometrics are not valid for network share authentication.

### IPv6 Interference

In some environments, **IPv6** routing issues can cause **SMB** negotiation failures.

1. Open **Network Connections** (`ncpa.cpl`).
2. Right-click your active adapter, then select **Properties**.
3. Uncheck **Internet Protocol Version 6 (TCP/IPv6)**.
4. Select **OK** and retry the connection.

### NTLM Authentication Levels

If the **Linux** server uses older authentication methods, adjust the local security policy:

1. Run `secpol.msc`.
2. Navigate to **Local Policies** > **Security Options**.
3. Set **Network security: LAN Manager authentication level** to **Send LM & NTLM - use NTLMv2 session security if negotiated**.

## Windows Share Guest or Everyone Access

If you are sharing a folder from a **Windows 11** computer and want simple local-network access, you can grant the share to **Everyone** or use guest-style access on a trusted private network. This is separate from SMB client settings and must be configured on the computer that is hosting the share.

### What Is Involved

1. Share the folder and add the **Everyone** group to the share permissions.
2. Make sure the folder's NTFS permissions also allow the intended users or groups.
3. If you want unauthenticated guest-style access, disable **Password protected sharing** on the host computer.
4. Keep **SMB** signing and firewall settings aligned with the rest of your network policy.

### Example Share Setup

1. Right-click the folder you want to share and open **Properties**.
2. Open the **Sharing** tab and select **Advanced Sharing**.
3. Enable **Share this folder**.
4. Open **Permissions** and add **Everyone** with the access level you want, such as **Read** or **Change**.
5. Open the **Security** tab and confirm the NTFS permissions match the share permissions.

### Guest-Style Access

If the goal is to allow access without a user name and password, open **Control Panel** > **Network and Sharing Center** > **Change advanced sharing settings** and turn off **Password protected sharing** for the private profile.

That setting makes the share behave more like a guest share on a trusted LAN, but it also reduces security and may not be permitted by all clients.

## SSH Key Authentication Between Two Windows 11 Computers

If you want passwordless SSH access from one **Windows 11** computer to another, the setup is different from SMB. You need an SSH server on the destination machine, an SSH client on the source machine, a public/private key pair, and the public key installed for the target user account.

### What Is Involved

1. Install or enable the **OpenSSH Server** feature on the destination computer.
2. Confirm the **OpenSSH Client** is available on the source computer.
3. Start the `sshd` service on the destination computer and set it to start automatically.
4. Allow inbound TCP port `22` through the **Windows Defender Firewall** on the destination computer.
5. Generate an SSH key pair on the source computer with `ssh-keygen`.
6. Copy the public key to the destination user's `authorized_keys` file.
7. Test the connection with `ssh` and confirm that the private key is used instead of a password.

### Example Setup

On the source computer, generate a key pair if you do not already have one:

```powershell
ssh-keygen -t ed25519 -C "your_name@source-computer"
```

Copy the public key from the source computer to the destination computer and place it in the target user's SSH folder:

```powershell
New-Item -ItemType Directory -Force $env:USERPROFILE\.ssh
notepad $env:USERPROFILE\.ssh\authorized_keys
```

Paste the contents of the source computer's `.pub` file into `authorized_keys`, then save the file.

If you are connecting to an account that is a member of the local Administrators group, Windows may instead use:

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

### Test the Connection

From the source computer, connect to the destination computer with:

```powershell
ssh destination-username@destination-hostname-or-ip
```

If the key is set up correctly, SSH should authenticate without prompting for the account password.

### Common Problems

- The `sshd` service is not running on the destination computer.
- Port `22` is blocked by the firewall or by network policy.
- The public key was copied to the wrong user profile.
- File permissions on `authorized_keys` are too open.
- The source computer is using a different private key than the one whose public key was installed on the destination computer.

## Security Considerations

Disabling **SMB** signing and enabling guest logons reduces the security posture of the **Windows** client. Apply these changes only on trusted internal networks.

SSH key authentication is safer than password-only access, but it still depends on protecting the private key on the source computer.

Do not apply these settings to devices connected to public or untrusted networks, as this increases exposure to man-in-the-middle and relay attacks.

## References

- [Microsoft documentation on SMB versions](https://learn.microsoft.com/en-us/windows-server/storage/file-server/troubleshoot/detect-enable-and-disable-smbv1-v2-v3)
- [PDQ: Set-SmbClientConfiguration](https://www.pdq.com/powershell/set-smbclientconfiguration/)
- [Microsoft PowerShell module reference](https://learn.microsoft.com/en-us/powershell/module/smbshare/set-smbclientconfiguration?view=windowsserver2025-ps)
- [IT Support Guides: Allow guest SMB connections](https://www.itsupportguides.com/knowledge-base/windows-11/windows-11-how-to-allow-guest-smb-connections-with-powershell/)
- [Dev.to: Enable SMB/CIFS file sharing support](https://dev.to/winsides/enable-smb-10-cifs-file-sharing-support-using-command-prompt-windows-powershell-43h6)
- [Search result for Windows 11 SMB client configuration PowerShell commands](https://search.brave.com/search?q=Windows%2011%20SMB%20client%20configuration%20PowerShell%20commands)
- [Microsoft OpenSSH documentation](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)

