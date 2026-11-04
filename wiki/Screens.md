# Screens

All screenshots are `vspdash` 2.0.1 on a VSP One Block 28 (serial 840477) during a synthetic
load test (fio on the HCI-FC host groups, ~1 M IOPS). `vsptop` shows the same tables without
the panels above them. The title line carries the program and version, the array, the screen,
the sort, the refresh interval, the date and time, and the topology summary (age, counts). The
line above the footer spells out the abbreviations in the cursor row's ATTR or ROLE column.

In every view the panels above the table follow the current array: IOPS / MB/s / read and
write response-time history (braille, 64 samples), cache (CLPR size, usage, write-pending,
read-hit %), pool capacity, the six busiest ports with their link type and response time, and
MP-blade busy %. Amber means a warning threshold, red a problem threshold; cache usage is
never coloured because an array runs its cache full by design.

## Arrays

![Arrays](images/screen-arrays.png)

The start screen: one row per array in `vsptop.conf` — LDEV, host-group and port counts
from the topology, cache usage and write-pending %, total IOPS and MB/s from a round-robin
sample, and the age of the topology data. The panels above show the array under the cursor.
`Enter` opens its LDEVs; `l h m p` open LDEVs, host groups, MP blades or ports directly.

## LDEVs

![LDEVs](images/screen-ldevs.png)

Every LDEV of the array (or those of the host group, port or MP blade you drilled from): read
and write IOPS, response time (µs), MB/s, the random share of I/O (`RND%`), read and write
cache-hit ratios (`rHIT%`, `wHIT%`), emulation, MP blade, pool, volume attributes (`ATTR`),
NVMe subsystem id, mainframe SSID, virtual LDEV id for GAD/VSM volumes, and the resource
group. Here the fio volumes run at 80 k IOPS each with sub-millisecond reads; `00:01:70` is
red because its 9.4 / 13.9 ms response times are over the 5 ms problem level while it is busy
(idle volumes are never flagged). Sort with `s`/`S`, filter with `/` (e.g. `/JNL`), and press
`w` to watch the top eight at one-second resolution.

## Host groups

![Host groups](images/screen-hostgroups.png)

One row per host group: host mode and type (FIBRE for WWN groups, iSCSI for IQN groups),
member count, read/write IOPS, the worst response time among its LDEVs, MB/s, `RND%` and
`rHIT%` weighted by IOPS, the ports it is defined on and the pools its volumes draw from. The
four HCI-FC groups driving the test are red: their busy LDEVs are above the response-time
problem level. `Enter` lists the group's LDEVs; `p` its ports.

## MP blades

![MP blades](images/screen-mpbs.png)

One row per MP (processor core) grouped by blade/MPU: busy %, cache usage and write-pending
% of its CLPR, the role of the resource it is currently serving (TARGET, SYSTEM, …), that
resource's id (`OWNER`) and, when it is an LDEV, the LDEV's attributes, plus the number of
LDEVs assigned to the blade. The right-hand column is a bar of each MP's current busy %. Under
this test every MP of MPB0 is in the 65–75 % band — above the 60 % problem level, hence red —
while the MP blades panel above summarises both MPUs. `Enter` lists the LDEVs assigned to the
blade.

## Ports

The port screen (not pictured) lists each installed port with its type, attributes
(`TAR`, `MCU`, `RCU`, `ELUN`), link speed, IOPS and MB/s, read/write IOPS, read/write
response time, read/write MB/s, the number of LDEVs mapped and NVMe subsystem ids; in
`vspdash` the right-hand column is a bar of how much of the link speed the current MB/s
consumes (`LINK%`). `Enter` lists the port's LDEVs; `h` its host groups.
