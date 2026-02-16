# Storage Management in Linux

## Partitioning Schemes

**MBR (Master Boot Record)** and **GPT (GUID Partition Table)** are two partitioning schemes used to organize and manage data on storage devices (HDDs and SSDs).

---

## LVM (Logical Volume Manager) Basics

**lsblk** - List block devices
```bash
sudo lsblk                     # Display all block devices and partitions
```

**fdisk** - Disk partitioning tool
```bash
sudo fdisk /dev/sdb            # Open disk for partitioning
sudo fdisk -l /dev/sdb         # List partitions on specific disk
```

---

## Physical Volume (PV) Management

**pvcreate** - Create physical volume
```bash
sudo pvcreate /dev/sdb1        # Initialize partition as physical volume
```

**pvdisplay** - Display physical volume information
```bash
sudo pvdisplay                 # Show details of all physical volumes
```

**pvs** - Display physical volume summary
```bash
sudo pvs                       # Show brief PV information
```

---

## Volume Group (VG) Management

**vgcreate** - Create new volume group
```bash
sudo vgcreate vg0 /dev/sdb1    # Create new volume group named vg0
sudo vgcreate vg0 /dev/sdb1 /dev/sdc1  # Create VG with multiple PVs
```

**vgextend** - Extend existing volume group
```bash
sudo vgextend vg0 /dev/sdb1    # Add physical volume to existing VG
```

**vgdisplay** - Display volume group information
```bash
sudo vgdisplay                 # Show details of all volume groups
sudo vgdisplay vg0             # Show specific volume group details
```

**vgs** - Display volume group summary
```bash
sudo vgs                       # Show brief VG information
```

---

## Logical Volume (LV) Management

**lvcreate** - Create new logical volume
```bash
sudo lvcreate -L 10G -n lv0 vg0          # Create 10GB logical volume
sudo lvcreate -l 100%FREE -n lv0 vg0     # Use all available space
sudo lvcreate -L 50G -n lv_data vg0      # Create named logical volume
```

**lvextend** - Extend logical volume
```bash
sudo lvextend -L +10G /dev/vg0/lv0       # Add 10GB to logical volume
sudo lvextend -l +100%FREE /dev/vg0/lv0  # Use all free space
sudo lvextend -l +100%FREE /dev/mapper/vg0-lv--0  # Extend using mapper path
```

**lvdisplay** - Display logical volume information
```bash
sudo lvdisplay                 # Show details of all logical volumes
sudo lvdisplay /dev/vg0/lv0    # Show specific LV details
```

**lvs** - Display logical volume summary
```bash
sudo lvs                       # Show brief LV information
```

---

## Filesystem Operations

**mkfs** - Create filesystem
```bash
sudo mkfs.ext4 /dev/sdb1       # Format partition as ext4
sudo mkfs.ext4 /dev/vg0/lv0    # Format logical volume as ext4
sudo mkfs.xfs /dev/vg0/lv0     # Format as XFS filesystem
```

**resize2fs** - Resize ext2/ext3/ext4 filesystem
```bash
sudo resize2fs /dev/vg0/lv0              # Resize to match LV size
sudo resize2fs /dev/mapper/vg0-lv--0     # Resize using mapper path
```

**xfs_growfs** - Resize XFS filesystem
```bash
sudo xfs_growfs /mnt/mount_point         # Grow XFS filesystem
```

---

## Mounting and Disk Usage

**mount** - Mount filesystem
```bash
sudo mount /dev/sdb1 /mnt/sdb_mount      # Mount partition
sudo mount /dev/vg0/lv0 /mnt/data        # Mount logical volume
sudo mount -a                            # Mount all entries in /etc/fstab
```

**umount** - Unmount filesystem
```bash
sudo umount /mnt/sdb_mount               # Unmount by mount point
sudo umount /dev/sdb1                    # Unmount by device
```

**df** - Disk space usage
```bash
df -h                          # Show disk usage (human-readable)
df -h /mnt/sdb_mount           # Show usage for specific mount point
```

**mkdir** - Create mount point
```bash
sudo mkdir /mnt/sdb_mount      # Create directory for mounting
sudo mkdir -p /mnt/data        # Create mount point with parents
```

---

## Persistent Mounting with /etc/fstab

**blkid** - Get filesystem UUID
```bash
sudo blkid /dev/sdb1           # Display UUID and filesystem attributes
sudo blkid                     # Show all block device attributes
```

**Edit /etc/fstab** - Add persistent mount entry
```bash
sudo vim /etc/fstab            # Edit fstab file
```

**fstab entry format:**
```bash
UUID=c9516c06-7a49-441d-b1cb-6cc3db593217 /mnt/sdb_mount ext4 defaults 0 0
/dev/vg0/lv0  /mnt/data  ext4  defaults  0  0
```

---

## Complete Procedure: Add New Disk to Existing LVM

**Step 1:** List block devices
```bash
sudo lsblk
```

**Step 2:** Partition the new disk
```bash
sudo fdisk /dev/sdb
```

**Step 3:** List partitions
```bash
sudo fdisk -l /dev/sdb
```

**Step 4:** Create physical volume
```bash
sudo pvcreate /dev/sdb1
```

**Step 5:** Display volume groups
```bash
sudo vgdisplay
```

**Step 6:** Display physical volumes
```bash
sudo pvdisplay
```

**Step 7:** Extend volume group
```bash
sudo vgextend vg0 /dev/sdb1
```

**Step 8:** Display logical volumes
```bash
sudo lvdisplay
```

**Step 9:** Check current disk usage
```bash
sudo df -h
```

**Step 10:** Extend logical volume
```bash
sudo lvextend -l +100%FREE /dev/mapper/vg0-lv--0
```

**Step 11:** Resize filesystem
```bash
sudo resize2fs /dev/mapper/vg0-lv--0
```

**Step 12:** Verify disk usage
```bash
sudo df -h
```

---

## Complete Procedure: Create New VG and LV from Scratch

**Step 1:** List block devices
```bash
sudo lsblk
```

**Step 2:** Partition the disk
```bash
sudo fdisk /dev/sdb
```

**Step 3:** Create physical volume
```bash
sudo pvcreate /dev/sdb1
```

**Step 4:** Create volume group
```bash
sudo vgcreate vg0 /dev/sdb1
```

**Step 5:** Create logical volume
```bash
sudo lvcreate -L 50G -n lv0 vg0    # Create 50GB LV
# OR
sudo lvcreate -l 100%FREE -n lv0 vg0    # Use all space
```

**Step 6:** Create filesystem
```bash
sudo mkfs.ext4 /dev/vg0/lv0
```

**Step 7:** Create mount point
```bash
sudo mkdir /mnt/data
```

**Step 8:** Mount logical volume
```bash
sudo mount /dev/vg0/lv0 /mnt/data
```

**Step 9:** Verify mount
```bash
df -h /mnt/data
```

**Step 10:** Get UUID for persistent mounting
```bash
sudo blkid /dev/vg0/lv0
```

**Step 11:** Add to /etc/fstab
```bash
sudo vim /etc/fstab
# Add: /dev/vg0/lv0  /mnt/data  ext4  defaults  0  0
```

**Step 12:** Mount all fstab entries
```bash
sudo mount -a
```

**Step 13:** Verify persistent mount
```bash
df -h /mnt/data
```

---

## Complete Procedure: Mount Disk Without LVM

**Step 1:** List block devices
```bash
sudo lsblk
```

**Step 2:** Partition the disk
```bash
sudo fdisk /dev/sdb
```

**Step 3:** List block devices after partitioning
```bash
sudo lsblk
```

**Step 4:** Format the partition
```bash
sudo mkfs.ext4 /dev/sdb1
```

**Step 5:** Create mount point
```bash
sudo mkdir /mnt/sdb_mount
```

**Step 6:** Mount the partition
```bash
sudo mount /dev/sdb1 /mnt/sdb_mount
```

**Step 7:** Verify mount
```bash
df -h /mnt/sdb_mount
```

**Step 8:** Get UUID
```bash
sudo blkid /dev/sdb1
```

**Step 9:** Add to /etc/fstab
```bash
sudo vim /etc/fstab
# Add: UUID=c9516c06-7a49-441d-b1cb-6cc3db593217 /mnt/sdb_mount ext4 defaults 0 0
```

**Step 10:** Mount all fstab entries
```bash
sudo mount -a
```

**Step 11:** Verify persistent mount
```bash
df -h /mnt/sdb_mount
```

---



```markdown
# Apache Web Server Installation on Ubuntu

## Step 1: Install Apache

**Update package index and install Apache:**
```bash
sudo apt update                # Update package lists
sudo apt install apache2       # Install Apache web server
```

**Check Apache service status:**
```bash
sudo systemctl status apache2  # Check if Apache is running
sudo systemctl enable apache2  # Enable Apache at boot
sudo systemctl start apache2   # Start Apache service
sudo systemctl stop apache2    # Stop Apache service
sudo systemctl restart apache2 # Restart Apache service
sudo systemctl reload apache2  # Reload Apache configuration
```

---

## Step 2: Verify Apache Installation

**Access Apache in browser:**
```
http://server_ip_address
```

---

## Step 3: Create Application Root Directory

```bash
sudo mkdir -p /var/www/test.com  # Create document root directory
```

---

## Step 4: Create Sample index.html Page

**Create HTML file:**
```bash
sudo vim /var/www/test.com/index.html
```

**Sample HTML content:**
```html
<html>
    <head>
        <title>Welcome to XYZ Server!</title>
    </head>
    <body>
        <h1>Success! The your_domain server block is working!</h1>
    </body>
</html>
```

---

## Step 5: Create Virtual Host Configuration

**Create virtual host config file:**
```bash
sudo vim /etc/apache2/sites-available/test.com.conf
```

**Virtual host configuration:**
```apache
<VirtualHost *:80>
    ServerAdmin webmaster@test.com
    ServerName test.com
    ServerAlias www.test.com
    DocumentRoot /var/www/test.com/
    ErrorLog /var/log/apache2/test_error.log
    CustomLog /var/log/apache2/test_access.log combined
</VirtualHost>
```

---

## Step 6: Enable Site and Reload Apache

**Enable virtual host:**
```bash
sudo a2ensite test.com.conf    # Enable site configuration
sudo systemctl reload apache2  # Reload Apache
```

**Disable site:**
```bash
sudo a2dissite test.com.conf   # Disable site configuration
sudo systemctl reload apache2  # Reload Apache
```

---

## Step 7: Test and Restart Apache

**Test configuration and restart:**
```bash
sudo apache2ctl configtest     # Test configuration for syntax errors
sudo apachectl -k graceful     # Graceful restart (no downtime)
sudo systemctl restart apache2 # Hard restart Apache
```

---

## Additional Apache Commands

**Apache module management:**
```bash
sudo a2enmod module_name       # Enable Apache module
sudo a2dismod module_name      # Disable Apache module
sudo a2enmod rewrite           # Enable rewrite module (common)
```

**Apache configuration files:**
```bash
/etc/apache2/apache2.conf      # Main Apache configuration
/etc/apache2/sites-available/  # Available site configurations
/etc/apache2/sites-enabled/    # Enabled site configurations
/var/log/apache2/              # Apache log files
```

---

## Step 8: Additional Configuration

- Add A & CNAME records in DNS for your domain
- Configure firewall to allow HTTP/HTTPS traffic
- Apply additional Apache security and performance configurations

---
```


# Nginx Web Server Installation on Ubuntu

## Step 1: Install Nginx

**Update package index and install Nginx:**
```bash
sudo apt update                # Update package lists
sudo apt install nginx         # Install Nginx web server
```

**Check Nginx service status:**
```bash
sudo systemctl status nginx    # Check if Nginx is running
sudo systemctl enable nginx    # Enable Nginx at boot
sudo systemctl start nginx     # Start Nginx service
sudo systemctl stop nginx      # Stop Nginx service
sudo systemctl restart nginx   # Restart Nginx service
sudo systemctl reload nginx    # Reload Nginx configuration
```

---

## Step 2: Create Application Root Directory
```bash
sudo mkdir -p /var/www/masud.xyz  # Create document root directory
```

---

## Step 3: Create Sample index.html Page

**Create HTML file:**
```bash
vim /var/www/masud.xyz/index.html
```

**Sample HTML content:**
```html
<html>
    <head>
        <title>Welcome to XYZ Server!</title>
    </head>
    <body>
        <h1>Success! The masud.xyz server block is working!</h1>
    </body>
</html>
```

---

## Step 4: Create Server Block Configuration

**Create server block config file:**
```bash
sudo vim /etc/nginx/sites-available/masud.xyz
```

**Server block configuration:**
```nginx
server {
    listen 80;
    listen [::]:80;
    
    root /var/www/masud.xyz;
    index index.html index.htm index.php;
    server_name masud.xyz www.masud.xyz;
    access_log /var/log/nginx/masud.xyz.access.log;
    error_log /var/log/nginx/masud.xyz.error.log;
    
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
    
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    
    # Client Side Caching Configuration
    location ~* \.(js|jpg|jpeg|gif|png|css|tgz|gz|rar|bz2|doc|pdf|ppt|tar|wav|bmp|rtf|swf|ico|flv|txt|woff|woff2|svg)$ {
        expires 90d;
        add_header Pragma public;
        add_header Cache-Control public;
        add_header Vary Accept-Encoding;
    }
    
    location @/ {
        rewrite ^/(.+)$ /index.php?_route_=$1 last;
    }
    
    # Block .env files
    location ~ /\.env {
        deny all;
    }
    
    # Block hidden files
    location ~ /\. {
        deny all;
    }
}
```

---

## Step 5: Enable Server Block

**Create symbolic link:**
```bash
sudo ln -s /etc/nginx/sites-available/masud.xyz /etc/nginx/sites-enabled/  # Enable site
```

**Disable server block:**
```bash
sudo rm /etc/nginx/sites-enabled/masud.xyz  # Disable site
```

---

## Step 6: Test and Restart Nginx

**Test configuration and restart:**
```bash
sudo nginx -t                  # Test configuration for syntax errors
sudo nginx -s reload           # Reload Nginx configuration
sudo systemctl restart nginx   # Restart Nginx service
```

---

## Additional Nginx Commands

**Nginx control:**
```bash
sudo nginx -s stop             # Stop Nginx immediately
sudo nginx -s quit             # Graceful shutdown
sudo nginx -s reopen           # Reopen log files
```

**Nginx configuration files:**
```bash
/etc/nginx/nginx.conf          # Main Nginx configuration
/etc/nginx/sites-available/    # Available site configurations
/etc/nginx/sites-enabled/      # Enabled site configurations
/var/log/nginx/                # Nginx log files
```

---

## Step 7: Additional Configuration

- Add A & CNAME records in DNS for your domain
- Configure firewall to allow HTTP/HTTPS traffic
- Apply additional Nginx security and performance configurations

---
