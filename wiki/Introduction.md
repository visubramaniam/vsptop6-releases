# Introduction

**vsptop6** shows what a Hitachi VSP array is doing *right now*, the way `top` shows a Linux
host: a full-screen terminal view that refreshes every 5 seconds (configurable from 5 s to
60 s), sortable and filterable by any column, with the objects that matter correlated to each
other so you can move from a busy host group to its LDEVs, from an LDEV to its MP blade, from a
port to the host groups on it or to the initiators logged in to it — all with single keystrokes.

It is the C/ncurses successor of the original Perl `vsptop` (2014–2015) and keeps its
operating model: a small set of collector programs read performance counters — and, since 3.0,
the configuration too — straight from the array through the RAID Manager Library over a command
device. No daemon, no `raidcom`, no cron job.

## Supported arrays

Any array that supports the Hitachi RAID Manager Library (RMLIB) over a command device:
VSP One Block (B2x/B8x), VSP 5000, VSP E/F/G series. Fibre Channel, iSCSI and NVMe/TCP front
ends are all represented; mainframe (FICON, 3390-x) volumes appear with their emulation.

## The screens

```
ARRAYS ──Enter──► LDEVS ──h──► HOST GROUPS ──p──► PORTS ──l──► LDEVS
                    │ m            │ l               │ h    │ i
                    ▼              ▼                 ▼      ▼
                  MPBS ──l──► LDEVS            HOST GROUPS  INITIATORS ──Enter──► LDEVS
```

| Screen | Columns |
|---|---|
| **Arrays** | serial, name, #LDEVs, #host groups, #ports, cache used %, write-pending %, total IOPS and MB/s, age of the topology data |
| **LDEVs** | read/write IOPS, read/write response time (µs, whole numbers), read/write MB/s, RND% (random share of I/O), rHIT% / wHIT% (read / write cache-hit ratios), emulation, MP blade, pool, attributes (`VOL_ATTR` minus the constant `CVS`: HDP, JNL, POOL, GAD, ELUN …), NVMe subsystem id, SSID (mainframe), virtual LDEV id (GAD / VSM volumes), resource group (`RSGID` resolved to its `RS_GROUP` name), `NAA_ID` (the volume's NAA identifier as hosts see it, 4.0) |
| **Host groups** | host mode, type (FIBRE for WWN groups, iSCSI for IQN groups), #LDEVs, read/write IOPS, max response time, read/write MB/s, RND%, rHIT% (IOPS-weighted over the group's LDEVs), ports, pools, `WWNS` (registered initiators, 4.0) |
| **MP blades** | per MP: busy %, cache usage, write pending, role, owning resource, the owning LDEV's attributes, #LDEVs |
| **Ports** | type (FIBRE/ISCSI/NVME_TCP/FICON), attributes (TAR/MCU/RCU/ELUN), link speed (Gbps, negotiated, from RMLIB), IOPS, MB/s, read/write IOPS, read/write response time (µs), read/write MB/s, #LDEVs, `nWWN` (initiators currently logged in), NVMe subsystem ids |
| **Initiators** (4.0) | one row per registered initiator per port — FC WWN, iSCSI IQN or NVMe host NQN — with its nickname, `LOGIN` Y/N (live, FC ports), the port, its target (host group, iSCSI target or NVMe subsystem) and the LDEVs behind it; plus an amber row for every FC login that is registered in no host group on that port. Registrations need CCI (`raidcom`); the logins themselves come from RMLIB |

Every screen has a three-line header: the title (array, screen, scope, sort column, refresh
rate), a **Cache Summary** (capacity, usage and write-pending in MB and %) and a **Storage
Summary** (array-wide total/read/write IOPS and MB/s).

Rows past a threshold you set — response time (LDEVs, host groups, ports), MP busy %, port link
utilisation — are amber at the warning level and red at the problem level so they stand out at a glance (amber/green/underline are available via `highlight=`). Cache usage is shown without a warning colour — a Hitachi array keeps its cache full by design; write-pending is the figure to watch.

Screenshots of each screen, with notes, are on the [Screens](Screens) page.

## Two front ends

The RPM installs two programs that share everything but the screen layout:

- **`vsptop`** — the classic `top`-like table above: one screen at a time, three header lines.
- **`vspdash`** — a btop-style dashboard. The current array's IOPS, MB/s and response-time
  history, cache and pool capacity meters, MP-blade busy meters and per-port sparklines sit in
  panels above the same table, which gains a **trend** column graphing the sort column over the
  last eight samples. All keys are identical; drilling, sorting, filtering and `w` watch change
  the table panel while the panels above keep following the current array.

![vspdash](images/vspdash.png)

## How it works

```
 vsptop.conf ──► vsptop ◄── keyboard
                   │
                   ├─ at start and every hour (background) ──► topostat ──► topology.db (SQLite)
                   │                                              │           correlations
                   │                             optional: raidcom get host_grp / nvm_subsystem /
                   │                                       hba_wwn / hba_iscsi / host_nqn
                   │                                 (full names, NVMe subsystem ids, registered initiators)
                   │ one process per data source, re-sampled every refresh
                   ▼
   allldevstat   allportstat   allmpinfo   cachestat   loginstat        (RAID Manager Library)
```

- **`topostat`** reads the configuration records through the RAID Manager Library: every LDEV
  (emulation, attributes, MP, pool, SSID, virtual id, resource group, its paths to host groups
  and ports or NVMe namespaces), every port (type, attributes, negotiated link speed, the
  initiators logged in), host modes, pool capacity, resource groups and the array's NAA prefix,
  and `vsptop` stores the relationships in a small
  SQLite file. It runs when `vsptop` starts and the data is missing or older than an hour, and
  again in the background every hour; the arrays screen shows how old the data is.
- **`raidcom`** (CCI) is optional. If it is installed, each refresh also asks it for the
  things RMLIB cannot give — host-group names longer than 16 characters, NVMe subsystem ids
  and, since 4.0, the WWNs / IQNs / host NQNs registered in each host group, iSCSI target and
  NVMe subsystem. Without it, long names are cut at 16 characters, NVMe subsystems are
  numbered by position and the INITIATORS screen shows logins only. Nothing else depends on it.
  The registrations cost one `raidcom` call per defined host group per refresh (about 260
  calls and 35 s on a 16-port array with ~100 host groups, in the background; `vsptop -T -v`
  prints the count and time).
- **`loginstat`** (4.0) runs only while the PORTS or INITIATORS screen is up and re-reads each
  port's login table every refresh, so `LOGIN` and `nWWN` follow logins as they happen.
- **`vsptop`** starts the collectors it needs for the current screen, reads their output, joins
  it with the topology, and draws the table. Collectors are small C programs that talk to the
  array through the RAID Manager Library over a command device; they never write to the array.

## Since 3.1: passwords at rest and the version check

`vsptop -P` encrypts the array passwords in `vsptop.conf` under a key that never leaves the
host (`pass=enc:…`), and `pass=ask` prompts at start instead of storing anything. Once per
start `vsptop` also asks this repository's releases page whether a newer version exists and
shows `new: x.y.z` at the right edge of the footer — the only network request the tool ever
makes; `[GENERAL] VERSIONCHECK=0` turns it off. Details in [Configure](Configure) and
[Security](Security).

## What it is not

vsptop6 is a live viewer, not a historian: it keeps no history and draws no graphs. For
long-term trending use Ops Center Analyzer / VSP 360. It also makes no configuration changes
of any kind — the array account it uses only needs a view-only role.
