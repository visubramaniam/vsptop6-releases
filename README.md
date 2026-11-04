# vsptop6 — releases

[![Release](https://img.shields.io/github/v/release/visubramaniam/vsptop6-releases?label=release&color=2ea44f)](https://github.com/visubramaniam/vsptop6-releases/releases/latest)
[![Language](https://img.shields.io/badge/language-C11-00599C?logo=c&logoColor=white)](https://en.wikipedia.org/wiki/C11_(C_standard_revision))
[![Platform](https://img.shields.io/badge/platform-RHEL%209%20x86__64-EE0000?logo=redhat&logoColor=white)](https://github.com/visubramaniam/vsptop6-releases/wiki/Install)

Binary RPM releases of **vsptop6**, a `top`-like terminal monitor for Hitachi VSP storage
arrays (LDEV, host group, MP blade and port metrics with drill-down between them). The
source repository is private; this repository holds only the packaged releases.

## Install

RHEL / Rocky / Alma 9, x86_64. The host needs Hitachi **Command Control Interface** (CCI /
RAID Manager, `/HORCM`) and the **RAID Manager Library** (`/usr/lib64/libsvrrm.so`) installed
first; the package checks for both and refuses to install without them.

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v2.0.1/vsptop6-2.0.1-1.el9.x86_64.rpm   # or the latest on the Releases page
```

`dnf` pulls in `ncurses-libs` and `sqlite-libs`. Then:

```sh
vi /usr/local/san/vsp/conf/vsptop.conf     # arrays, RMLIB user/password (same on every array), command devices, thresholds
genarrayhash                               # builds the topology database (also hourly via cron)
vsptop                                     # top-like table
vspdash                                    # the same monitor as a dashboard (UTF-8, 256-colour terminal)
```

Upgrading from 1.x: copy the new `[THRESHOLD]` block from `vsptop.conf.example` into your
`vsptop.conf` (two-level warning/problem thresholds, no IOPS/MB/s thresholds) and run
`genarrayhash` once after installing — 2.0 records attributes, SSID, virtual LDEV ids, resource
groups, host modes, link speeds and pool capacity that older topology files lack.

Only one `vsptop`/`vspdash` runs per host (a second start is refused), and the refresh interval
is floored by the array's LDEV count, so a monitor cannot overload the array's controllers.

Full instructions are installed with the package in `/usr/local/san/vsp/doc/`
(`INSTALL.md`, `CONFIGURE.md`, `README.md`).
