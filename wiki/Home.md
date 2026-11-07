# vsptop6

A `top`-like terminal monitor for Hitachi VSP storage arrays — LDEVs, host groups,
MP blades, ports and (since 4.0) initiators, with drill-down between them, refreshing every few seconds. Two front
ends share one engine: `vsptop` (classic table) and `vspdash` (btop-style dashboard).

| Page | What it covers |
|---|---|
| [Introduction](Introduction) | What vsptop6 is, the screens, how it works |
| [Screens](Screens) | Annotated screenshots of every screen |
| [README](README) | Quick start, `vsptop` and `vspdash`, key reference |
| [Install](Install) | Prerequisites (RAID Manager Library; CCI optional), command devices, the RPM |
| [Configure](Configure) | Array user account, `vsptop.conf`, passwords at rest (`vsptop -P`), the topology, optional `horcm0.conf` |
| [Security](Security) | Credentials, encrypted passwords, file permissions, the version check, what the tool can and cannot do |

Latest release: see the [Releases](https://github.com/visubramaniam/vsptop6-releases/releases) page.
Install in one line (RHEL / Rocky / Alma 9, x86_64, with the RAID Manager Library already present):

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v4.0.0/vsptop6-4.0.0-1.el9.x86_64.rpm
```
