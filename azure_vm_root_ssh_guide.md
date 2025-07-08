# Enable Root SSH Access on Azure VM

This guide explains how to enable root SSH access on Azure Linux VMs for development purposes.

## Prerequisites
- Azure VM with SSH key authentication already configured
- Access to the VM with a non-root user account

## Steps

**1. SSH into your VM with your current user:**
```bash
ssh -i /path/to/your/private/key luci@your-vm-ip
```

**2. Switch to root user:**
```bash
sudo su -
```

**3. Check if root access is currently disabled:**
```bash
grep root /etc/shadow
```
If you see "LOCK" in the output, it means root access is disabled by default.

**4. Enable root access by setting a password:**
```bash
passwd root
```
Enter a strong password when prompted.

**5. Verify root access is now enabled:**
```bash
grep root /etc/shadow
```
You should now see the password hash instead of "LOCK".

**6. Configure SSH to allow root login:**
Edit the SSH configuration file:
```bash
nano /etc/ssh/sshd_config
```

**7. Find and modify these lines:**
- Change `PermitRootLogin no` to `PermitRootLogin yes`
- Optionally, if you want to keep using SSH keys, use `PermitRootLogin without-password` instead

**8. Copy your SSH key to root (if using key-based authentication):**
```bash
mkdir -p /root/.ssh
cp /home/luci/.ssh/authorized_keys /root/.ssh/
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

**9. Restart SSH service:**
```bash
systemctl restart sshd
```

**10. Test root SSH access:**
From your local machine:
```bash
ssh -i /path/to/your/private/key root@your-vm-ip
```

## Security Note
⚠️ **Important**: Azure Linux images disable root user access by default for security reasons. This configuration is recommended for development environments only. For production environments, use sudo access with your regular user account instead of enabling direct root SSH access.

## Troubleshooting
- If SSH connection fails, check the SSH service status: `systemctl status sshd`
- Verify the SSH configuration syntax: `sshd -t`
- Check firewall rules if needed: `ufw status` (Ubuntu) or `firewall-cmd --list-all` (CentOS/RHEL)

## Compatible Linux Distributions
This guide works with most Linux distributions on Azure VMs including:
- Ubuntu
- CentOS
- RHEL
- Debian
- SUSE Linux