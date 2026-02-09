# TN_Devops_Linux
Linux for Devops 

# Basic Linux Commands

---

## Directory Navigation

**pwd** - Print working directory
```bash
pwd                        # Display current directory full path
```

**cd** - Change directory
```bash
cd Desktop/                # Change to Desktop directory
cd .                       # Stay in current directory (no change)
cd ..                      # Move to parent directory
cd ~                       # Move to home directory
cd -                       # Return to previous directory (toggle between two)
```

---

## File and Directory Listing

**ls** - List directory contents
```bash
ls                         # List files in current directory
ls -l                      # Long format (permissions, owner, size, date)
ls -a                      # Show all files including hidden (. files)
ls -lh                     # Long format with human-readable sizes
ls -ltr                    # Long format, sorted by time (oldest first)
ls ltr                     # WRONG: Missing dash (-), should be 'ls -ltr'
```

---

## File and Directory Management

**mkdir** - Create directories
```bash
mkdir testdir              # Create single directory
mkdir -p test1/test2/test3 # Create nested directories (parents auto-created)
```

**rmdir** - Remove empty directories
```bash
rmdir testdir              # Remove empty directory only
rmdir test1                # FAILS: Won't work if directory has contents
                           # Use 'rm -r test1' to remove non-empty directory
```

**touch** - Create file or update timestamp
```bash
touch newfile.txt          # Create new empty file
touch newfile.txt          # Update timestamp of existing file to current time
```

---

## File Viewing

**cat** - Display file content
```bash
cat L2TP.txt               # Display entire file content

```

---

## Date and Time

**date** - Display or set date/time
```bash
date                       # Show current date and time
date +"%d-%m-%y %H:%M:%S"  # Custom format: day-month-year hour:min:sec
TZ="America/New_York" date # Show time in different timezone
```

**timedatectl** - Control system time and timezone
```bash
timedatectl                # Show current time and timezone settings
timedatectl list-timezones # List all available timezones
```

**cal** - Display calendar
```bash
cal                        # Show current month calendar
cal 2024                   # Show entire year calendar
cal -3                     # Show previous, current, next month
```

---

## Command History

**history** - View command history
```bash
history                    # Show all previous commands with numbers
history 10                 # Show last 10 commands
!34                        # Execute command number 34 from history
!!                         # Execute last command again
```

---

## System Information

**uname** - Display system information
```bash
uname                      # Show kernel name (Linux)
uname -a                   # Show all system information (kernel, hostname, version, architecture)
```

**hostname** - Display system hostname
```bash
hostname                   # Show system hostname
```

**uptime** - System uptime and load
```bash
uptime                     # Show how long system running + load average
```

**whoami** - Current username
```bash
whoami                     # Display current logged-in username
```

**who** - Logged in users
```bash
who                        # Show all currently logged-in users
```

---

## Process Management

**ps** - List processes
```bash
ps                         # Show your current terminal processes
ps aux                     # Show all processes (all users, detailed)
                           # a=all users, u=user-oriented, x=include non-terminal
ps -ef                     # Show all processes in full format
ps aux | grep nginx        # Find nginx processes (pipe to grep for filtering)
```

**top** - Real-time process monitor
```bash
top                        # Interactive process viewer (q to quit)
                           # Shows CPU, memory usage, running processes
```

**htop** - Enhanced process viewer
```bash
htop                       # Better than top (colorful, easier navigation)
                           # Needs installation: sudo apt install htop
```

**command &** - Run in background
```bash
command &                  # EXAMPLE: Run command in background
                           # Actual usage: firefox &, script.sh &
```

---

## Disk and Memory Usage

**df** - Disk space (filesystem level)
```bash
df                         # Show disk space for all filesystems
df -h                      # Human-readable format (GB, MB instead of bytes)
```

**du** - Directory space usage
```bash
du                         # Show size of current directory and subdirectories
du -h                      # Human-readable sizes (KB, MB, GB)
du -sh directory           # Summary of directory total size only
du -h --max-depth=1        # Show size one level deep
```

**free** - Memory usage
```bash
free                       # Show RAM and swap usage in kilobytes
free -h                    # Human-readable format (GB, MB)
free -m                    # Show in megabytes
```

---


## File Search

**find** - Search for files/directories
```bash
find -size +100M           # Find files larger than 100MB in current directory
find -size +10M            # Find files larger than 10MB
find -type d               # Find directories only
```

**which** - Locate command executable path
```bash
which ls                   # Show path to ls command (/usr/bin/ls)
which -a python            # Show all python executable paths in PATH
which -a java              # Show all java executable paths
which -a chrome            # Show all chrome executable paths
```

---

## Network Configuration

**ip** - Modern network configuration tool
```bash
ip a                       # Show all network interfaces and IP addresses
ip addr show               # Same as 'ip a' (verbose form)
ip link show               # Show network interfaces status (up/down)
ip route                   # Show routing table
```

**ping** - Test network connectivity
```bash
ping www.google.com        # Continuous ping (Ctrl+C to stop)
ping -c 4 8.8.8.8          # Send only 4 packets then stop
ping -i 2 host             # Ping with 2 second interval between packets
                           # Replace 'host' with actual hostname or IP
```

---

## Environment Variables

**env** - Display environment variables
```bash
env                        # List all environment variables
```

**echo** - Display text or variable value
```bash
echo $VAR                  # Display value of VAR variable
                           # If not set, shows nothing
```

---

## User Management

**sudo user** - WRONG command
```bash
sudo -i                    # sudo -i (login as root)
```

---

## Command Shortcuts (Aliases)

**alias** - Create command shortcuts
```bash
alias ll='ls -la'          # Create shortcut: ll now runs 'ls -la'
alias                      # List all current aliases
```

**unalias** - Remove alias
```bash
unalias ll                 # Remove the 'll' alias
```

---

## System Control

**init** - Change system runlevel
```bash
init 0                     # Shutdown system (runlevel 0)
                           # Needs sudo: sudo init 0
init 6                     # Reboot system (runlevel 6)
```

**exit** - Exit current shell/terminal
```bash
exit                       # Close terminal session or logout
```

---

## Terminal Control

**clear** - Clear terminal screen
```bash
clear                      # Clear screen (Ctrl+L also works)
```

---
