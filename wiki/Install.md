# Install

vsptop6 runs on a Linux host that has a **command device** from each VSP array mapped to it.
Everything it needs from the array comes through one Hitachi component:

| Prerequisite | Used by | What it is |
|---|---|---|
| **RAID Manager Library (RMLIB)** — required | every collector: `allldevstat`, `allportstat`, `allmpinfo`, `cachestat` (metrics) and `topostat` (the topology) | `libsvrrm.so`, the shared library the collectors load at run time; it talks to the array over the command device directly, no daemon |
| **Command Control Interface (CCI / RAID Manager)** — optional | topology enrichment only | the `raidcom` CLI and the HORCM daemon under `/HORCM`. Since 3.0 vsptop6 uses it, when present, for the two things RMLIB cannot give: host-group names longer than 16 characters and NVMe subsystem ids. Without it those names are cut at 16 characters and NVMe subsystems are numbered by position; everything else is identical |

The RPM refuses to install until RMLIB is present and only prints a note when CCI is absent.
Supported platform: RHEL / Rocky / Alma 9, x86_64 (the release RPM is tagged `.el9`).

## 1. Prepare the host

```sh
dnf install -y rpm-build gcc make        # only if you will build from source; not needed to install the RPM
```

Map a **command device** from every array you want to monitor:

1. On the array (Storage Navigator / Ops Center Administrator / `raidcom`), take a small LDEV
   (a few MB is enough), enable the *Command Device* attribute on it and turn on
   **user authentication** (`raidcom modify ldev -ldev_id <id> -command_device y -security 1`
   or the equivalent in the GUI). RMLIB logs in with a user name and password, so an
   unauthenticated command device is not enough.
2. Map it to the vsptop host through a host group / LUN path like any other LUN.
3. On the host, rescan and identify it:
   ```sh
   rescan-scsi-bus.sh                   # or echo "- - -" > /sys/class/scsi_host/hostN/scan
   lsblk -o NAME,VENDOR,MODEL,SIZE | grep OPEN-V-CM     # command devices report MODEL OPEN-V-CM
   ```
   With CCI installed, `echo /dev/sd* | /HORCM/usr/bin/inqraid -f -CLI` also prints the array
   **SERIAL** per device. Without it, put the candidate `/dev/sdX` in `vsptop.conf` and run
   `vsptop -T -v`: the collectors print `warning: /dev/sdX answered as serial N, vsptop.conf
   says M` when the device belongs to another array, so one pass sorts out which is which.
   Note the `/dev/sdX` name for each serial.

If several hosts map the same command device, each host needs its own; command devices are
not shared between hosts.

## 2. Install the RAID Manager Library (RMLIB)

RMLIB is its own download on [Hitachi Vantara Support Connect](https://support.hitachivantara.com)
(*RAID Manager Library* / `RMLIB`), separate from the CCI package. Install the 64-bit library:

```sh
mkdir -p /tmp/rmlib && cd /tmp/rmlib
unzip <downloaded-rmlib-package>.zip
# follow the readme in the package; on Linux x64 it installs
#   /usr/lib64/libsvrrm.so           (64-bit: the one the vsptop6 collectors load)
#   /usr/lib/libsvrrm.so             (32-bit; harmless but not used)
#   /opt/RMLIB/bin/whatrmver         (prints the library version)
# The package also ships rmlib.h; that header is only needed to compile the
# collectors from source, not to run the RPM's binaries.
ldconfig
```

Verify — this is what the RPM's pre-install check looks for:

```sh
ls -l /usr/lib64/libsvrrm.so*
file -L /usr/lib64/libsvrrm.so       # must say "ELF 64-bit"
/opt/RMLIB/bin/whatrmver             # Ver&Rev: 01-34-03/00 or similar; vsptop shows it in its title line
```

If only `/usr/lib/libsvrrm.so` exists you have the 32-bit library; install the x64 package.

## 3. Optional: install CCI (RAID Manager) for full host-group names and NVMe ids

Skip this section if 16-character host-group names and positional NVMe subsystem numbers are
acceptable, or if the host must not run a HORCM daemon. Download *Command Control Interface*
for Linux from Support Connect (the package is named like `RMHORC_X64` inside the CCI media;
use the x64 build). Then, as root:

```sh
mkdir -p /tmp/cci && cd /tmp/cci
unzip <downloaded-cci-package>.zip            # or mount the ISO
cd /                                          # CCI installs relative to the root
cpio -idmu < /tmp/cci/LINUX/RMHORC_X64         # creates /HORCM
/HORCM/horcminstall.sh
/HORCM/usr/bin/raidqry -h | head -3          # prints the CCI version
```

vsptop6 finds `raidcom` at `/HORCM/usr/bin/raidcom` (`[HORCM] BASE` in `vsptop.conf`) and
uses it automatically (`[HORCM] USE=auto`); the HORCM instance it needs is described in
[Configure](Configure) section 2. Nothing in vsptop6 depends on `/HORCM/usr/bin` being
in `PATH`.

## 4. Install vsptop6

From the GitHub release (the `.el9` RPM works on RHEL/Rocky/Alma 9):

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v3.0.1/vsptop6-3.0.1-1.el9.x86_64.rpm
```

`dnf` fetches `ncurses-libs` and `sqlite-libs` from the OS repositories, runs the
prerequisite check and installs:

```
/usr/local/san/vsp/bin/        vsptop, vspdash, the collectors (incl. topostat)
/usr/local/san/vsp/conf/       vsptop.conf (created from vsptop.conf.example on first install)
/usr/local/san/vsp/var/        topology.db (written by vsptop from topostat)
/usr/local/san/vsp/doc/        README, methodology
/usr/bin/vsptop, /usr/bin/vspdash   symlinks
```

If the check fails you'll see:

```
vsptop6: the RAID Manager Library (RMLIB, libsvrrm.so) was not found in /usr/lib64 or /usr/lib.
```

Install RMLIB (section 2) and run `dnf install` again. A note that `/HORCM/usr/bin/raidcom`
is not installed is informational (section 3).

Building from source instead: see the README (`make`, `make collectors`, `make install`) —
the same prerequisites apply, plus `ncurses-devel` and `sqlite-devel`.

## 5. Next

Continue with [Configure](Configure): the array user account, `vsptop.conf`, and (only
with CCI) the HORCM instance for `raidcom`.

## Upgrading / removing

```sh
dnf upgrade https://github.com/visubramaniam/vsptop6-releases/releases/download/vX.Y.Z/vsptop6-X.Y.Z-1.el9.x86_64.rpm
dnf remove vsptop6         # keeps conf/vsptop.conf and var/topology.db
```

`vsptop.conf` is marked `config(noreplace)`: an upgrade never overwrites it; if the example
changed, the new one is written as `vsptop.conf.example` for comparison. Upgrading from 2.x
also removes the old `genarrayhash` binary, its `/usr/bin` symlink and `/etc/cron.d/vsptop6`.
