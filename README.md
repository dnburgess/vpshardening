# VPS Hardening Example
This is an example of steps that you might take to help harden your VPS security.

This is in no way meant to be a be-all end-all solution, so be sure to test it. Verify it. I may have missed something and I take no responsibility for your server's security.

# Debian VPS Lockdown

This may not be 100% correct or complete. Verify these steps for yourself before you use them. I assume no responsibility for the security of your VPS or other setups.

## 1. Initial Server Setup

### 1.1. Update System Packages
Ensure your system is up-to-date:
```sudo apt update && sudo apt list --upgradable && sudo apt upgrade -y && sudo apt autoremove -y```

### 1.2. Set System Timezone
Accurate time is important for logs and scheduled tasks.

```sudo dpkg-reconfigure tzdata```

### 1.3. Create a Non-Root User
Never operate directly as root. Create a new user and give it sudo privileges.

```adduser <your_new_user>```

```sudo usermod -aG sudo <your_new_user>```

Log out from root and log back in as <your_new_user>:

## 2. Configure Firewall (UFW)
UFW (Uncomplicated Firewall) will manage our network rules.

### 2.1. Set Default Policies
Deny all incoming traffic and allow all outgoing traffic by default.

```sudo ufw default deny incoming```

```sudo ufw default allow outgoing```

### 2.2. Allow Initial SSH Connection (Port 22)
Temporarily allow SSH on the default port so you don't get locked out before changing it.

```sudo ufw allow OpenSSH```

#### This is equivalent to 'sudo ufw allow 22/tcp'
### 2.3. Allow Pangolin Ports
Pangolin requires specific ports for its operation:

- TCP Port 443 (HTTPS): For Pangolin's web UI and SSL-secured resources.
- UDP Port 51820 (WireGuard): Default port for WireGuard tunnels used by Pangolin.
- TCP Port 80 (HTTP): (Optional) May be needed for SSL certificate validation (e.g., Let's Encrypt HTTP-01 challenge). If your SSL strategy (like DNS-01) doesn't require it, you might omit this.

```sudo ufw allow https```   # For Pangolin UI (TCP 443)

```sudo ufw allow 51820/udp``` # For Pangolin WireGuard Tunnels
#### sudo ufw allow http    # Optional for SSL validation (TCP 80)
### 2.4. Enable UFW
```sudo ufw enable```

Check the status to ensure rules are active:

```sudo ufw status verbose```

You should see rules for OpenSSH, HTTPS, and UDP 51820 (and HTTP if you added it).

## 3. Secure SSH Access
### 3.1. Test Current Non-Root User SSH Access

⚠️ Important: Before making SSH changes, open a new terminal window and confirm you can log in as <your_new_user> on port 22.

```ssh <your_new_user>@<your_vps_ip>```

If successful, proceed. Keep this original SSH session open as a backup until all SSH changes are tested.

### 3.2. Generate SSH Key Pair (On Your Local Machine)
If you don't already have an SSH key pair, generate one on your local computer (not the VPS).


#### Choose one of the following:
```ssh-keygen -t ed25519 -C "your_email@example.com_or_description"``` (without quotes)
#### OR for RSA:
```ssh-keygen -t rsa -b 4096 -C "your_email@example.com_or_description"``` (without quotes)

Follow the prompts. It's recommended to use a strong passphrase.

### 3.3. Copy Public Key to VPS

Use ssh-copy-id from your local machine to copy your public key to the VPS. 

Replace <your_new_user> and <your_vps_ip>. 

If your key is not the default id_ed25519 or id_rsa, specify it with -i ~/.ssh/your_key_name.pub.

```ssh-copy-id -i ~/.ssh/id_ed25519.pub <your_new_user>@<your_vps_ip>```
#### Or if you used a custom key name:
```ssh-copy-id -i ~/.ssh/my_custom_key.pub <your_new_user>@<your_vps_ip>```

You'll be prompted for <your_new_user>'s password on the VPS.

Alternatively, you can manually add the key.

On your local machine, display your public key:

```cat ~/.ssh/id_ed25519.pub # Or your_key_name.pub```
Copy the output.

On the VPS (as <your_new_user>):

```mkdir -p ~/.ssh```

```chmod 700 ~/.ssh```

```echo "paste_your_public_key_string_here" >> ~/.ssh/authorized_keys```

```chmod 600 ~/.ssh/authorized_keys```

### 3.4. Modify SSH Server Configuration on VPS
Edit the SSH daemon configuration file:

```sudo nano /etc/ssh/sshd_config```

Make the following changes (uncomment lines by removing # if necessary, or add them):

- Change Port: Choose a unique port number (e.g., above 1024, not used by other services). Example: Port 2222
- AddressFamily inet: This restricts SSH to IPv4. If you need IPv6 access via SSH, comment this line out or set it to any. For most Pangolin setups focusing on tunneling, IPv4 for SSH is often sufficient.
- PermitRootLogin no: Crucial for security.
- PubkeyAuthentication yes: Ensures key-based authentication is enabled.
- AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2: Standard location for public keys.
- PasswordAuthentication yes: Leave this as yes for now. You will change this to no after successfully testing login with the new port and key.
- PermitEmptyPasswords no: Good security practice.
- (Optional) ChallengeResponseAuthentication no: Further hardens by disabling challenge-response authentication if not needed.
- (Optional) UsePAM no: If you are only using key-based authentication and not other PAM modules for SSH. Review carefully if you have specific PAM needs. For a simple setup, UsePAM no along with PasswordAuthentication no and ChallengeResponseAuthentication no can be more secure.

Save the file and exit nano (Ctrl+X, then Y, then Enter).

### 3.5. Restart SSH Service

```sudo systemctl restart ssh```

### 3.6. Confirm SSH is Listening on the New Port
```sudo ss -tulpn | grep ssh```

- Or: ```sudo netstat -tulpn | grep ssh```
- Or: ```sudo lsof -i :<your_new_ssh_port>```

You should see the SSH service listening on <your_new_ssh_port>.

### 3.7. Update UFW for New SSH Port
Allow your new SSH port and remove the old rule.

```sudo ufw allow <your_new_ssh_port>/tcp```

```sudo ufw delete allow OpenSSH``` # Or ```sudo ufw delete allow 22/tcp```

```sudo ufw status verbose```

Ensure the new port is allowed and the old one is no longer listed (unless you have a specific reason to keep port 22 open temporarily, which is generally not recommended long-term).

### 3.8. Test SSH with New Port and Key
⚠️ Important: From a new terminal window on your local machine, try logging in using your SSH key and the new port.

```ssh -p <your_new_ssh_port> <your_new_user>@<your_vps_ip>```

If you can log in successfully without being prompted for a password (only your key passphrase, if you set one), then proceed. If it fails, troubleshoot using your original open SSH session.

### 3.9. Disable Password Authentication (Final SSH Hardening Step)
Once you've confirmed key-based login on the new port works perfectly:
Edit sshd_config again:

```sudo nano /etc/ssh/sshd_config```

Change:

- PasswordAuthentication no
Save the file and restart SSH:

```sudo systemctl restart ssh```

Test login again from a new terminal to be certain. You should no longer be able to log in with a password.

## 4. Set Up Automatic Security Updates
### 4.1. Configure Unattended Upgrades
This will automatically install security updates.

```sudo apt install unattended-upgrades apt-listchanges```

```sudo dpkg-reconfigure --priority=low unattended-upgrades```

You can configure apt-listchanges to email you about updates by editing /etc/apt/listchanges.conf.
Review the configuration in /etc/apt/apt.conf.d/50unattended-upgrades to customize (e.g., enable auto-reboot if needed).

## 5. Ongoing Security Practices

Regular Backups: Implement a robust backup strategy for your VPS, especially critical configuration files and any persistent data for Pangolin.

Minimize Software: Only install necessary software to reduce the attack surface.

Keep Software Updated: Follow Software's documentation for updates.

You are now ready to install and configure Software on your hardened Debian VPS!
