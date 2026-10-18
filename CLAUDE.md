# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The generic layout shared by every formula (`map.jinja` merge chain, TOFS,
`package/config/service` split, Kitchen, CI, pre-commit) is described in
`../CLAUDE.md`. This file only covers what is specific to `ufw-formula`.

## Commands

```bash
bundle install
bin/kitchen list                                  # 6 instances: 2 suites x 3 platforms
bin/kitchen test default-debian-13-3008-2         # one instance, end to end
bin/kitchen converge without-ipv6-debian-11-3006-27
bin/kitchen verify   without-ipv6-debian-11-3006-27
pre-commit run --all-files
```

There is no test runner for the Python in `_modules/` and `_states/`, and no
pre-commit hook lints it: Kitchen is the only thing that executes it.

## Architecture

### Rules go through a custom `ufw` state, not Salt core

`_states/ufw.py` provides the `ufw.allow` / `ufw.allowed` / `ufw.limit` /
`ufw.deny` / `ufw.enabled` states, backed by `_modules/ufw.py`. Each rule state
builds a `ufw ...` command line (`_as_rule`), runs it, and derives the state
result from ufw's own output:

- `Skipping` → unchanged; `Rule(s) added|inserted|updated` → changed; anything
  else → failure with that line as the comment. Idempotence therefore rests
  entirely on ufw recognising an identical rule.
- `deny` is issued as `insert 1 deny` so denies take precedence over allows,
  retried as a plain `deny` when the ruleset is still empty
  (`Invalid position '1'`).
- `app: '*'` and `to_port: '*'` mean "any" and are dropped from the command.
- `from_addr` (and `from_port`) are resolved with `socket.gethostbyname` on
  the minion unless they start with a digit or `:`.
- Rules are additive only: removing an entry from the pillar never deletes the
  rule from the host.
- `default_incoming` / `default_outgoing` states exist but no SLS uses them;
  default policies come from the `/etc/default/ufw` template instead.

### Pillar → states

The four rule sections are each rendered by one SLS under `ufw/config/`:

| Pillar key     | SLS                | State used                         |
|----------------|--------------------|------------------------------------|
| `services`     | `services.sls`     | `ufw.allow` / `ufw.limit` / `ufw.deny` |
| `applications` | `applications.sls` | `ufw.allow` / `ufw.limit` / `ufw.deny` with `app` |
| `interfaces`   | `interfaces.sls`   | `ufw.allowed` with `interface`     |
| `open`         | `open.sls`         | `ufw.allowed` with `from_addr`     |

The method is chosen per entry: `deny: true` wins over `limit: true`, otherwise
`allow`. In `services`, the pillar key is the destination port unless `to_port`
is set; `from_addr` may be a string or a list, and expands into one state per
address. State IDs are `ufw-svc-<method>-<key>-<from_addr>` and
`ufw-app-<method>-<app>[-<from_addr>]`, so two entries that need the same key
must be disambiguated through the key itself (pillar.example's `'*/multiple'`
with `to_port: '*'`).

`comment` is skipped on `osfinger` `Debian-8` and `CentOS-6`, and is passed
wrapped in double quotes because it ends up on a shell command line.

### The `enabled` switch

`ufw:enabled` (default `false`) gates the whole service side:

- `service/enable.sls`: `ufw --force enable` and `ufw logging <level>`;
- `service/reload.sls`: the `reload-ufw` `cmd.wait`, which every rule state
  `listen_in`s only when enabled;
- `service/running.sls`: `service.running` when enabled, `service.dead`
  otherwise.

Rules are added either way — ufw accepts rules while inactive.

### Files

`config/file.sls` renders `/etc/default/ufw` (from `settings`) and
`/etc/ufw/sysctl.conf` (from `sysctl`) through TOFS. `defaults.yaml` sets both
dicts to `{}`: the effective defaults are the `.get(..., default)` calls at the
top of `ufw/files/default/*.tmpl.jinja`, not the map files.

`/etc/ufw/applications.d` is filled by a `file.recurse` from
`salt://ufw/files/applications.d` (literal path, not TOFS, `clean: False`).
Adding an application profile means adding a file there; the INI section name
(e.g. `[Saltmaster]`) is the name to use under `applications`.

## Tests

- Both suites converge `test/salt/pillar/default.sls`; `without-ipv6` layers
  `test/salt/pillar/without-ipv6.sls` (`settings:ipv6: false`) on top.
- Both suites run the same controls (`test/integration/default`), which check
  `ufw status` output per rule. Nothing asserts `IPV6=no` for the
  `without-ipv6` suite.
- There are no `_mapdata` reference dumps in this fork: `ufw._mapdata` is in
  `state_top` but its output is not compared.
