# NIT Academy — Kickstart Lab 03: HTTP Installation Repository and `ks.cfg`

**Platform:** RHEL 8.1 • **Prerequisites:** Labs 01–02 completed.

> **Lab conventions:** Single all-in-one RHEL **8.1** server `192.168.209.137/24`; isolated network `192.168.209.0/24`; DHCP clients `.140`–`.150`; BIOS/legacy PXE boot using `pxelinux.0`. **Do not use `.139` from the earlier transcript.** Commands assume root (`sudo -i`).
>
> **Safety:** Keep the lab isolated from production DHCP and validate disk targets before installation; Kickstart partitioning can erase a disk. The server must have a **static** IP.


## 1. Lab objective and services

Complete the single-server build by publishing the **full RHEL 8.1 installation media** over HTTP and a Kickstart file. Lab 02 already supplies this kernel command line:

```text
inst.repo=http://192.168.209.137/content/rhel8.1/x86_64/dvd/
inst.ks=http://192.168.209.137/content/rhel8.1/x86_64/dvd/ks.cfg
```

## 2. Install and start Apache HTTP Server

```bash
sudo -i
dnf install -y httpd
systemctl enable --now httpd
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

## 3. Copy the RHEL DVD installation tree

```bash
mount /dev/sr0 /mnt  # omit if already mounted
mkdir -p /var/www/html/content/rhel8.1/x86_64/dvd
cp -a /mnt/. /var/www/html/content/rhel8.1/x86_64/dvd/
restorecon -RFv /var/www/html/content
```

Using `/mnt/.` includes hidden files as well. The source should be the full RHEL 8.1 DVD, including `.treeinfo`, `BaseOS/`, and `AppStream/`.

Verify:

```bash
ls -la /var/www/html/content/rhel8.1/x86_64/dvd/
curl -I http://192.168.209.137/content/rhel8.1/x86_64/dvd/.treeinfo
curl -I http://192.168.209.137/content/rhel8.1/x86_64/dvd/BaseOS/repodata/repomd.xml
```

These URLs should return `200 OK` for an accessible complete media tree. A directory URL may be forbidden without an index; test **files**, not directory listings.

## 4. Prepare the Kickstart file

The RHEL installer often saves its installation answers as `/root/anaconda-ks.cfg`. Review it before reusing: it may contain a root password hash, machine-specific storage names, or other host-specific information.

```bash
ls -l /root/anaconda-ks.cfg
cp /root/anaconda-ks.cfg /var/www/html/content/rhel8.1/x86_64/dvd/ks.cfg
chmod 644 /var/www/html/content/rhel8.1/x86_64/dvd/ks.cfg
```

**Recommended teaching example:** replace `/var/www/html/content/rhel8.1/x86_64/dvd/ks.cfg` with the following **adapted** template. It intentionally keeps the key values and packages from your draft but avoids hard-coding an unverified `nvme0n1` disk and a weak plaintext `redhat` password. **Never boot it until you verify disk selection and partitioning.**

```kickstart
#version=RHEL8
# RHEL 8.1 unattended installation - isolated training lab
text
reboot

url --url="http://192.168.209.137/content/rhel8.1/x86_64/dvd/"
repo --name="appstream" --baseurl="http://192.168.209.137/content/rhel8.1/x86_64/dvd/AppStream/"

keyboard --vckeymap=us --xlayouts='us'
lang en_US.UTF-8
timezone America/New_York --utc
network --bootproto=dhcp --device=link --activate

# Password generated with e.g. openssl passwd -6; replace before use:
rootpw --lock
# Instead, supply a secure hashed password or configure an admin user.

selinux --enforcing
firstboot --disable
skipx
services --disabled="kdump,rhsmcertd" --enabled="sshd,chronyd"

# WARNING: autopart selects storage. On machines with multiple disks,
# explicitly specify and validate disk(s) before using destructive options.
autopart --type=lvm

%packages
@core
chrony
cloud-init
dracut-config-generic
dracut-norescue
firewalld
grub2
kernel
rsync
tar
-plymouth
%end

%post --erroronfail
# Place safe site-specific post-install steps here.
%end
```

**Security and boot notes:**

- `rootpw --lock` prevents root password login; make sure the deployment includes a usable administrator account or configure a **secure hashed password** before an unattended deployment, or you may lock yourself out.
- For a simple lab with a known disposable VM, you may instead use `rootpw --iscrypted <SHA-512-hash>`; **do not** distribute a common plaintext `redhat` root password.
- The lecture uses both `graphical` and `skipx` (which conflicts with a graphical desktop goal). The lab uses `text` plus `skipx` for a minimal server installation.
- The instructor's `ignoredisk --only-use=nvme0n1` is valid only when that disk **actually exists**. VM disks may be `/dev/vda`, `/dev/sda`, or NVMe. The original `clearpart --all --initlabel` and `zerombr` are destructive; this template omits them pending target-disk confirmation. For reliable automation, choose a deliberate disk-specific partitioning policy.
- Some package names may be unavailable from your exact media; validate `%packages` and remove unneeded entries if the installation reports dependency errors.
- RHEL subscription/entitlement handling and package sources depend on your environment.

> **Instructor readiness check:** Before using this template with a student VM, choose its exact target disk and administrative access method. A copied `anaconda-ks.cfg` can be more reliable once sanitized to your actual VM hardware.

## 5. Publish and verify `ks.cfg`

```bash
restorecon -v /var/www/html/content/rhel8.1/x86_64/dvd/ks.cfg
curl -f http://192.168.209.137/content/rhel8.1/x86_64/dvd/ks.cfg | head -n 20
```

Optionally install Kickstart validation tools if available:

```bash
dnf install -y pykickstart
ksvalidator /var/www/html/content/rhel8.1/x86_64/dvd/ks.cfg
```

`ksvalidator` validates syntax, **not** target disk suitability, HTTP reachability, package availability, or full installation correctness.

## 6. Configure HTTP YUM repositories

Create `/etc/yum.repos.d/server.repo`:

```ini
[RHEL8.1RepoBaseOSPackages_http]
name=RHEL8Yum
baseurl=http://192.168.209.137/content/rhel8.1/x86_64/dvd/BaseOS
gpgcheck=0
enabled=1

[RHEL8.1AppPackages_http]
name=RHEL8AppPakages
baseurl=http://192.168.209.137/content/rhel8.1/x86_64/dvd/AppStream
gpgcheck=0
enabled=1
```

For a training server with both file-based repositories (Lab 01) and HTTP repositories, keep track of which are enabled; duplicate repository **content** is not necessary even when IDs differ. To test only HTTP repositories:

```bash
dnf clean all
dnf --disablerepo='*' \
  --enablerepo=RHEL8.1RepoBaseOSPackages_http \
  --enablerepo=RHEL8.1AppPackages_http repolist
```

> In production, enable GPG validation and use centrally managed/signed repositories.

## 7. End-to-end test

1. Confirm `dhcpd`, `tftp.socket`, and `httpd` are active.
2. Verify PXE client firmware is set to **legacy BIOS** for `pxelinux.0`.
3. Place client and server on the same isolated network.
4. Give the student VM a **disposable disk** and verify its intended installation disk.
5. Ensure `ks.cfg` has a working administrator authentication method and appropriate partitioning.
6. PXE-boot the VM; follow Anaconda messages on the console.
7. Confirm the installer downloads `ks.cfg`, accesses `BaseOS` and `AppStream`, and completes.

```bash
systemctl is-active dhcpd tftp.socket httpd
curl -fI http://192.168.209.137/content/rhel8.1/x86_64/dvd/ks.cfg
journalctl -u dhcpd -n 20 --no-pager
journalctl -u httpd -n 20 --no-pager
```

## 8. Common problems

| Symptom | Verification |
|---|---|
| Installer cannot find source | `inst.repo` path points to `.../dvd/`; media tree includes `.treeinfo` and packages |
| Kickstart file 404 | Confirm `ks.cfg` is in document root path and URL matches `.137` |
| Permission denied serving file | Check `restorecon`, file permissions, httpd and firewall |
| Root login unavailable | `rootpw --lock` needs a deliberately configured alternative administrator credential |
| Partitioning error | Inspect target VM disk type/size; update storage directives to match |
| Package errors | Confirm package availability and accessible BaseOS/AppStream metadata |
| BIOS menu works but UEFI fails | BIOS `pxelinux.0` is not the required UEFI loader |

## 9. Completion checklist

- [ ] HTTP server running and reachable
- [ ] Installation tree available at the exact `.137` URL
- [ ] `ks.cfg` published and syntax-checked
- [ ] Boot menu references valid `inst.repo` and `inst.ks`
- [ ] Kickstart authentication and disk policy adapted safely for target VM
- [ ] PXE client completes an automated installation

**Next labs:** optional UEFI PXE with GRUB EFI, secure production repository settings, and Ansible post-provisioning.
