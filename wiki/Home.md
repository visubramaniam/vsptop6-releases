# vsptop6

A `top`-like terminal monitor for Hitachi VSP storage arrays — LDEVs, host groups,
MP blades and ports, with drill-down between them, refreshing every few seconds.

| Page | What it covers |
|---|---|
| [Introduction](Introduction) | What vsptop6 is, the screens, how it works |
| [README](README) | Quick start, `vsptop` and `vspdash`, key reference |
| [Install](Install) | Prerequisites (CCI, RAID Manager Library), command devices, the RPM |
| [Configure](Configure) | Array user account, `horcm0.conf`, `vsptop.conf`, building the topology |
| [README](README) | Quick start and key reference |
| [Security](Security) | Credentials, file permissions, what the tool can and cannot do |

Latest release: see the [Releases](https://github.com/visubramaniam/vsptop6-releases/releases) page.
Install in one line (RHEL / Rocky / Alma 9, x86_64, with CCI and RMLIB already present):

```sh
dnf install https://github.com/visubramaniam/vsptop6-releases/releases/download/v1.0.2/vsptop6-1.0.2-1.el9.x86_64.rpm
```
