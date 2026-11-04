# Configure

Three things to set up, in this order:

1. an **array user account** the tools log in with,
2. a **HORCM instance** so `raidcom` (used by `genarrayhash`) can reach every array,
3. **`vsptop.conf`**, read by both `genarrayhash` and `vsptop`.

Throughout, the examples use two arrays, serial `970272` on `/dev/sdb` and `970249` on
`/dev/sdc` (from `inqraid`, see [Install](Install) step 1).

## 1. Array user account

Create a user on each array and give it a **view-only** role — *Storage Administrator
(View Only)* is enough: vsptop6 only ever reads configuration and performance counters.
Users can only be created in the array's own management interface:

- **VSP One Block (B2x / B8x)**: the **Maintenance Utility** → *Administration → User
  Administration → User Groups / Users*;
- **VSP 5000 and earlier**: **Storage Navigator** → *Administration → User Groups* → create the
  user in a group that has the view-only role.

`raidcom` cannot create users, and Ops Center only manages its own accounts. The same array
account is then used for the `raidcom -login` that `genarrayhash` performs and for the RMLIB
login the collectors perform.

> **Monitoring several arrays: create the same user name and password on every array.**
> `genarrayhash` drives all arrays through one HORCM instance, and `raidcom -login` is
> per instance, not per array: `raidcom -login <user> <pass> -IH0` logs in to every array
> the instance knows in one go, while logging in to each array separately with `-s <serial>`
> does not work. `genarrayhash` therefore logs in once with the credentials of the **first**
> array in `vsptop.conf` and warns if any other array lists different ones. (The collectors
> attach to each command device individually, so they could use different accounts, but
> keeping one account everywhere is the only configuration that works for `genarrayhash`.)

> **Password rules.** No shell special characters — `#` `!` `?` `|` `$` `*` `[` `]` `\` `{`
> `}` `` ` `` `"` `'` `&` or spaces; use letters, digits and `; : . + - % @` instead.
> Upper- and lower-case letters are both fine: the credentials are passed to the array exactly
> as typed (the original Hitachi collectors used to lower-case them; vsptop6 removed that).
> The user name follows the same character rules.

Test it from the host once the HORCM instance below is running:

```sh
/HORCM/usr/bin/raidcom -login vsptop vsptop2026 -IH0 && echo login ok     # logs in to every array in instance 0
```

## 2. HORCM instance (`/etc/horcm0.conf`)

`raidcom` works through a HORCM daemon. vsptop6 expects **one instance that knows all the
arrays**; `genarrayhash` logs in to the instance once (`raidcom -login … -IH<n>`) and then
runs `raidcom get … -IH<n> -s <serial>` per array. Instance `0` is the default
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
it mode `600` — it contains the array password). `vsptop` and `genarrayhash` find it
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
cmd=/dev/sdb            # command device on THIS host (inqraid), used by the collectors

[970249]
unit=1
user=vsptop
pass=vsptop2026
cmd=/dev/sdc

# ---- highlighting ------------------------------------------------------------
[THRESHOLD]
highlight=red           # rows over a threshold: red (default) | amber | green | underline | bold | reverse | dim
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

[RMLIB]
INST=831                # base RMLIB instance number; each collector uses 831..837

# ---- paths and behaviour ----------------------------------------------------
[GENERAL]
# BIN_DIR and DB default to <prefix>/bin and <prefix>/var/topology.db, where
# <prefix> is the parent of this file's directory. Only set them to run from
# somewhere else (e.g. a source checkout).
#BIN_DIR=/usr/local/san/vsp/bin
#DB=/usr/local/san/vsp/var/topology.db
REFRESH=5               # seconds between samples; keys 1-9 and 0 change it live
DRILLDOWN=0             # 1: use ldevstat/portstat for views of <= 8 LDEVs/ports
```

Field by field:

| Key | Meaning |
|---|---|
| `[ARRAYS] serial=name` | every array to monitor; the name is what the screens show |
| `unit` | 0, 1, 2 … — an index used internally; any unique integers |
| `user` / `pass` | the array account from section 1 (no shell special characters). **Must be the same for every array** — `genarrayhash` logs in to the HORCM instance once with the first array's credentials |
| `cmd` | the command device **for that serial, on this host**; it must match the `HORCM_CMD` line for the same serial |
| `highlight` | how threshold breaches are drawn: `red` (default, reverse video), `amber`, `green`, `underline`, `bold`, `reverse`, `dim` |
| `responsewarn` / `responsealert`, `mpbwarn` / `mpbalert`, `linkwarn` / `linkalert` | warning (amber) and problem (red) levels for response time (LDEVs, host groups, ports — judged only while read or write IOPS exceed `activeiops`, default 100), MP busy % and port link utilisation; `?` in vsptop shows them. `ldevresponse` and `mpbbusy` from older files are read as the problem level; `portiops` / `portmbps` are ignored |
| `[HORCM] BASE`, `INST` | where `raidcom` is and which instance holds the command devices |
| `[RMLIB] INST` | base instance number handed to RMLIB; change only if 831–837 collide with other RMLIB clients on the host |
| `REFRESH` | sampling interval in seconds |
| `DRILLDOWN` | 1 enables the original per-item collectors for small views; 0 (default) uses the array-wide collectors everywhere |

Sections the old Perl version used (`[BACKEND]`, `[HTNM]`) are ignored.

## 4. Build the topology database

`genarrayhash` turns `raidcom` output into `topology.db` (LDEV ↔ host group ↔ port ↔ MPB ↔
pool, port roles/types, emulation). Dry-run it first to see what it finds:

```sh
genarrayhash -n -v | head -40        # -n: print edges, write nothing; -v: show each raidcom call
genarrayhash -n | awk -F'\t' '{print $1, $2}' | sort | uniq -c
```

You should see `HG_LDEV`, `PORT_HG`, `LDEV_MPB`, `LDEV_POOL`, `PORT_ROLE`, `LDEV_EMUL` … for
each serial. Typical problems and what they mean:

| Symptom | Cause |
|---|---|
| `(login returned non-zero; continuing)` then no edges | wrong user/password (the first array's are used for the whole instance) |
| `warning: <serial> has different credentials from <serial>` | arrays in `vsptop.conf` don't share one account; create the same user/password on each array |
| no edges for one serial only | that serial isn't in `horcm0.conf`, or its account differs from the first array's |
| edges only for one array | second `HORCM_CMD` line missing or HORCM not restarted after editing it |
| `raidcom: command device … not found` | `cmd=` or `HORCM_CMD` names a device that moved after a reboot — re-run `inqraid` |

Then write the database and keep it fresh:

```sh
genarrayhash                                   # writes /usr/local/san/vsp/var/topology.db
cat /etc/cron.d/vsptop6                        # installed by the RPM: hourly refresh
```

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
- the `TOPO AGE` column reads `none` → `genarrayhash` has not run for that serial.

A quick standalone test of one collector, exactly as `vsptop` runs it:

```sh
ARGS=$(mktemp)                                    # private 0600 file with an unpredictable name
printf '970272 vsptop vsptop2026 831 /dev/sdb\n' > "$ARGS"
/usr/local/san/vsp/bin/cacheinfo "$ARGS"        # expect: CLPR0,<size MB>,<used %>,<wp %>,<sf>
rm -f "$ARGS"
```
