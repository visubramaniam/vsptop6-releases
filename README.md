# vsptop6 — releases

[![Release](https://img.shields.io/github/v/release/visubramaniam/vsptop6-releases?label=release&color=2ea44f)](https://github.com/visubramaniam/vsptop6-releases/releases/latest)
[![Language](https://img.shields.io/badge/language-C11-00599C?logo=c&logoColor=white)](https://en.wikipedia.org/wiki/C11_(C_standard_revision))
[![Platform](https://img.shields.io/badge/platform-RHEL%209%20x86__64-EE0000?logo=redhat&logoColor=white)](https://github.com/visubramaniam/vsptop6-releases/wiki/Install)

Binary RPM releases of **vsptop6**, a `top`-like terminal monitor for Hitachi VSP storage
arrays (LDEV, host group, MP blade and port metrics with drill-down between them). The
source repository is private; this repository holds only the packaged releases.

## Install

RHEL / Rocky / Alma 9, x86_64. The host needs the Hitachi **RAID Manager Library**
(`/usr/lib64/libsvrrm.so`) installed first; the package checks for it and refuses to install
without it. Hitachi **Command Control Interface** (CCI / `raidcom`) is optional since 3.0: when
present it adds host-group names longer than 16 characters and NVMe subsystem ids to the
topology; everything else comes from RMLIB over the command device.

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v3.0.1/vsptop6-3.0.1-1.el9.x86_64.rpm   # or the latest on the Releases page
```

`dnf` pulls in `ncurses-libs` and `sqlite-libs`. Then:

```sh
vi /usr/local/san/vsp/conf/vsptop.conf     # arrays, RMLIB user/password, command devices, thresholds
vsptop                                     # top-like table; builds the topology from RMLIB on first start, refreshes it hourly
vspdash                                    # the same monitor as a dashboard (UTF-8, 256-colour terminal)
```

Upgrading from 2.x: `genarrayhash` and its cron job are gone — the first `vsptop` start
rebuilds the topology from RMLIB (`vsptop -T -v` does it by hand). New in `vsptop.conf`:
`[HORCM] USE=auto|yes|no` (whether to use `raidcom` when present), `[GENERAL] TOPOREFRESH=60`
(minutes between refreshes) and `[LNKSPD]` (per-port speed overrides, normally empty). Copy
them from `vsptop.conf.example`. Upgrading from 1.x: also copy the `[THRESHOLD]` block.

Only one `vsptop`/`vspdash` runs per host (a second start is refused), and the refresh interval
is floored by the array's LDEV count, so a monitor cannot overload the array's controllers.

Full instructions are installed with the package in `/usr/local/san/vsp/doc/`
(`INSTALL.md`, `CONFIGURE.md`, `README.md`).
