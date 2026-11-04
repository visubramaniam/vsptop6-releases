# README — quick start and key reference

## Quick start

```sh
# 1. prerequisites on the host: Hitachi CCI (/HORCM) and the RAID Manager Library (/usr/lib64/libsvrrm.so)
# 2. install
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v2.0.1/vsptop6-2.0.1-1.el9.x86_64.rpm
# 3. configure: arrays, array user/password (see Configure for password rules), command devices
vi /usr/local/san/vsp/conf/vsptop.conf
# 4. one HORCM instance that knows every array (see Configure), then build the topology
genarrayhash
# 5. run
vsptop                 # top-like table
vspdash                # the same monitor as a btop-style dashboard
```

## What gets installed

```
/usr/local/san/vsp/bin/     vsptop, vspdash, genarrayhash, collectors
/usr/local/san/vsp/conf/    vsptop.conf, vsptop.conf.example
/usr/local/san/vsp/var/     topology.db
/usr/local/san/vsp/doc/     this documentation
/usr/bin/vsptop, /usr/bin/vspdash, /usr/bin/genarrayhash   (symlinks)
/etc/cron.d/vsptop6         hourly genarrayhash
```

## vspdash — the dashboard view

`vspdash` is the same monitor as a btop-style dashboard: the current array's IOPS / MB/s /
response-time history, cache and pool meters, MP-blade busy meters, per-port sparklines, and
the table you would see in `vsptop` in the bottom panel, with a right-hand graph column: a bar of
each MP's current busy % on the MP blade screen, and on the port screen a bar of how much of the
port's link speed (`LNKSPD`, Gbps) the current MB/s consumes — 32G fibre ≈ 3200 MB/s, 100G
Ethernet ≈ 12500 MB/s. Every `vsptop` key works unchanged (drill, sort,
filter, `w` watch); the panels above always follow the current array. It wants a UTF-8,
256-colour terminal (`TERM=xterm-256color`) and about 34 rows; narrower terminals get fewer
port rows, 8-colour terminals a reduced palette, non-UTF-8 locales ASCII graphs.

![vspdash](images/vspdash.png)

## Keys

| Key | Action |
|---|---|
| `Enter` | drill into the selected row: array → LDEVs, LDEV → its host groups, host group → its LDEVs, MP blade → its LDEVs, port → its host groups |
| `l` `h` `m` `p` | LDEVs / host groups / MP blades / ports **related to the selected row** |
| `L` `H` `M` `P` | all LDEVs / host groups / MP blades / ports of the array |
| `a` | arrays screen |
| `Esc`, `Backspace`, `←` | back |
| `s` / `S` | sort by the next / previous column (metrics descend, names ascend) |
| `r` | reverse the sort |
| `/` | filter rows by a substring (any column); `c` clears |
| `w` | **watch**: pin the top 8 rows of the current sort (sort by `rIOPS`, `BUSY%`, `MB/s`… first) and sample just those every second with the per-item collectors `ldevstat` / `mpbstat` / `portstat`. The title shows `WATCH top 8`. `w` again, or leaving the screen or array, goes back to the array-wide sampler and the previous refresh interval. On LDEV, MP blade and port screens only. Needs the per-item collectors installed (the RPM includes them). |
| `1`–`9`, `0` | refresh every 5, 10 … 45 s; `0` = 60 s (while watching, `1`–`9` set the watch interval in seconds). A floor applies by LDEV count — up to 1000 LDEVs 5 s, up to 10000 15 s, more 30 s — and cannot be lowered: the title shows `(min)` when it is in effect |
| `↑` `↓` `PgUp` `PgDn` `Home` `End` | move |
| `?` | help and the current thresholds |

The line above the footer spells out the abbreviations in the cursor row's ATTR (LDEV, MPB screens) or ROLE (port screen) column, e.g. `ATTR: GAD = global-active device, HDP = Dynamic Provisioning`.

Only one `vsptop`/`vspdash` runs per host: a second start is refused with a message naming the running one. Together with the refresh floor this keeps a monitor from overloading the array's controllers (see [Security](Security)).
| `q` | quit |

Rows at a threshold's warning level (`[THRESHOLD]` in `vsptop.conf`: response time 2–5 ms, MP busy or link utilisation 50–60 %) are amber, at the problem level (above 5 ms / 60 %) red; `highlight=` changes the problem colour (amber, green, underline, bold, reverse, dim).

## Command-line options

```
vsptop  [-c vsptop.conf] [-s serial[:screen[:scope:key]]] [-D] [-t ticks]
   -s 970272                 start on that array's LDEV screen
   -s 970272:ports           start on its port screen (ldevs | hgs | mpbs | ports)
   -s 970272:ldevs:hg:esx01  start scoped to a host group (hg | ldev | mpb | port)
   -D                        print the screen as text instead of curses (scripts, ssh without a tty)
   -t N                      exit after N refreshes (with -D)

genarrayhash  [-c vsptop.conf] [-s serial] [-n] [-v]
   -n   dry run: print the relationships as TSV, write nothing
   -v   show each raidcom command as it runs
```

## Troubleshooting

| Symptom | Meaning / fix |
|---|---|
| `** <collector> exited with no output for <serial>` in the footer | the collector could not log in to the array: check `user`/`pass` and `cmd` for that serial in `vsptop.conf` (password rules in [Configure](Configure)) |
| `TOPO AGE` shows `none` | `genarrayhash` has not run for that array — run it, check `raidcom -login` works |
| LDEV rows show `-` for metrics | the first sample takes one refresh interval; if it persists, see the `**` line |
| ports show `-` IOPS | same; on the arrays screen each array is sampled in turn, allow a few refreshes |
| install fails with "RAID Manager … not found" | install CCI and the RAID Manager Library first ([Install](Install)) |
