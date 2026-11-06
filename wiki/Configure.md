# Configure

Three things to set up, in this order:

1. an **array user account** the tools log in with,
2. **`vsptop.conf`**, read by `vsptop` and `vspdash`,
3. optionally a **HORCM instance** so `raidcom` can reach every array — only with CCI installed,
   and only for host-group names longer than 16 characters and NVMe subsystem ids; everything
   else comes from RMLIB through the command device, with no daemon.

Throughout, the examples use two arrays, serial `970272` on `/dev/sdb` and `970249` on
`/dev/sdc` (see [Install](Install) step 1 for finding the command devices).

## 1. Array user account

Create a user on each array and give it a **view-only** role — *Storage Administrator
(View Only)* is enough: vsptop6 only ever reads configuration and performance counters.
Users can only be created in the array's own management interface:

- **VSP One Block (B2x / B8x)**: the **Maintenance Utility** → *Administration → User
  Administration → User Groups / Users*;
- **VSP 5000 and earlier**: **Storage Navigator** → *Administration → User Groups* → create the
  user in a group that has the view-only role.

`raidcom` cannot create users, and Ops Center only manages its own accounts. The same array
account is then used for the RMLIB login the collectors perform and, when `raidcom` is
installed, for the `raidcom -login` that the topology refresh performs.

> **Monitoring several arrays with `raidcom` enrichment: create the same user name and
> password on every array.** The collectors (including `topostat`) attach to each command
> device individually, so RMLIB alone works with different accounts per array. The optional
> `raidcom` enrichment does not: `raidcom -login` is per HORCM instance, not per array —
> `raidcom -login <user> <pass> -IH0` logs in to every array the instance knows in one go,
> while logging in to each array separately with `-s <serial>` does not work. The refresh
> therefore logs in with the credentials of the array being refreshed and the login applies
> to all of them, so with different accounts only the last login wins. Either keep one
> account everywhere or set `[HORCM] USE=no`.

> **Password rules.** No shell special characters — `#` `!` `?` `|` `$` `*` `[` `]` `\` `{`
> `}` `` ` `` `"` `'` `&` or spaces; use letters, digits and `; : . + - % @` instead.
> Upper- and lower-case letters are both fine: the credentials are passed to the array exactly
> as typed (the original Hitachi collectors used to lower-case them; vsptop6 removed that).
> The user name follows the same character rules.

Test it from the host: put the account in `vsptop.conf` (section 3) and run `vsptop -T -v`;
a wrong user name or password shows as `topostat exited with status 2`. With CCI installed and
the HORCM instance from section 2 running, `raidcom -login vsptop vsptop2026 -IH0 && echo
login ok` tests the same account for the enrichment.

## 2. HORCM instance (`/etc/horcm0.conf`) — only with CCI

`raidcom` works through a HORCM daemon. This section is optional and needs CCI installed ([Install](Install) section 3): without CCI (or with
`[HORCM] USE=no`) the topology is complete except that host-group names are cut at 16
characters (the RMLIB limit) and NVMe subsystems are numbered by position instead of by
their subsystem id. With it, vsptop6 expects **one instance that knows all the arrays**; each
topology refresh logs in (`raidcom -login … -IH<n>`) and runs `raidcom get host_grp -key
host_grp` per port and `raidcom get nvm_subsystem` with `-IH<n> -s <serial>`. Instance `0` is the default
(`[HORCM] INST=0` in `vsptop.conf`).

Create `/etc/horcm0.conf`:

```
HORCM_MON
#ip_address     service         poll(10ms)      timeout(10ms)
localhost       31000           1000            3000

HORCM_CMD
#dev_name
\\.\CMD-970272:/dev/sdb
\\.\CMD-970249:/dev/sdc

HORCM_LDEV
#dev_group      dev_name        Serial#         CU:LDEV(LDEV#)  MU#

HORCM_INST
#dev_group      ip_address      service
```

Notes:

- One `HORCM_CMD` line per array: `\\.\CMD-<serial>:/dev/sdX`. The `CMD-<serial>` form
  survives device renumbering after a reboot better than a bare `/dev/sdX`; you can list
  several devices for one serial separated by colons for redundancy.
- `HORCM_LDEV` and `HORCM_INST` stay empty — vsptop6 does not manage replication pairs.
- Port `31000` must be free; each HORCM instance needs its own port.

Start it and check:

```sh
/HORCM/usr/bin/horcmstart.sh 0
/HORCM/usr/bin/raidqry -l -IH0                 # lists both arrays with their serials
/HORCM/usr/bin/raidcom -login vsptop vsptop2026 -IH0
/HORCM/usr/bin/raidcom get ldev -ldev_list mapped -key front_end -fx -IH0 -s 970272 | head -3
```

To start it at boot, use CCI's own `horcmgr` service setup, or a small systemd unit:

```ini
# /etc/systemd/system/horcm0.service
[Unit]
Description=Hitachi HORCM instance 0
After=network.target

[Service]
Type=forking
ExecStart=/HORCM/usr/bin/horcmstart.sh 0
ExecStop=/HORCM/usr/bin/horcmshutdown.sh 0
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```sh
systemctl enable --now horcm0
```

If you must use a different instance number, set `[HORCM] INST=` in `vsptop.conf` to match.

## 3. `vsptop.conf`

Location: `/usr/local/san/vsp/conf/vsptop.conf` (created from the example by the RPM; keep
it mode `600` — it contains the array password). `vsptop` and `vspdash` find it
automatically; `-c <path>` or `VSPTOP_CONF=<path>` overrides.

```ini
# ---- the arrays: serial = display name ------------------------------------
[ARRAYS]
970272=DC1_B85
970249=DC2_B85

# ---- one section per serial ------------------------------------------------
[970272]
unit=0                  # index; keep them unique and in order
user=vsptop             # array user
pass=vsptop2026         # see section 1: no shell special characters
cmd=/dev/sdb            # command device on THIS host (lsblk MODEL OPEN-V-CM / inqraid), used by the collectors

[970249]
unit=1
user=vsptop
pass=vsptop2026
cmd=/dev/sdc

# ---- highlighting ------------------------------------------------------------
[THRESHOLD]
highlight=red           # rows over a threshold: red (default) | amber | green | underline | bold | reverse | dim
theme=auto              # auto (terminal colours, default) | dark (black background) | light (white background); d toggles
responsewarn=2000       # response time, microseconds: LDEV r/w RT, host-group max RT, port r/w RT
responsealert=5000      #   2-5 ms is a warning (amber), above 5 ms a problem (red)
activeiops=100          #   ...and only when read or write IOPS exceed this: idle volumes report noisy response times
mpbwarn=50              # MP busy %: 50-60 warning, above 60 a problem
mpbalert=60
linkwarn=50             # port link utilisation (MB/s as % of LNKSPD): 50-60 warning, above 60 a problem
linkalert=60
# no thresholds on IOPS or MB/s: both depend on the block size

# ---- CCI -----------------------------------------------------------------------
[HORCM]
BASE=/HORCM             # where CCI is installed (raidcom is BASE/usr/bin/raidcom)
INST=0                  # the HORCM instance from section 2
USE=auto                # auto: enrich the topology with raidcom if BASE/usr/bin/raidcom exists; yes | no

# ---- port link speed overrides (Gbps) ---------------------------------------
[LNKSPD]
# Normally empty: the negotiated speed comes from RMLIB. Only for a
# port that still shows AUTO (link down at refresh time, or an array that does not
# report it) list the speed here; it feeds the link utilisation columns.
#CL1-A=32

[RMLIB]
INST=831                # base RMLIB instance number; each collector uses 831..837

# ---- paths and behaviour ----------------------------------------------------
[GENERAL]
# BIN_DIR and DB default to <prefix>/bin and <prefix>/var/topology.db, where
# <prefix> is the parent of this file's directory. Only set them to run from
# somewhere else (e.g. a source checkout).
#BIN_DIR=/usr/local/san/vsp/bin
#DB=/usr/local/san/vsp/var/topology.db
REFRESH=5               # seconds between samples; keys 1-9 and 0 change it live  (floored by LDEV count: <=1000 LDEVs 5 s, <=10000 15 s, more 30 s — cannot be lowered)
DRILLDOWN=0             # 1: sample only the LDEVs/ports in view when there are <= 64
TOPOREFRESH=60          # minutes between topology refreshes (topostat, in the background); 0 = only at startup / -T
```

Field by field:

| Key | Meaning |
|---|---|
| `[ARRAYS] serial=name` | every array to monitor; the name is what the screens show |
| `unit` | 0, 1, 2 … — an index used internally; any unique integers |
| `user` / `pass` | the array account from section 1 (no shell special characters). Must be the same for every array if `raidcom` enrichment is used (`raidcom -login` is per HORCM instance) |
| `cmd` | the command device **for that serial, on this host**; it must match the `HORCM_CMD` line for the same serial |
| `highlight` | how threshold breaches are drawn: `red` (default, reverse video), `amber`, `green`, `underline`, `bold`, `reverse`, `dim` |
| `theme` | `auto` (default): the terminal's own background with the dark palette; `dark`: paint the background black; `light`: paint it white with a palette of darker shades (for light terminals or printing). The `d` key toggles dark/light while running |
| `responsewarn` / `responsealert`, `mpbwarn` / `mpbalert`, `linkwarn` / `linkalert` | warning (amber) and problem (red) levels for response time (LDEVs, host groups, ports — judged only while read or write IOPS exceed `activeiops`, default 100), MP busy % and port link utilisation; `?` in vsptop shows them. `ldevresponse` and `mpbbusy` from older files are read as the problem level; `portiops` / `portmbps` are ignored |
| `[HORCM] BASE`, `INST` | where `raidcom` is and which instance holds the command devices |
| `[HORCM] USE` | `auto` (default): use `raidcom` for full host-group names and NVMe subsystem ids when `BASE/usr/bin/raidcom` exists; `yes` forces it (errors are reported); `no` never runs `raidcom` |
| `[LNKSPD] CLn-X=Gbps` | overrides the port speed RMLIB reports for the link utilisation columns; normally not needed since the negotiated speed is read from RMLIB |
| `[RMLIB] INST` | base instance number handed to RMLIB; change only if 831–837 collide with other RMLIB clients on the host |
| `REFRESH` | sampling interval in seconds |
| `DRILLDOWN` | 1 scopes the LDEV and port collectors (`ids=`) to the objects in view when there are 64 or fewer; 0 (default) samples the whole array everywhere |
| `TOPOREFRESH` | minutes after which an array's topology is rebuilt in the background while `vsptop` runs (default 60; 0 disables periodic refresh) |

Sections the old Perl version used (`[BACKEND]`, `[HTNM]`) are ignored.

## 4. Build the topology database

`vsptop` builds `topology.db` itself: on the first start (or whenever an array has no
topology, or one older than `TOPOREFRESH` minutes) it runs the `topostat` collector for that
array, which reads LDEV ↔ host group ↔ port ↔ MPB ↔ pool, port types/roles/speeds, emulation,
attributes, resource groups and pool usage straight from RMLIB. Later refreshes happen in
the background while the screens keep running. To build or check it by hand:

```sh
vsptop -T -v                 # rebuild every array and exit; -v shows each step
vsptop -T -s 970272          # one array only
/usr/local/san/vsp/bin/topostat /tmp/args | awk -F'\t' '{print $1, $2}' | sort | uniq -c   # raw edges (args file as in section 5)
```

You should see `HG_LDEV`, `PORT_HG`, `LDEV_MPB`, `LDEV_POOL`, `PORT_ROLE`, `LDEV_EMUL` … for
each serial. Typical problems and what they mean:

| Symptom | Cause |
|---|---|
| `topostat … exited with status 2` and no edges | wrong `user`/`pass` or `cmd` for that serial (RMLIB login failed) |
| `raidcom: … login` warnings, edges still stored | the enrichment login failed: arrays don't share one account, or `[HORCM] INST` is wrong; set `USE=no` to silence it |
| host-group names cut at 16 characters, NVMSSID shows 0,1,2… | `raidcom` not installed or `USE=no` — the RMLIB limit; install CCI or live with it |
| `LNKSPD` shows `AUTO` and link utilisation `-` on a port | RMLIB reported no negotiated speed (link down when the topology was built, or not reported by that array): rebuild with `vsptop -T -s <serial>` once the link is up, or add the port to `[LNKSPD]` |
| `raidcom: command device … not found` | `cmd=` or `HORCM_CMD` names a device that moved after a reboot — re-run `lsblk` / `inqraid` |

## 5. Run

```sh
vsptop
```

The arrays screen lists every array from `vsptop.conf`; `TOPO AGE` shows how old the
database is. `Enter` drills into an array; `?` lists the keys. If a screen stays on dashes:

- `** <collector> exited with no output …` in the footer → the collector could not log in:
  check `user`/`pass` and `cmd` for that serial (password rules in section 1);
- `Cache Summary … (waiting for cachestat)` for more than two refreshes → same check for
  `cachestat`;
- the `TOPO AGE` column reads `none` → the topology build for that serial failed; run `vsptop -T -v -s <serial>` to see why.

A quick standalone test of one collector, exactly as `vsptop` runs it:

```sh
printf '970272 vsptop vsptop2026 831 /dev/sdb\n' > /tmp/args; chmod 600 /tmp/args
timeout 8 /usr/local/san/vsp/bin/cachestat /tmp/args   # expect lines: CLPR0,<size MB>,<used %>,<wp %>,<sf>
rm -f /tmp/args
```
