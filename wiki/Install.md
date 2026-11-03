# Install

vsptop6 runs on a Linux host that has a **command device** from each VSP array mapped to it.
It talks to the arrays two ways, and both come from Hitachi:

| Prerequisite | Used by | What it is |
|---|---|---|
| **Command Control Interface (CCI / RAID Manager)** | `genarrayhash` | the `raidcom` CLI and the HORCM daemon, installed under `/HORCM` |
| **RAID Manager Library (RMLIB)** | the collectors (`allldevstat`, `allportstat`, `cachestat`, `allmpinfo`, …) | `libsvrrm.so`, the shared library the collectors load at run time |

The RPM refuses to install until both are present. Supported platform: RHEL / Rocky / Alma 9,
x86_64 (the release RPM is tagged `.el9`).

## 1. Prepare the host

Map a **command device** from every array you want to monitor (if you already run CCI on this host you probably have these):

1. On the array (Storage Navigator / Ops Center Administrator / `raidcom`), take a small LDEV
   (a few MB is enough), enable the *Command Device* attribute on it and turn on
   **user authentication** (`raidcom modify ldev -ldev_id <id> -command_device y -security 1`
   or the equivalent in the GUI). RMLIB logs in with a user name and password, so an
   unauthenticated command device is not enough.
2. Map it to the vsptop host through a host group / LUN path like any other LUN.
3. On the host, rescan and identify it:
   ```sh
   rescan-scsi-bus.sh                   # or echo "- - -" > /sys/class/scsi_host/hostN/scan
   echo /dev/sd* | /HORCM/usr/bin/inqraid -f -CLI
   ```
   You get one line per device; the command devices show the array **SERIAL** and
   `PRODUCT_ID = OPEN-V-CM`. Note the `/dev/sdX` name for each serial — you need it twice later.

If several hosts map the same command device, each host needs its own; command devices are
not shared between HORCM instances on different hosts.

## 2. Install CCI (RAID Manager)

Download *Command Control Interface* for Linux from
[Hitachi Vantara Support Connect](https://support.hitachivantara.com) (the package is named
like `RMHORC_X64` inside the CCI media; use the x64 build). Then, as root:

```sh
mkdir -p /tmp/cci && cd /tmp/cci
unzip <downloaded-cci-package>.zip            # or mount the ISO
cd /                                          # CCI installs relative to the root
cpio -idmu < /tmp/cci/LINUX/RMHORC_X64         # creates /HORCM
/HORCM/horcminstall.sh
```

Verify:

```sh
/HORCM/usr/bin/raidqry -h | head -3          # prints the CCI version
ls /HORCM/usr/bin/raidcom
```

Add `/HORCM/usr/bin` to `root`'s `PATH` (`/etc/profile.d/horcm.sh`) if you want to run
`raidcom` without the full path; vsptop6 always uses the full path.

## 3. Install the RAID Manager Library (RMLIB)

RMLIB is a **separate** download on Support Connect (*RAID Manager Library* / `RMLIB`), not
part of the CCI package. Install the 64-bit library:

```sh
mkdir -p /tmp/rmlib && cd /tmp/rmlib
unzip <downloaded-rmlib-package>.zip
# follow the readme in the package; on Linux x64 it installs
#   /usr/lib64/libsvrrm.so           (64-bit: the one the vsptop6 collectors load)
#   /usr/lib/libsvrrm.so             (32-bit; harmless but not used)
# The package also ships rmlib.h; that header is only needed to compile the
# collectors from source, not to run the RPM's binaries.
ldconfig
```

Verify — this is what the RPM's pre-install check looks for:

```sh
ls -l /usr/lib64/libsvrrm.so*
file -L /usr/lib64/libsvrrm.so       # must say "ELF 64-bit"
```

If only `/usr/lib/libsvrrm.so` exists you have the 32-bit library; install the x64 package.

## 4. Install vsptop6

From the GitHub release (the `.el9` RPM works on RHEL/Rocky/Alma 9):

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v1.0.2/vsptop6-1.0.2-1.el9.x86_64.rpm
```

`dnf` fetches `ncurses-libs` and `sqlite-libs` from the OS repositories, runs the
prerequisite check and installs:

```
/usr/local/san/vsp/bin/        vsptop, genarrayhash, the collectors
/usr/local/san/vsp/conf/       vsptop.conf (created from vsptop.conf.example on first install)
/usr/local/san/vsp/var/        topology.db (written by genarrayhash)
/usr/local/san/vsp/doc/        README, methodology
/usr/bin/vsptop, /usr/bin/genarrayhash   symlinks
/etc/cron.d/vsptop6            hourly genarrayhash
```

If the check fails you'll see one of:

```
vsptop6: Hitachi Command Control Interface (RAID Manager) was not found (/HORCM/usr/bin/raidcom is missing).
vsptop6: the RAID Manager Library (RMLIB, libsvrrm.so) was not found in /usr/lib64 or /usr/lib.
```

Install the missing prerequisite (sections 2–3) and run `dnf install` again.

## 5. Next

Continue with [Configure](Configure): the HORCM instance for `raidcom`, the array user
account, and `vsptop.conf`.

## Upgrading / removing

```sh
dnf upgrade https://github.com/visubramaniam/vsptop6-releases/releases/download/vX.Y.Z/vsptop6-X.Y.Z-1.el9.x86_64.rpm
dnf remove vsptop6         # keeps conf/vsptop.conf and var/topology.db
```

`vsptop.conf` is marked `config(noreplace)`: an upgrade never overwrites it; if the example
changed, the new one is written as `vsptop.conf.example` for comparison.
