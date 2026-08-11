<img width="895" height="889" alt="image" src="https://github.com/user-attachments/assets/09916755-e2b8-42a5-8790-95b576f20ddc" />



# Linux Deleted File Recovery

> **Interview Scenario:** You accidentally delete an important file from
> a Linux server. How would you recover it?

This guide explains the practical recovery workflow, starting with the
safest and most useful check: **whether a running process still has the
deleted file open**.

------------------------------------------------------------------------

## ⚠️ Important Disclaimer

This document is for **education, interview preparation, and authorized
troubleshooting**.

File recovery is not guaranteed. The result depends on the filesystem,
storage type, permissions, backups/snapshots, and whether the blocks
containing the deleted data have been overwritten.

If this happens on a production server:

-   Avoid unnecessary writes to the affected filesystem.
-   Do not experiment with recovery commands blindly.
-   Prefer restoring from a known-good backup or snapshot when
    available.
-   Follow your organization's incident-response and change-management
    procedures.

------------------------------------------------------------------------

# 1. What Happens When You Delete a File?

On Linux, deleting a file normally removes its **directory entry** and
decrements the file's link count.

The actual data blocks may remain available until they are reused or
overwritten.

This leads to an important distinction:

``` text
File deleted
     │
     ├── Process still has file open?
     │       │
     │       └── YES → Recovery may be straightforward
     │
     └── No process has it open
             │
             ├── Backup / Snapshot available → Restore
             │
             └── No backup
                     │
                     └── Consider filesystem/file-carving recovery
```

------------------------------------------------------------------------

# 2. FIRST: Check Whether the File Is Still Open

This should usually be your first investigation step.

A process can continue using a file even after its directory entry has
been deleted.

## Using `lsof`

Run:

``` bash
sudo lsof +L1
```

You can also filter the output:

``` bash
sudo lsof +L1 | grep deleted
```

Example:

``` text
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NLINK NAME
app      1234 user    5w   REG    8,1  1048576    0  /var/log/app.log (deleted)
```

Important fields:

-   **PID** → Process ID
-   **FD** → File descriptor
-   **NAME** → Deleted file
-   **FD 5w** → File descriptor 5 is open for writing

------------------------------------------------------------------------

# 3. Recover the File Through `/proc`

If the process still has the deleted file open, Linux exposes the
process's file descriptors through:

``` text
/proc/<PID>/fd/<FD>
```

For the example above:

``` text
PID = 1234
FD  = 5
```

You can inspect the descriptor:

``` bash
sudo ls -l /proc/1234/fd/5
```

You may see something similar to:

``` text
/proc/1234/fd/5 -> /var/log/app.log (deleted)
```

Now copy the contents to a safe location:

``` bash
sudo cp /proc/1234/fd/5 /tmp/recovered_app.log
```

Verify the recovered file:

``` bash
ls -lh /tmp/recovered_app.log
file /tmp/recovered_app.log
```

For a text file:

``` bash
head /tmp/recovered_app.log
```

### Why does this work?

The process still owns an open file descriptor that references the
underlying file object.

Removing the filename from the directory does not immediately invalidate
an already-open file descriptor.

Conceptually:

``` text
/var/log/app.log
       │
       X  ← directory entry deleted
       │
       ▼
   File still open
       │
       ▼
/proc/1234/fd/5
       │
       ▼
Underlying file data
```

------------------------------------------------------------------------

# 4. Important: Act Before the Process Closes the File

The `/proc/<PID>/fd/<FD>` approach works only while the relevant file
descriptor remains open.

If the application:

-   restarts,
-   closes the file,
-   crashes,
-   or the process is terminated,

the descriptor may disappear.

Therefore, if the file is still open:

**Recover/copy it before restarting or stopping the process unless your
incident procedure requires otherwise.**

------------------------------------------------------------------------

# 5. What If No Process Has the File Open?

If:

``` bash
sudo lsof +L1
```

does not show the deleted file, the next options are:

## Option 1 --- Restore From Backup

This is normally the preferred production recovery method.

Examples:

``` text
Backup server
Cloud backup
Database/application backup
Object storage backup
Filesystem backup
```

Restore from the latest known-good copy and verify the result.

------------------------------------------------------------------------

# 6. Restore From a Snapshot

If your infrastructure uses snapshots, check whether a recent snapshot
contains the file.

Common examples include:

-   AWS EBS snapshots
-   GCP Persistent Disk snapshots
-   Azure managed disk snapshots
-   LVM snapshots
-   Storage-array snapshots
-   VM snapshots

A snapshot can often be safer than attempting low-level undelete
operations on the original production disk.

------------------------------------------------------------------------

# 7. Check Version Control or Application History

If the deleted file was configuration, code, or another versioned
artifact, check its history.

For Git:

``` bash
git log --all -- path/to/file
```

You can inspect a previous version with:

``` bash
git show <commit>:path/to/file
```

Then restore it if appropriate.

------------------------------------------------------------------------

# 8. Filesystem Recovery Tools

If there is:

-   no open file descriptor,
-   no usable backup,
-   no snapshot,
-   and the data is important,

filesystem recovery tools may be considered.

## `extundelete`

`extundelete` is designed for recovering deleted files from
**ext3/ext4** filesystems.

Example workflow:

``` bash
sudo umount /dev/sdX
sudo extundelete /dev/sdX --restore-file path/to/file
```

Or to attempt broader recovery:

``` bash
sudo extundelete /dev/sdX --restore-all
```

> **Do not blindly run these commands against a production filesystem.**
> Recovery normally requires careful handling of the underlying block
> device, and the filesystem should generally be unmounted to reduce
> further changes.

------------------------------------------------------------------------

# 9. TestDisk

[TestDisk](https://www.cgsecurity.org/) is an open-source recovery
utility that can help with lost partitions and filesystem recovery.

Typical workflow:

``` text
sudo testdisk
        │
        ├── Select disk
        ├── Select partition
        ├── Analyse / Advanced
        ├── Select the appropriate recovery option
        └── Copy recovered data to another storage location
```

Always recover data to a **different storage location** when possible.

------------------------------------------------------------------------

# 10. PhotoRec

PhotoRec is another recovery tool from the TestDisk project.

Unlike normal filesystem-aware recovery, PhotoRec focuses on **file
carving** based on known file signatures.

Example:

``` bash
sudo photorec /dev/sdX
```

It can be useful when filesystem metadata is no longer sufficient.

However, file carving may:

-   lose original filenames,
-   lose directory structure,
-   recover incomplete files,
-   produce many unrelated files.

------------------------------------------------------------------------

# 11. Scalpel

Scalpel is a file-carving utility that can search storage for files
based on configured file signatures.

Example:

``` bash
sudo scalpel /dev/sdX -o /recovery
```

The exact configuration depends on the file types you are trying to
recover.

------------------------------------------------------------------------

# 12. The Most Important Rule: Avoid Writes

After accidental deletion, **writing new data to the affected filesystem
can overwrite blocks belonging to the deleted file.**

Avoid unnecessary operations such as:

``` text
Large downloads
Package installations
Large log generation
Copying files onto the same filesystem
Creating temporary files on the affected disk
```

When possible:

``` text
Affected Disk
     │
     ├── STOP unnecessary writes
     │
     └── Recover to a DIFFERENT disk
```

------------------------------------------------------------------------

# 13. Recommended Interview Answer

If an interviewer asks:

> **"You accidentally deleted a production file on Linux. How would you
> recover it?"**

A strong answer is:

> "First, I would avoid unnecessary writes to the affected filesystem. I
> would check whether the deleted file is still held open by a running
> process using `lsof +L1`. If it is open, I would identify the PID and
> file descriptor and copy the file through `/proc/<PID>/fd/<FD>` to a
> safe location. If no process has it open, I would check backups,
> snapshots, and application/version history. Only if those options are
> unavailable would I consider filesystem recovery or file-carving tools
> such as `extundelete`, TestDisk, PhotoRec, or Scalpel, preferably
> against an appropriate disk image or unmounted device."

This demonstrates understanding of:

-   Linux processes
-   File descriptors
-   `/proc`
-   Filesystems
-   Backups
-   Snapshots
-   Production safety
-   Disaster recovery

------------------------------------------------------------------------

# 14. Quick Decision Tree

``` text
             File accidentally deleted
                       │
                       ▼
            Stop unnecessary writes
                       │
                       ▼
              Check `lsof +L1`
                       │
             ┌─────────┴─────────┐
             │                   │
            YES                  NO
             │                   │
             ▼                   ▼
       File still open?      Check backup
             │                   │
             ▼                   ▼
     Find PID + FD           Backup exists?
             │                   │
             ▼             ┌─────┴─────┐
  Copy `/proc/PID/fd/FD`   │           │
             │            YES          NO
             ▼             │           │
        Verify file        ▼           ▼
                       Restore      Check snapshot
                                       │
                                       ▼
                                  Check history
                                       │
                                       ▼
                              Recovery tools
```

------------------------------------------------------------------------

# 15. Useful Commands Cheat Sheet

### Find deleted-but-open files

``` bash
sudo lsof +L1
```

### Filter deleted entries

``` bash
sudo lsof +L1 | grep deleted
```

### Inspect a process file descriptor

``` bash
sudo ls -l /proc/<PID>/fd/<FD>
```

### Recover an open deleted file

``` bash
sudo cp /proc/<PID>/fd/<FD> /safe/location/recovered_file
```

### Verify recovered file

``` bash
file /safe/location/recovered_file
ls -lh /safe/location/recovered_file
```

### Check Git history

``` bash
git log --all -- path/to/file
```

### Recover a file from a Git commit

``` bash
git show <commit>:path/to/file
```

------------------------------------------------------------------------

# 16. Key Takeaways

1.  **Do not panic.**
2.  **Avoid unnecessary writes to the affected filesystem.**
3.  Check for deleted-but-open files with:

``` bash
sudo lsof +L1
```

4.  If the file is still open, use:

``` bash
/proc/<PID>/fd/<FD>
```

5.  Prefer **backups and snapshots** for production recovery.
6.  Recover to **different storage** whenever possible.
7.  If no backup exists, consider filesystem recovery/file carving
    carefully.
8.  Recovery is **not guaranteed** once the underlying data has been
    overwritten.

------------------------------------------------------------------------

## 🎯 Interview Takeaway

**The first question to ask is not "Which recovery tool should I use?"**

It is:

> **"Is the deleted file still open by a running process?"**

That single check can turn a potentially difficult recovery into a
simple file copy.

------------------------------------------------------------------------

## 📚 Topics Covered

`Linux` · `Linux Administration` · `SRE` · `DevOps` · `File Descriptors`
· `/proc` · `Filesystem Recovery` · `Disaster Recovery` ·
`Production Troubleshooting` · `System Administration`
