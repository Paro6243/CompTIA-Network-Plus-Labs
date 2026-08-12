# NAS vs SAN Lab (Samba + iSCSI on Parrot OS)

## Goal
Build both a NAS-style share (Samba/SMB) and a SAN-style block device (iSCSI target + initiator) on the same Parrot OS box, then capture and compare their traffic in Wireshark to see the real difference between **file-level** and **block-level** storage access.

---

## Part 1: NAS (Samba/SMB)

### Setup
- Installed Samba (`apt install samba`), enabled + started `smbd`
- Created a shared folder `~/nas-share` and defined it in `/etc/samba/smb.conf`:
  ```ini
  [nas-share]
     path = /home/paro/nas-share
     browseable = yes
     writable = yes
     guest ok = no
     valid users = paro
  ```
- Validated config with `testparm`, set a Samba password with `smbpasswd -a paro`
- Connected locally with `smbclient`, then mounted properly:
  ```bash
  sudo mount -t cifs //localhost/nas-share /mnt/nas-test -o username=paro
  ```

### Permission model — two gates, both must pass
Tested what happens when Samba share config and Linux filesystem permissions disagree:

- Created `readonly-test.txt` with `chmod 444` (read-only for everyone at the OS level)
- Share itself was still fully writable (`read only = No`)
- Result: **write blocked** — `Permission denied` — even though the share config allowed it

**Takeaway:** Samba never overrides the OS. A share being "writable" only grants permission at the *share* level — the actual Linux file permissions on the server are checked independently on every operation. Both gates must say yes.

### Client-side mount quirk
By default, CIFS mounts showed all files as owned by `root:root` client-side, regardless of real server ownership — this blocked writes to a file that should have been writable. Fixed with:
```bash
sudo mount -t cifs //localhost/nas-share /mnt/nas-test -o username=paro,uid=$(id -u paro),gid=$(id -g paro)
```
This proved client-side `ls -l` output is just a local display/mapping layer — it is **not** the authority on what's actually allowed. The server's real permissions are enforced independently every time, regardless of what the client shows.

### Wireshark (filter: `smb2`)
Every operation referenced the file by name:
```
Create Request File: testfile.txt
Read Request Len:24 Off:0 File: testfile.txt
Close Request File: testfile.txt
```
This is file-level access — the protocol conversation is entirely in terms of named files and file handles.

---

## Part 2: SAN (iSCSI)

### Setup
- Installed `targetcli-fb` (target/server) and `open-iscsi` (initiator/client)
- Created a 500MB backing file to act as a virtual disk:
  ```bash
  sudo dd if=/dev/zero of=/iscsi-disks/disk1.img bs=1M count=500
  ```
- In `targetcli`: created a fileio backstore, an iSCSI target (auto-generated IQN), attached the backstore as LUN 0, and created an ACL restricting access to this machine's specific initiator IQN
- On the initiator side: discovered and logged into the target
  ```bash
  sudo iscsiadm -m discovery -t sendtargets -p 127.0.0.1
  sudo iscsiadm -m node -T <target-iqn> -p 127.0.0.1 --login
  ```
- `lsblk` showed a brand-new block device, `/dev/sdb`, 500M — indistinguishable from a physically attached disk
- Formatted with `mkfs.ext4`, mounted, wrote a test file

### Wireshark (filter: `iscsi`)
No filenames anywhere — only LUN + Logical Block Address (LBA):
```
SCSI Read(10) LUN: 0x00 1024 bytes (2 blocks) at LBA: 0x0000...
SCSI Write(10) LUN: 0x00 2048 bytes (4 blocks) at LBA: 0x0000...
SCSI Synchronize Cache(10) LUN: 0x00
```
The iSCSI target has zero concept of "files" — it only understands raw blocks. The ext4 filesystem (and its knowledge of `sanfile.txt`, `lost+found`, etc.) exists entirely on the **client** side. The target is a dumb block-storage service.

---

## Side-by-side comparison

| | NAS (SMB) | SAN (iSCSI) |
|---|---|---|
| Access level | File-level | Block-level |
| Protocol | SMB2 (port 445) | iSCSI (port 3260) |
| Appears to OS as | Network share | Local disk |
| Wireshark shows | `File: testfile.txt` | `LUN + LBA`, no filenames |
| Filesystem knowledge | Lives on the server | Lives on the client |

## Key takeaways
1. **File-level vs block-level is visible on the wire, not just in theory** — SMB talks in filenames, iSCSI talks in block addresses.
2. **Permission enforcement in SMB is two independent layers** (share config + real filesystem perms) — both must allow an operation, and the client's displayed permissions are not authoritative.
3. **In iSCSI, the target is "dumb"** — all filesystem intelligence sits on the initiator (client) side; the target just serves raw blocks.
