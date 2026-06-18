---
layout: page
title: Linux UAC
permalink: /linux_uac/
---

# Linux Live Response

## Current Users
who
w
last

Purpose:
- Active users
- Recent logins
- Suspicious sessions

## Running Processes
ps auxf
pstree -ap

Look for:
- Unknown binaries
- Processes in /tmp
- Deleted executables

## Network Activity
ss -antup
ip addr
ip route

Look for:
- Reverse shells
- Unexpected listening ports
- External connections

## Scheduled Tasks
crontab -l
ls -la /etc/cron*

Look for:
- Persistence
- Malware execution

## Systemd Services
systemctl list-units --type=service
systemctl list-unit-files

Look for:
- New or suspicious services
- Services running from unusual paths

## SSH Artifacts
cat ~/.ssh/authorized_keys

Look for:
- Unauthorized public keys
- Recently added access

## Command History
cat ~/.bash_history
cat ~/.zsh_history

Look for:
- wget/curl
- privilege escalation
- attacker commands

## Temporary Directories
ls -alh /tmp
ls -alh /var/tmp
ls -alh /dev/shm

Look for:
- Scripts
- Payloads
- Archives

## Open Files
lsof

Useful for:
- Deleted but running binaries
- Suspicious file access

```
# Examples

# Network connections IPV4
lsof -i 4

# Don't map host names (-n), port numbers (-P), usernames (-l)
lsof -i 4 -nlP

lsof -nlP

# A protocol name - TCP, UDP
lsof -i TCP
lsof -i UDP

# All tcp connection in LISTEN state
lsof -iTCP -sTCP:LISTEN

# All tcp connection in ESTABLISHED state
lsof -iTCP -sTCP:ESTABLISHED

# Show all network connections with PID and process name
lsof -i -nP

# Only listening ports
lsof -iTCP -sTCP:LISTEN -nP

# Only established connections
lsof -iTCP -sTCP:ESTABLISHED -nP

# Deleted but still open files (VERY IMPORTANT)
lsof +L1

# Open files for a process
lsof -p <PID>

# Executable, cwd, libraries used by process
lsof -p <PID> | egrep 'txt|cwd|mem'

# Files opened by a specific user
lsof -u root

# Files under suspicious directories
lsof +D /tmp
lsof +D /var/tmp

# Which process is using a file
lsof /path/to/file

# Which process has a directory open
lsof +D /home/user

# Network connections of a specific process
lsof -p <PID> -i

# Specific remote host
lsof -i @1.2.3.4

# Specific TCP port range
lsof -i TCP:1-1024

# IPv6 only
lsof -i 6
#

```

## Loaded Kernel Modules
lsmod

Look for:
- Unknown modules
- Rootkit indicators

## Containers
docker ps -a
docker images

Look for:
- Unauthorized containers
- Crypto miners

## Memory Related
cat /proc/<pid>/cmdline
cat /proc/<pid>/environ

Useful when:
- Malware is memory-only
- Process arguments are hidden later