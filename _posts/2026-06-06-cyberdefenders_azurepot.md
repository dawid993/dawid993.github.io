---
layout: page
title: AzurePot Lab
permalink: /cyberdefenders_azurepot/
---

Today I will start working on the AzurePot Lab from CyberDefenders. Before solving the first question, let's take a look at the provided artifacts.

## Artifacts

The challenge provides three artifacts:

- `sdb.vhd.gz` — a Virtual Hard Disk (VHD) of the main drive, obtained from an Azure disk snapshot.
- `ubuntu.20211208.mem.gz` — a memory dump created using LiME (Linux Memory Extractor).
- `uac.tgz` — an archive containing artifacts collected by UAC (Unix-like Artifacts Collector).

The first set of questions concerns the disk image (`sdb.vhd`), so our first task is to mount it and inspect its contents.

## Mounting the VHD

My initial approach was to use `guestmount`, a tool from the `libguestfs` suite that allows mounting virtual machine disk images without directly attaching them to the host system. While this is a powerful and forensic-friendly solution, it required additional setup and dependencies.

After some quick research, I found that Linux loop devices provide a much simpler approach for this image.

### What is a loop device?

A loop device is a pseudo-device that allows Linux to treat a regular file as a block device. In practice, this means that disk image files such as `.img`, `.iso`, and `.vhd` can be attached and accessed as if they were physical disks.

Once attached, standard disk utilities such as `fdisk`, `lsblk`, and `mount` can be used against the image.

### Creating the loop device

First, decompress the image:

```bash
gunzip sdb.vhd.gz
```

Then create a loop device:

```bash
sudo losetup -Pf sdb.vhd
```

Options used:

- `-f` — automatically finds the first available loop device.
- `-P` — scans the image for partitions and creates partition mappings.

Verify which loop device was assigned:

```bash
losetup
```

or

```bash
lsblk
```

Example output:

```text
loop0
├─loop0p1
└─loop0p2
```

### Mounting the partition

After identifying the partition of interest, mount it:

```bash
sudo mkdir -p /mnt/vhd
sudo mount -o ro /dev/loop0p1 /mnt/vhd
```

The `-o` flag allows additional mount options to be specified. In this case, `ro` mounts the filesystem as read-only, preventing accidental modifications to the evidence.

Once mounted, the filesystem becomes accessible under `/mnt/vhd`.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/mount.webp' | relative_url }}"
    alt="Mounted VHD using Linux loop devices"
    width="900">

  <figcaption>
    Mounting the VHD using Linux loop devices
  </figcaption>
</figure>

> **Forensic Note:** When working with disk images, it is generally recommended to mount them read-only. This helps preserve the integrity of the evidence and prevents accidental modifications during analysis.

## Alternative Approach: guestmount

Another option is to use `guestmount`, which is part of the `libguestfs` toolkit.

Unlike loop devices, `guestmount` does not expose the image through the kernel's block device subsystem. Instead, it uses a userspace appliance to inspect and mount virtual disks. This approach is often preferred in forensic environments because it provides better isolation and supports a wide range of virtual disk formats.

### Installation

```bash
sudo apt install libguestfs-tools
```

### Enumerating filesystems

Before mounting the image, identify available filesystems:

```bash
virt-filesystems -a sdb.vhd --all --long
```

Example output:

```text
Name       Type        VFS
/dev/sda1  filesystem  ext4
/dev/sda2  filesystem  swap
```

### Mounting the image

```bash
mkdir /mnt/vhd

guestmount \
  -a sdb.vhd \
  -m /dev/sda1 \
  --ro \
  /mnt/vhd
```

The `--ro` option mounts the image in read-only mode, which is strongly recommended during forensic analysis.

### Unmounting

```bash
guestunmount /mnt/vhd
```

## Loop Devices vs Guestmount

Both approaches allow access to files stored inside virtual disk images, but they work differently.

### Loop Devices

**Advantages**

- Built into the Linux kernel.
- Fast and easy to use.
- Minimal setup required.
- Works with standard Linux tools.

**Disadvantages**

- Less isolated from the host system.
- Easier to accidentally mount read-write.
- Limited to formats supported by the kernel.

### Guestmount

**Advantages**

- Better isolation from the host system.
- Supports many virtual disk formats.
- Well suited for forensic workflows.
- Read-only access is straightforward.

**Disadvantages**

- Requires additional dependencies.
- More complex setup.
- Slightly slower than loop devices.

For this challenge, the loop device approach was sufficient and significantly quicker to set up. Since the image could be mounted without any compatibility issues, it was the most practical option for the initial analysis.

With the disk mounted, we can now start examining the filesystem and answering the first set of challenge questions.

## Question 1

**There is a script that runs every minute to do cleanup. What is the name of the file?**

Since the question mentions a script running every minute, my first thought was **cron jobs**. On Linux systems, recurring scheduled tasks are commonly managed by the `cron` daemon.

Cron configurations can be found in several locations, including:

```text
/etc/crontab
/etc/cron.d/
/var/spool/cron/
/var/spool/cron/crontabs/
```

While examining the mounted disk image, I navigated to the cron spool directory:

```bash
cd /mnt/vhd/var/spool/cron
ls
```

This revealed a `crontabs` directory containing user-specific cron configurations.

```bash
sudo ls crontabs/
```

Output:

```text
root
```

Since only the root user had a crontab configured, I inspected its contents:

```bash
sudo cat crontabs/root
```

Near the end of the file, I found the following entry:

```text
* * * * * /root/[REDACTED]
```

The five asterisks indicate that the command executes every minute.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/crontab.webp' | relative_url }}"
    alt="Crontab for root"
    width="900">

  <figcaption>
    Crontab for root
  </figcaption>
</figure>

## Question 2

**The script in Q1 terminates processes associated with two Bitcoin miner malware files. What is the name of the first malware file?**

After identifying the cron job, the next step was to inspect the script being executed every minute.

```bash
sudo cat /mnt/vhd/root/[REDACTED]
```

Contents:

```bash
#!/bin/bash

for PID in `ps -ef | egrep "[REDACTED]" | grep "/tmp" | awk '{ print $2 }'`
do
    kill -9 $PID
done

chown root.root /tmp/k*
chmod [REDACTED] /tmp/k*
```

The script enumerates running processes and searches for two specific process names before terminating them with `kill -9`.

The interesting part is the `egrep` expression, which contains the names of both suspected cryptomining malware processes.

```bash
ps -ef | egrep "[REDACTED]"
```

Since the question asks for the name of the first malware file, we can extract the answer directly from this expression.

## Question 3 

**The script changes the permissions for some files. What is their new permission?**

Solution is in the script file I presented in previous question

## Question 4

**What is the SHA256 of the botnet agent file?**

At first, I focused on the `/tmp` directory because the cleanup script from previous questions was targeting files located there. However, none of the files I examined appeared to be the botnet agent referenced by the challenge.

Since the lab revolves around a compromised Apache server, I shifted my attention to the web server logs. Attackers often download and execute payloads directly from the command line using tools such as `wget` or `curl`, leaving traces in shell history, process listings, or web server logs.

I navigated to the Apache logs:

```bash
cd /mnt/vhd/var/log/apache2
```

and started looking for suspicious commands:

```bash
grep -RiE "wget|curl" .
```

This quickly revealed several interesting requests and command executions.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs1.webp' | relative_url }}"
    alt="Apache logs reveal some insight"
    width="900">

  <figcaption>
    Apache logs reveal suspicious download activity
  </figcaption>
</figure>

While reviewing the commands executed by the attacker, I noticed references to `chmod +x`, which is commonly used after downloading a binary to make it executable.

To narrow down the investigation, I searched specifically for those commands:

```bash
grep -Ri "chmod +x" .
```

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs2.webp' | relative_url }}"
    alt="Searching for chmod executions"
    width="900">

  <figcaption>
    Looking for executable payloads
  </figcaption>
</figure>

This led me to the location of the downloaded binary.

<figure>
  <img
    src="{{ '/assets/images/posts/azurePot/apachelogs3.webp' | relative_url }}"
    alt="Location of the botnet agent"
    width="900">

  <figcaption>
    Identifying the botnet agent location
  </figcaption>
</figure>

After locating the file, I calculated its SHA256 hash:

```bash
sha256sum dk86
```


