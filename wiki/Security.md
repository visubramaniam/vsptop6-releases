# Security

## What vsptop6 can do to an array

Nothing. Every call it makes is a read: RAID Manager Library `getrminfo()` for configuration
records and performance counters and, only when CCI is installed, `raidcom get host_grp` and
`raidcom get nvm_subsystem`. It never calls `setrminfo()` or any `raidcom add/modify/delete`. The array account it uses should therefore be **view-only**
(*Storage Administrator (View Only)*); it does not need, and should not have, any other role.

## Credentials

- The array user name and password are stored once, in `/usr/local/san/vsp/conf/vsptop.conf`.
  The package creates that file with mode `600`, owner `root`; keep it that way.
- Use a **dedicated account** with a long password rather than an administrator's.
- **Password characters:** do not use shell special characters — `#` `!` `?` `|` `$` `*`
  `[` `]` `\` `{` `}` `` ` `` `"` `'` `&` or spaces. Use letters, digits and `; : . + - % @`
  instead. (The tools pass the password through a whitespace-separated argument file; the
  excluded characters would break or alter that.)
- **Case:** credentials are case-sensitive and passed to the array exactly as typed. (The
  original Hitachi collector programs lower-cased them before logging in; vsptop6 removed
  that, so mixed-case user names and passwords work with every collector.)
- At run time `vsptop` hands each collector its credentials through a file in a private
  directory (`$XDG_RUNTIME_DIR/vsptop-<pid>/` or `/tmp/vsptop-<pid>/`, mode `0700`, files
  `0600`) that is removed when `vsptop` exits — never on the command line, so they are not
  visible in `ps`.
- With CCI installed (optional), each topology refresh logs in to HORCM by running
  `raidcom -login` **without arguments** on a private pseudo-terminal and answering its
  per-array `User` / `Password` prompts from `vsptop.conf` — the same way an operator would
  type them. The password never appears on a command line, in `ps`, or in the environment.

## One instance per host

`vsptop` and `vspdash` take a lock (`var/vsptop.lock`, next to `topology.db`) while they run,
so a second copy started by anyone on the same host is refused with a message naming the
running one (program, pid, user, start time). Each instance drives its own RMLIB samplers
against the same command devices, and two sets would double the load on the array's
controllers — the same reason the refresh interval is floored by LDEV count. Several people
can still share one session (`tmux`/`screen`), or run from different hosts.

## Privileges on the host

- Reading a command device requires access to the raw block device, which normally means
  `root`. `vsptop` and `vspdash` (and the collectors they start) run as root for that reason.
  There is no cron job and no daemon of vsptop6's own.
- The tools open no network ports and make no outbound connections: RMLIB talks to the array
  over the command device (in-band) only. If CCI is used, `raidcom` talks to the local HORCM
  daemon over `localhost`, and HORCM to the array over the same command device.
- The topology database (`/usr/local/san/vsp/var/topology.db`, directory mode `700`) contains
  LDEV ids, host-group names, port and pool ids — configuration metadata, no credentials and no
  host data.

## Package

- The RPM is a binary package. Its source repository is private; the collectors contain
  Hitachi RAID Manager Library interface details that are not redistributable.
- The package checks for `libsvrrm.so` before installing and refuses otherwise (CCI is only
  noted when absent); it adds no users, no services, no cron entries and no SUID binaries.
  Installed files: `/usr/local/san/vsp/**` and the two `/usr/bin` symlinks.
- Each release's RPM is attached to a [GitHub Release](https://github.com/visubramaniam/vsptop6-releases/releases)
  with its SHA-256 shown on the page; verify after download with `sha256sum`.

## How the code is checked

Every change to the (private) source repository runs through three GitHub Actions workflows
before it can become a release:

| Workflow | What it does |
|---|---|
| **CI** | builds `vsptop` and `vspdash` on Rocky Linux 9 and runs the test suite against mock collectors and recorded topology output |
| **Code scan** | compiles all vsptop6 code with `-Wall -Wextra -Werror`; **cppcheck** (warning, performance, portability); **gcc -fanalyzer**; **flawfinder** (any level-4+ finding fails the build unless reviewed and annotated in the source); an **AddressSanitizer + UndefinedBehaviorSanitizer** build running the full test suite with leak detection; **shellcheck**; **rpmlint** on the package spec |
| **Release** | builds the RPM on a host that has RMLIB, after CI and the scan have passed |

Design choices that follow from this:

- The optional `raidcom` calls use `fork`/`execv`, not a shell, and the login is answered
  through a pseudo-terminal: the array password is never a command-line argument and is never
  interpreted by `sh`.
- Collectors receive their credentials through a `0600` file in a private per-process
  directory, never on a command line.
- Every memory allocation is checked; the sanitizer run has no findings.
- The original Hitachi collector programs are no longer built or shipped; every collector in
  the package is vsptop6 code and goes through the same gates.

GitHub's own code scanning (CodeQL) is not used because it requires GitHub Advanced Security
on a private repository.

## Reporting a problem

Open an issue on [vsptop6-releases](https://github.com/visubramaniam/vsptop6-releases/issues).
For anything security-sensitive (a way to change array state, credential exposure), do not
post details publicly — open the issue with just a request for contact and it will be followed
up privately.
