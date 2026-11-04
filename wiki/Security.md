# Security

## What vsptop6 can do to an array

Nothing. Every call it makes is a read: `raidcom get ldev / port / host_grp / pool / resource`
for configuration and RAID Manager Library `getrminfo()` for performance counters. It never issues a `raidcom add/modify/delete`
or any RMLIB write. The array account it uses should therefore be **view-only**
(*Storage Administrator (View Only)*); it does not need, and should not have, any other role.

## Credentials

- The array user name and password are stored once, in `/usr/local/san/vsp/conf/vsptop.conf`.
  The package creates that file with mode `600`, owner `root`; keep it that way.
- Use a **dedicated account** with a long password rather than an administrator's.
- **Password characters:** do not use shell special characters — `#` `!` `?` `|` `$` `*`
  `[` `]` `\` `{` `}` `` ` `` `"` `'` `&` or spaces. Use letters, digits and `; : . + - % @`
  instead. (The tools pass the password through a whitespace-separated argument file and, for
  `raidcom -login`, through a shell; the excluded characters would break or alter that.)
- **Case:** credentials are case-sensitive and passed to the array exactly as typed. (The
  original Hitachi collector programs lower-cased them before logging in; vsptop6 removed
  that, so mixed-case user names and passwords work with every collector.)
- At run time `vsptop` hands each collector its credentials through a file in a private
  directory (`$XDG_RUNTIME_DIR/vsptop-<pid>/` or `/tmp/vsptop-<pid>/`, mode `0700`, files
  `0600`) that is removed when `vsptop` exits — never on the command line, so they are not
  visible in `ps`.
- `genarrayhash` logs in with `raidcom -login <user> <password>`; the password is on that
  command line for the fraction of a second the login takes. This is how CCI's login works.
  If that is a concern on a shared host, run `raidcom -login` interactively once per boot
  from a session that stays logged in, and leave `pass=` unused for `genarrayhash`.

## Privileges on the host

- Reading a command device requires access to the raw block device, which normally means
  `root`. `vsptop`, `genarrayhash` and the hourly cron job run as root for that reason.
- The tools open no network ports and make no outbound connections. `raidcom` talks to the
  local HORCM daemon over `localhost`; HORCM talks to the array over the command device (in-band)
  only.
- The topology database (`/usr/local/san/vsp/var/topology.db`, directory mode `700`) contains
  LDEV ids, host-group names, port and pool ids — configuration metadata, no credentials and no
  host data.

## Package

- The RPM is a binary package. Its source repository is private; the collectors contain
  Hitachi RAID Manager Library interface details that are not redistributable.
- The package checks for `/HORCM/usr/bin/raidcom` and `libsvrrm.so` before installing and
  refuses otherwise; it adds no users, no services and no SUID binaries. Installed files:
  `/usr/local/san/vsp/**`, the two `/usr/bin` symlinks and `/etc/cron.d/vsptop6`.
- Each release's RPM is attached to a [GitHub Release](https://github.com/visubramaniam/vsptop6-releases/releases)
  with its SHA-256 shown on the page; verify after download with `sha256sum`.

## How the code is checked

Every change to the (private) source repository runs through three GitHub Actions workflows
before it can become a release:

| Workflow | What it does |
|---|---|
| **CI** | builds `vsptop` and `genarrayhash` on Rocky Linux 9 and runs the test suite against mock collectors and recorded `raidcom` output |
| **Code scan** | compiles all vsptop6 code with `-Wall -Wextra -Werror`; **cppcheck** (warning, performance, portability); **gcc -fanalyzer**; **flawfinder** (any level-4+ finding fails the build unless reviewed and annotated in the source); an **AddressSanitizer + UndefinedBehaviorSanitizer** build running the full test suite with leak detection; **shellcheck**; **rpmlint** on the package spec |
| **Release** | builds the RPM on a host that has CCI/RMLIB, after CI and the scan have passed |

Design choices that follow from this:

- `genarrayhash` runs `raidcom` with `fork`/`execv`, not through a shell: the array password
  is passed as a plain argument and is never interpreted by `sh`.
- Collectors receive their credentials through a `0600` file in a private per-process
  directory, never on a command line.
- Every memory allocation is checked; the sanitizer run has no findings.
- The original Hitachi collector programs are compiled but not gated: they are kept
  byte-for-byte as received and are only used as fallbacks when the replacement collectors
  are absent.

GitHub's own code scanning (CodeQL) is not used because it requires GitHub Advanced Security
on a private repository.

## Reporting a problem

Open an issue on [vsptop6-releases](https://github.com/visubramaniam/vsptop6-releases/issues).
For anything security-sensitive (a way to change array state, credential exposure), do not
post details publicly — open the issue with just a request for contact and it will be followed
up privately.
