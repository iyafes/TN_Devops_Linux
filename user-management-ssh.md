# Linux User and Group Management & Permissions

## User Management

**adduser** - Create new user (interactive)
```bash
sudo adduser john              # Create user with home directory (interactive prompts)
sudo adduser --no-create-home username  # Create user without home directory
```

**useradd** - Create new user (non-interactive)
```bash
sudo useradd username          # Basic user creation
sudo useradd -m username       # Create user with home directory
sudo useradd -m -s /bin/bash username  # Create user with home dir and shell
```

**cat /etc/passwd** - List all users
```bash
cat /etc/passwd                # Display all system users
cat /etc/passwd | grep username  # Find specific user
```

**usermod** - Modify user account
```bash
sudo usermod -c "Full Name" username       # Change user comment/full name
sudo usermod -l new_name old_name          # Change username
sudo usermod -d /new/home username         # Change home directory
sudo usermod -aG sudo username             # Add user to sudo group
sudo usermod -aG group1,group2 username    # Add user to multiple groups
sudo usermod --expiredate 2024-12-31 username  # Set account expiration date
```

**passwd** - Manage passwords
```bash
sudo passwd username           # Change user password
sudo passwd -l username        # Lock user account (disable login)
sudo passwd -u username        # Unlock user account
```

**chage** - Change password aging
```bash
sudo chage -d 0 username       # Force password change on next login
sudo chage -l username         # Display password aging information
sudo chage -M 90 username      # Set password to expire in 90 days
```

**deluser** - Delete user
```bash
sudo deluser username          # Remove user (keep home directory)
sudo deluser --remove-home username  # Remove user and home directory
```

**userdel** - Delete user (alternative)
```bash
sudo userdel username          # Remove user
sudo userdel -r username       # Remove user and home directory
```

---

## User Information

**id** - Display user and group IDs
```bash
id                             # Current user ID and groups
id username                    # Specific user information
id -u username                 # User ID only
id -g username                 # Primary group ID
```

**finger** - Display user information
```bash
finger username                # Show user details (may need installation)
```

**last** - Login history
```bash
last username                  # Show user's login history
last                           # Show all users' login history
last -n 10                     # Show last 10 logins
```

**w** - Who is logged in
```bash
w                              # Show logged in users and their activity
```

**who** - Display logged in users
```bash
who                            # Show currently logged in users
```

---

## Group Management

**addgroup** - Create new group
```bash
sudo addgroup groupname        # Create group
sudo addgroup --gid 1001 groupname  # Create group with specific GID
```

**groupadd** - Create new group (alternative)
```bash
sudo groupadd groupname        # Create group
sudo groupadd -g 1001 groupname  # Create with specific GID
```

**cat /etc/group** - List all groups
```bash
cat /etc/group                 # Display all system groups
cat /etc/group | grep groupname  # Find specific group
```

**adduser to group** - Add user to group
```bash
sudo adduser username groupname  # Add user to group (Debian/Ubuntu)
```

**gpasswd** - Group password and membership
```bash
sudo gpasswd -a username groupname  # Add user to group
sudo gpasswd -d username groupname  # Remove user from group
```

**usermod for groups** - Modify user group membership
```bash
sudo usermod -aG groupname username  # Add user to supplementary group
sudo usermod -aG group1,group2 username  # Add to multiple groups
sudo usermod -G groupname username   # Set user's groups (removes from others)
```

**groupmod** - Modify group
```bash
sudo groupmod -n new_name old_name  # Rename group
sudo groupmod -g 1002 groupname     # Change group GID
```

**getent** - Display group members
```bash
getent group groupname         # Show all members of a group
```

**delgroup** - Delete group
```bash
sudo delgroup groupname        # Remove group
```

**groupdel** - Delete group (alternative)
```bash
sudo groupdel groupname        # Remove group
```

---

## Sudo Privileges

**sudo** - Execute command as superuser
```bash
sudo command                   # Run command with root privileges
sudo -i                        # Login as root user
sudo -u username command       # Run command as another user
sudo su -                      # Switch to root user
```

**Grant sudo access** - Add user to sudo group
```bash
sudo usermod -aG sudo username # Add user to sudo group (Debian/Ubuntu)
sudo usermod -aG wheel username # Add user to wheel group (CentOS/RHEL)
```

**visudo** - Edit sudoers file safely
```bash
sudo visudo                    # Edit /etc/sudoers file (with syntax checking)
```

**Sudoers file configuration** - /etc/sudoers
```bash
# Give user full sudo privileges
username ALL=(ALL:ALL) ALL

# Allow user to run all commands without password
username ALL=(ALL:ALL) NOPASSWD: ALL

# Allow user to run specific command without password
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Allow user to run custom script without password
username ALL=(ALL) NOPASSWD: /path/to/script.sh

# Allow group to have sudo privileges
%groupname ALL=(ALL:ALL) ALL
```

---

## File Permissions

### Permission Groups
- **u** (user/owner) - File owner
- **g** (group) - Group members
- **o** (others) - All other users
- **a** (all) - All users (owner, group, others)

### Permission Types
- **r** (read) - Read file content or list directory
- **w** (write) - Modify file or directory
- **x** (execute) - Execute file or access directory

### Numeric Permissions
- **4** = Read (r)
- **2** = Write (w)
- **1** = Execute (x)
- **0** = No permission

**Common combinations:**
- **7** = rwx (4+2+1) - Read, write, execute
- **6** = rw- (4+2) - Read, write
- **5** = r-x (4+1) - Read, execute
- **4** = r-- (4) - Read only
- **0** = --- - No permission

---

## chmod - Change File Permissions

**Numeric (octal) mode:**
```bash
chmod 755 file.txt             # rwxr-xr-x (owner: rwx, group: rx, others: rx)
chmod 644 file.txt             # rw-r--r-- (owner: rw, group: r, others: r)
chmod 700 file.txt             # rwx------ (owner: rwx, others: none)
chmod 600 file.txt             # rw------- (owner: rw, others: none)
chmod 777 file.txt             # rwxrwxrwx (all: full permissions - NOT recommended)
chmod -R 755 directory         # Apply recursively to directory
```

**Symbolic mode:**
```bash
chmod u+x file.sh              # Add execute for owner
chmod g+w file.txt             # Add write for group
chmod o-r file.txt             # Remove read for others
chmod a+x file.sh              # Add execute for all
chmod u+rwx,g+rx,o+r file.txt  # Set multiple permissions
chmod u=rwx,g=rx,o=r file.txt  # Set exact permissions (= instead of +)
chmod +x script.sh             # Add execute for all (shorthand)
chmod -x script.sh             # Remove execute for all
```

**Common permission examples:**
```bash
chmod 755 script.sh            # Standard executable script
chmod 644 document.txt         # Standard readable file
chmod 600 private.key          # Private file (owner only)
chmod 700 ~/.ssh               # SSH directory
chmod 600 ~/.ssh/id_rsa        # SSH private key
chmod 644 ~/.ssh/id_rsa.pub    # SSH public key
```

---

## chown - Change File Ownership
```bash
chown user file.txt            # Change owner only
chown user:group file.txt      # Change owner and group
chown :group file.txt          # Change group only
chown -R user:group directory  # Change recursively
chown --reference=ref_file target_file  # Match permissions from reference file
```

---

## chgrp - Change Group Ownership
```bash
chgrp groupname file.txt       # Change file group
chgrp -R groupname directory   # Change recursively
```

---

## umask - Default Permission Mask
```bash
umask                          # Show current umask (e.g., 0022)
umask 022                      # Set umask (files: 644, directories: 755)
umask 077                      # Set umask (files: 600, directories: 700)
```

**How umask works:**
- Default file permissions: 666 (rw-rw-rw-)
- Default directory permissions: 777 (rwxrwxrwx)
- Umask 022: Files become 644, directories become 755
- Umask 077: Files become 600, directories become 700

---



## System Shutdown and Reboot

**init** - Change runlevel
```bash
init 0                         # Halt the system (complete stop)
init 6                         # Reboot the system
```

**halt** - Halt the system
```bash
sudo halt                      # Halt the system immediately
```

**shutdown** - Schedule shutdown or reboot
```bash
sudo shutdown -h now           # Shutdown immediately
sudo shutdown -r now           # Restart immediately
sudo shutdown -h +10           # Shutdown in 10 minutes
sudo shutdown -h +1:20         # Shutdown in 1 hour and 20 minutes
sudo shutdown -h 22:00 "System will be shutting down for maintenance at 10:00 PM. Please save your work."  # Shutdown at specific time with message
sudo shutdown -r +1:40 "System will be rebooting in 1 hour and 40 minutes."  # Reboot with delay and message
```

**reboot** - Reboot the system
```bash
sudo reboot                    # Reboot the system immediately
```

---


## File Analysis and Search

**file** - Determine file type
```bash
file package.json              # Identify file format/type
```

**diff** - Compare files
```bash
diff -u file1.txt file2.txt    # Compare two files (unified format)
```

**head** - Display first lines
```bash
head file.txt                  # Display first 10 lines
head -n 5 file.txt             # Display first 5 lines
```

**tail** - Display last lines
```bash
tail file.txt                  # Display last 10 lines
tail -n 5 file.txt             # Display last 5 lines
```

**which** - Locate command executable
```bash
which -a ls                    # Show location of ls binary (/usr/bin/ls)
```

**find** - Search for files
```bash
find /path/to/search -name "*.txt"      # Find files with .txt extension
find /path/to/search -type f -mtime -7  # Find files modified in last 7 days
```

**grep** - Search text patterns
```bash
grep -i "pattern" file.txt     # Case-insensitive search for pattern
```

**man** - Display manual pages
```bash
man vim                        # Display manual for vim command
```

---


# SSH Installation and Configuration on Ubuntu

## What is SSH?

SSH (Secure Shell) is a network protocol that provides secure encrypted communication between a client and server. All data transmitted is encrypted, preventing theft and remote network attacks. Commonly used in UNIX-like systems for secure remote access.

---

## SSH Architecture

SSH uses public-key cryptography to authenticate the remote system and users. It operates on three hierarchical layers.

**Connection Initialization:**
1. Client initiates TCP connection to SSH server on port 22 (default)
2. Server responds and TCP handshake is established
3. Server sends identification string (protocol version, etc.)
4. Client sends its identification information
5. Server sends its public key to client
6. If server not in client's "known_hosts" file, user is prompted to trust it
7. Client generates random session key and encrypts it with server's public key
8. Server decrypts session key using its private key
9. Connection is now secured using the session key

**Authentication Steps:**
1. Client sends authentication information (e.g., public key) to server
2. Server verifies if public key is in "authorized_keys"
3. If not authorized, server rejects and notifies client
4. Client attempts authentication using username and password
5. Server validates credentials
6. Client and server can now exchange encrypted data and commands

---

## Step 1: Install SSH on Ubuntu
```bash
sudo apt update                # Update package lists
sudo apt install openssh-server # Install OpenSSH server
```

---

## Step 2: Start and Enable SSH Service
```bash
sudo systemctl status ssh      # Check SSH service status
sudo systemctl enable --now ssh # Enable and start SSH service
sudo systemctl start ssh       # Start SSH service
sudo systemctl stop ssh        # Stop SSH service
sudo systemctl enable ssh      # Enable SSH at boot
sudo systemctl disable ssh     # Disable SSH at boot
```

---

## Step 3: Configure Custom SSH Port

**Backup configuration file before editing:**
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config_back  # Backup config
```

**Edit SSH configuration:**
```bash
sudo vim /etc/ssh/sshd_config  # Edit SSH config file
```

**Security configurations to apply:**
1. Disable empty passwords: `PermitEmptyPasswords no`
2. Change default SSH port: `Port 2345`
3. Disable root login: `PermitRootLogin no`
4. Configure idle timeout: `ClientAliveInterval 300`
5. Disable SSH protocol 1: `Protocol 2`
6. Allow specific users only: `AllowUsers User1 User2`
7. Disable password-based login (optional)

**Restart SSH service after changes:**
```bash
sudo systemctl restart ssh     # Apply configuration changes
```

---

## Step 4: Allow SSH Through Firewall

**Configure UFW (Uncomplicated Firewall):**
```bash
sudo ufw allow from any to any port 2222 proto tcp  # Allow SSH on custom port
sudo ufw allow 22/tcp          # Allow SSH on default port 22
sudo ufw allow ssh             # Allow SSH service
sudo ufw status                # Check firewall status
sudo ufw enable                # Enable firewall
```

---
