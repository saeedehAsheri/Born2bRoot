# Born2bRoot
This documentation  helps you understand the concepts, the purpose of the project, and exactly what you need to do step‑by‑step, including what to add to the system files for sudo, SSH, UFW, password policy, and your monitoring script.

## 1. What Is the Born2beroot Project?

Born2beroot is basically your first real experience with setting up and managing a Linux server. Instead of just using Linux, you actually install it, configure it, secure it, and learn how a real server works behind the scenes.

You do it all manually without graphical interfaces—just the terminal. The purpose is to help you understand what system administrators, DevOps engineers, and security engineers work with every day.

In this project you learn:

- How to install a Linux OS (Debian)

- How users, permissions, groups, and sudo work

- How to secure a system with password rules and firewall

- How to run SSH on a custom port

- How to manage services on Linux

- How to write a monitoring script that shows system information

This is a practical, hands‑on experience that teaches you how a real server behaves.
## 2. Why Do We Do This Project? (What is it useful for?)

This project prepares you for real jobs in areas such as:

- System administration

- DevOps

- Cloud computing (AWS, Azure, GCP)

- Cybersecurity

- Backend engineering

Modern companies use Linux servers everywhere. If you know how to install, secure, and configure one—you already have a strong foundation.

Born2beroot teaches you:

- How to think like a system administrator

- How to configure a machine safely

- How to manage users and groups properly

- How to secure remote access (SSH)

- How to set strong password policies

- How to understand logs and monitor systems

This project gives you the basics that almost every tech job builds on.

## 3. Step‑by‑Step: What You Actually Do

Below is a clear, simple walkthrough of everything you must do.

### Step 1 — Install VirtualBox

Download VirtualBox for your operating system. Create a new virtual machine and attach the Debian ISO. [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Nothing complicated here. :)

### Step 2 — Install Debian (No Graphical Interface)

Inside VirtualBox, boot the ISO and install Debian with: [Download Debian](https://www.debian.org/distrib/)


- No desktop environment

- Your chosen hostname (usually your intra login)

- A root password

- A regular user (your login)

- Partitioning with LVM (and encryption if your subject requires)

After installation, you reboot into your fresh server.

### Step 3 — Install and Configure Sudo

By default, Debian does NOT give normal users sudo permissions. So we:

#### 3.1 Install sudo and add your user to the sudo group

Log in as root:
```
su -
apt install sudo -y
usermod -aG sudo your_login
exit
```
Log in again with your normal user.

#### 3.2 Configure sudo behavior

You edit the sudo configuration safely using:
```
sudo visudo
```
Add or make sure you have:
```
# basic permissions
root ALL=(ALL:ALL) ALL
your_login ALL=(ALL:ALL) ALL


# log all sudo commands
Defaults logfile="/var/log/sudo/sudo.log"


# allow maximum 3 wrong password tries
Defaults passwd_tries=3


# ask for password every time (more secure)
Defaults timestamp_timeout=0


# require use of terminal
Defaults requiretty
```
Then create the log folder:
```
sudo mkdir -p /var/log/sudo
sudo touch /var/log/sudo/sudo.log
sudo chmod 640 /var/log/sudo/sudo.log
```
### Step 4 — Set Password Policy

A secure server must force strong passwords. Debian uses two places to enforce rules.

#### 4.1 Edit ```/etc/login.defs``` to control expiration
```
sudo nano /etc/login.defs
```
Set:
```
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7
```
Apply to your user:
```
sudo chage -M 30 -m 2 -W 7 your_login
```
#### 4.2 Edit /etc/pam.d/common-password for complexity rules

Install the module:
```
sudo apt install libpam-pwquality -y
```
Edit file:
```
sudo nano /etc/pam.d/common-password
```
Add or modify this line:
```
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```
This enforces:

- Minimum 10 characters

- At least one uppercase, lowercase, number, special character

- Rejects passwords similar to username

- Works for root too

### Step 5 — Configure UFW Firewall

Install UFW:
```
sudo apt install ufw -y
```
Set default rules:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Allow SSH port (Born2beroot uses port 4242):
```
sudo ufw allow 4242/tcp
```
Enable firewall:
```
sudo ufw enable
```
Check status:
```
sudo ufw status
```
### Step 6 — Configure SSH

Install SSH server:
```
sudo apt install openssh-server -y
```
Edit the configuration:
```
sudo nano /etc/ssh/sshd_config
```
Change or add:
```
Port 4242
PermitRootLogin no
PasswordAuthentication yes
UsePAM yes
AllowUsers your_login
```
Restart the service:
```
sudo systemctl restart ssh
```
Now your server can be accessed remotely (from the host machine) using:
```
ssh -p 4242 your_login@your_vm_ip
```
### Step 7 — Users and Groups

Depending on your subject, you may need to create extra users or groups. Examples:
```
sudo groupadd evaluating
sudo useradd -m testuser -s /bin/bash
sudo passwd testuser
sudo usermod -aG evaluating testuser
```
To see users:
```
getent passwd
```
To see groups:
```
getent group
```
To remove a user:
```
sudo deluser testuser
```
To remove a group:
```
sudo groupdel evaluating
```
### Step 8 — Monitoring Script

Your monitoring script prints important information about the system every time it runs. Create file:
```
sudo nano /usr/local/bin/monitoring.sh
```
Paste your script:
```bash
#!/bin/bash


# ARCH
arch=$(uname -a)


# CPU PHYSICAL
cpuf=$(grep "physical id" /proc/cpuinfo | wc -l)


# CPU VIRTUAL
cpuv=$(grep "processor" /proc/cpuinfo | wc -l)


# RAM
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_use=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_percent=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')


# DISK
disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf ("%.1fGb
"), disk_t/1024}')
disk_use=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} END {print disk_u}')
disk_percent=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} {disk_t+= $2} END {printf("%d"), disk_u/disk_t*100}')


# CPU LOAD
cpul=$(vmstat 1 2 | tail -1 | awk '{printf $15}')
cpu_op=$(expr 100 - $cpul)
cpu_fin=$(printf "%.1f" $cpu_op)


# LAST BOOT
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')


# LVM USE
lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)


# TCP CONNEXIONS
tcpc=$(ss -ta | grep ESTAB | wc -l)


# USER LOG
ulog=$(users | wc -w)


# NETWORK
ip=$(hostname -I)
mac=$(ip link | grep "link/ether" | awk '{print $2}')


# SUDO
cmnd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)


wall " Architecture: $arch
CPU physical: $cpuf
vCPU: $cpuv
Memory Usage: $ram_use/${ram_total}MB ($ram_percent%)
Disk Usage: $disk_use/${disk_total} ($disk_percent%)
CPU load: $cpu_fin%
Last boot: $lb
LVM use: $lvmu
Connections TCP: $tcpc ESTABLISHED
User log: $ulog
Network: IP $ip ($mac)
Sudo: $cmnd cmd"
```
Make the script executable:
```
sudo chmod +x /usr/local/bin/monitoring.sh
```
Run it manually to check:
```
sudo /usr/local/bin/monitoring.sh
```
### Step 9 — Set Up Cron to Run Script Every 10 Minutes
Open cron:
```
sudo crontab -e
```
Add this:
```
*/10 * * * * /usr/local/bin/monitoring.sh
```
Now your monitoring script runs automatically. :)

## You can find some useful definations in this part
### What is Virtual machine?
A Virtual Machine (VM) is a computer that runs inside another computer.

Think of it like this:
You open a window on your computer, and inside that window, another operating system (like Debian) is running — completely separate from your main system.

A VM has:

- Virtual CPU

- Virtual RAM

- Virtual hard disk

- Virtual network

- Its own operating system

You can install, delete, restart, or clone a VM without affecting your real machine.
### What is the diffirence between apt and aptitude?
Both apt and aptitude are package managers in Debian.
They help you install, remove, and update software.

#### APT

- Modern package manager
- Default and most recommended tool
- Simple and fast
- Always available on Debian

#### Aptitude

- Older package manager
- More advanced dependency resolver
- Has a text-based UI
- Not installed by default
### What is AppArmor?
AppArmor is a Linux security system that protects your computer by controlling what applications are allowed to do.
It uses profiles, which are like “permission rules” for each program.
AppArmor can limit:
- What files a program can read
- What folders it can write to
- Whether it can use the network
- What system resources it can access
- If a program gets attacked or behaves strangely, AppArmor prevents it from damaging the system.
Debian uses AppArmor by default because it is simple, lightweight, and effective.
### What is LVM?
LVM (Logical Volume Manager) is a powerful and flexible way of managing disk storage in Linux.
Instead of using fixed partitions, LVM gives you dynamic volumes that you can resize easily.
LVM has 3 layers:

1. Physical Volume (PV)
These are actual disks or disk partitions, like /dev/sda2.

2. Volume Group (VG)
A pool of storage made from one or more physical volumes.

3. Logical Volume (LV)
These are like partitions created from the pool of storage.
You can resize them anytime.

**Advantages of LVM:**

- You can resize partitions without restarting.

- You can combine multiple disks into one big storage space.

- You can create snapshots.

- It is more flexible than normal partitioning.
### What is UFW?
UFW (Uncomplicated Firewall) is a simple firewall tool for Linux systems.
A firewall controls which connections are allowed to enter or leave your system.

**What does UFW do?**
- Protects your server
- Blocks unwanted access
- Allows only the ports you choose


