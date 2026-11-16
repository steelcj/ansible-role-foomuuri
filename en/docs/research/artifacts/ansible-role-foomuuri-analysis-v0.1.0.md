---
name: ansible-role-foomuuri-analysis
org: universal-cake
lang: en
type: analysis
version: 0.1.0
date: 2026-11-15
---

# ansible-role-foomuuri: Pre-Build Analysis

Version: 0.1.0

Status: Draft

An analysis made before writing `ansible-role-foomuuri`, a role that installs and configures the Foomuuri nftables firewall on several Linux families. It compares Robert de Bock's `ansible-role-firewall` with an existing Foomuuri role published by Tubby1981 (the uploaded files), checks both against Foomuuri itself, and proposes a design that keeps Robert's structure and testing discipline while exposing Foomuuri's full rule language.

## Scope and method

| Item | Value |
|---|---|
| Robert's role | `robertdebock/ansible-role-firewall`, cloned 2026-11-15, last commit `1f32061` (2026-09-22) |
| Robert's EPEL role | `robertdebock/ansible-role-epel`, cloned 2026-11-15 |
| Existing Foomuuri role | Tubby1981's `tubby1981.system.foomuuri`, from the uploaded `README.md`, `meta`, `handlers`, `tasks`, `defaults`, `vars` and `foomuuri.conf.j2` |
| Foomuuri | Source and documentation cloned from `FoobarOy/foomuuri` on 2026-11-15, latest release 0.34 (2026-10-16), `main` at `e5ac100` |
| Live test | The uploaded template was rendered with `ansible-core` for six variable sets, and each result was run through `foomuuri check` from the Foomuuri source tree |
| Packaging status | Distribution archives and package pages, see References |

Everything marked "tested" below was reproduced in that environment. Everything marked "to verify" was not.

## Summary

Robert's role is the right model for structure, platform handling and testing, but it cannot express a Foomuuri-class firewall: it opens and closes ports, and nothing else. Tubby1981's role covers almost all of Foomuuri's configuration sections and is a useful catalogue of what users want to set, but its shipped defaults lock out SSH and outgoing traffic, two of its documented features generate configurations that Foomuuri rejects, a misspelt zone name silently removes rules, and its install logic misses Ubuntu 24.04 and all Red Hat family systems.

The recommendation is to write `ansible-role-foomuuri` in Robert's style (distribution lookup tables in `vars/main.yml`, an `assert.yml`, Molecule across a container matrix, preparation roles outside the role), install from distribution packages wherever they exist (Fedora, EPEL, Debian, Ubuntu 24.04 and later), and pass Foomuuri rules through as strings rather than inventing a second rule language in YAML. Configuration is validated in a staging directory with `foomuuri --set=etc_dir=... check` before it touches `/etc/foomuuri`, and changes are applied with `reload`, never `restart`.

## Robert de Bock's ansible-role-firewall

### What it does

The role removes conflicting firewall packages, installs one backend per family, starts it, and opens or closes a list of ports or service names <a id="cite-debock-2026a"></a>([de Bock, 2026a](#ref-debock-2026a)). The backend is fixed per distribution:

| Family | Backend | Removed as conflicting |
|---|---|---|
| Debian, Ubuntu | `ufw` | `firewalld` |
| Fedora, EL 8 and 9, openSUSE | `firewalld` | `ufw` (not on EL) |
| EL 7 | `firewalld` plus `iptables-services` | `ufw` |
| Alpine | `iptables`, from a fixed template | `firewalld`, `ufw` |

Its only input is `firewall_services`, a list of items with `name` (port number or service name), optional `protocol` (`tcp` or `udp`), optional `rule` (`allow` or `deny`) and optional `state` (`present` or `absent`).

### Patterns worth copying

Distribution lookup tables. `vars/main.yml` holds private dictionaries such as `_firewall_packages` and `_firewall_service`, resolved in order `distribution-major_version`, then `distribution` or `os_family`, then `default`. A new platform is one line in a table, not a new task file.

Input assertions. `tasks/assert.yml` checks every variable's type and allowed values, runs once, and is delegated to `localhost`, so bad input fails before any host is touched.

Container awareness. Every task that would start a service or load rules is skipped when `ansible_connection` is a container connection, so the role can be linted and converged in Docker.

Preparation outside the role. `meta/main.yml` has `dependencies: []`. `robertdebock.bootstrap` is listed in `requirements.yml` and called only from `molecule/default/prepare.yml`. The same convention applies to EPEL: Robert's `epel` role installs `epel-release` on EL 8, 9 and 10 and Amazon Linux 2 <a id="cite-debock-2026b"></a>([de Bock, 2026b](#ref-debock-2026b)), and roles that need EPEL expect it to have run first, which matches how uc-gradus already runs EPEL before this role on Red Hat family systems.

A real test matrix. Molecule runs on Alpine, EL, Debian latest and bookworm, Fedora, and Ubuntu latest and jammy, using his systemd-capable container images.

Template validation. Where the role writes rules itself (Alpine), the template task uses `validate: "iptables-restore --test %s"`, so an invalid file is never installed.

### Consultation finding

Recorded in the format of the Robert de Bock roles CFAP.

| Field | Content |
|---|---|
| Need | Manage a multizone, bidirectional nftables firewall with NAT, iplists, macros and templates, across Debian, Ubuntu, EL, Fedora and Arch |
| Candidate | `robertdebock.firewall` |
| Repository | https://github.com/robertdebock/ansible-role-firewall |
| Can it do it? | No |
| Evidence | The only rule input is a port or service name with protocol and allow or deny. No zones, no outgoing or forwarding policy, no NAT, no source address matching. On Debian it drives `ufw`, on EL `firewalld`, and it removes the other, so it would also fight a Foomuuri install. The iptables template is fixed and cannot be extended through variables. |
| Action | Write our own (`ansible-role-foomuuri`), following his conventions. Do not run `robertdebock.firewall` on the same hosts. |

This also answers the question this project started with: Robert's role cannot be made to produce more complex rule sets through its variables or defaults alone; its template and tasks would have to be rewritten.

## Tubby1981's foomuuri role

### What it gets right

It covers nearly every Foomuuri section: global options, zones, macros (service, address and composite), templates, zone-zone policies, any-zone policies, iplists with global settings, SNAT, DNAT, zonemap, mangle chains, special chains, raw `.nft` files and the connectivity monitor.

It separates role-level and inventory-level data with paired variables (`foomuuri_macros` and `foomuuri_extra_macros`, and so on), so group and host variables can add to a base without repeating it.

Macro output is correct for both styles it accepts. Rendering `tcp: [53]` with `udp: [53]` gives `dns-full tcp 53; udp 53`, which is the form Foomuuri requires when one macro holds two rules (tested).

It merges duplicate zone-zone policies from both lists and removes repeated rules.

### Defects found

| Severity | Finding | Evidence |
|---|---|---|
| Critical | The defaults lock the host out. `foomuuri_zone_policies` contains only `public-localhost { ping; drop log }` and there is no `localhost-public` policy. Foomuuri then adds `reject log` to every unconfigured pair <a id="cite-foobaroy-2026d"></a>([FoobarOy, 2026d](#ref-foobaroy-2026d)), so SSH in, and DNS, package updates and NTP out, are all blocked. The README claims the default includes `ssh`; `defaults/main.yml` does not. | Tested. Generated ruleset: `public-localhost_4` accepts only ICMP echo before the drop; `localhost-public_4` ends in `reject`. |
| High | The documented NAT example fails. `foomuuri_config: {ip_forward: "yes"}` produces `Unknown foomuuri{} option: ip_forward yes`. Foomuuri has no such option; forwarding is a sysctl, and upstream's own sysctl file says to add it in `/etc/sysctl.d/` <a id="cite-foobaroy-2026a"></a>([FoobarOy, 2026a](#ref-foobaroy-2026a)). | Tested, `foomuuri check` fails. |
| High | The monitor feature cannot work. The template writes `target gateway 192.168.1.1 30s` at top level; Foomuuri expects `target name { ... }` blocks <a id="cite-foobaroy-2026e"></a>([FoobarOy, 2026e](#ref-foobaroy-2026e)). | Tested, `Unknown line: target gateway 192.168.1.1 30s`. |
| High | A policy naming an unknown zone is dropped without a word. The template keeps a policy only if both zones exist, so `pubilc` instead of `public` removes the whole policy, and `foomuuri check` still passes. | Tested, case "d". |
| High | Ubuntu 24.04 is sent to a source build. The package path requires Ubuntu 24.10 or later, but Foomuuri is packaged in Ubuntu 23.10 and 24.04 <a id="cite-foobaroy-2026b"></a>([FoobarOy, 2026b](#ref-foobaroy-2026b)). | Read from `install.yml`. |
| High | No Red Hat family support. `foomuuri_required_packages` has a `RedHat` entry, but `tasks/main.yml` fails on anything other than Debian, Ubuntu or Arch, although Foomuuri is packaged in Fedora and EPEL. | Read from `tasks/main.yml`. |
| Medium | Changes are applied with `systemctl restart`. Foomuuri's unit maps stop to `foomuuri stop`, which removes the ruleset, so every configuration change opens a short unfiltered window. `reload` is atomic. | Read from `handlers/main.yml` and upstream `foomuuri.service`. |
| Medium | Validation happens after the file is installed. The handler runs `foomuuri check` on `/etc/foomuuri` once the new file is in place; a failure stops the play but leaves the bad file on disk for the next reboot (where `foomuuri-boot.service` falls back to the last good ruleset, then `foomuuri start` fails). | Read from `configure.yml` and handlers. |
| Medium | Conflicting firewalls are not disabled. Upstream's post-install steps disable `firewalld`, `ufw`, `nftables`, `netfilter-persistent` and others and flush the ruleset <a id="cite-foobaroy-2026b-2"></a>([FoobarOy, 2026b](#ref-foobaroy-2026b)); the role does none of this. | Read from tasks. |
| Medium | Rules are silently reordered. Bare `drop`, `drop log` and `accept` lines are moved to the end of each section, but `reject`, `reject log` or `drop log "message"` are not, so the result depends on spelling. | Read from template. |
| Medium | The source install never upgrades and is not pinned. It downloads whatever GitHub calls "latest", unauthenticated, and `creates: /usr/sbin/foomuuri` means it never runs again. `nftables` is missing from the Debian dependency list used on that path. | Read from `install.yml` and `defaults/main.yml`. |
| Medium | The AUR build is likely to fail. The build directory is created by root, then `makepkg` runs as `ansible_user_id`, which is usually root under `become`, and `makepkg` refuses to run as root. | Read from `install.yml`, to verify on Arch. |
| Low | Debian backports is added on every Debian 12 or later host, including Debian 13 where Foomuuri is in `main`. | Read from `install.yml`. |
| Low | An unrelated gate: the role fails unless a variable named `firewall` equals `foomuuri`. | Read from `tasks/main.yml`. |
| Low | `vars/Archlinux.yml` sets `foomuuri_binary_path: /usr/bin/foomuuri`, but every task and handler hard-codes `/usr/sbin/foomuuri`. The `*_dependencies` lists in both vars files are never used. | Read from files. |
| Low | The README lists commands that do not exist: `foomuuri zones`, `foomuuri version`, `foomuuri restart`, `foomuuri iplist show`. `foomuuri list` is obsolete (now `ruleset list`). | Checked against `foomuuri --help` in 0.34. |
| Low | `meta/main.yml` advertises Debian bullseye and Ubuntu focal; the README says Ubuntu 24.10+ for packages while the metadata says noble. | Read from files. |

In-template mutation of the default dictionaries (`all_any_zone_rules[section].extend(...)`) looked like it might duplicate rules on repeated renders. Rendering twice in one play did not duplicate anything, so it is not listed as a defect, but the new role should still build merged values without mutating inputs.

### Verdict

The role does not meet or beat Robert's approach on robustness, but it beats it by a wide margin on expressiveness. Use it as a catalogue of features and variable shapes, not as a code base.

## Foomuuri facts that shape the design

### Packaging by platform

| Platform | Source of the package | Version seen | Install method for the role |
|---|---|---|---|
| Fedora (all current) | Fedora repository | current | `package` |
| EL 9 and 10 (RHEL, Alma, Rocky, CentOS Stream) | EPEL | 0.28 in stable, 0.30 and 0.31 in testing <a id="cite-fedora-2026"></a>([Fedora Project, 2026](#ref-fedora-2026)) | `package`, EPEL already enabled by `robertdebock.epel` |
| EL 8 | not found | none | Out of scope: Foomuuri needs Python 3.9 or later, EL 8's `/usr/bin/python3` is 3.6. To verify. |
| Debian 13 trixie | `main` | 0.27, 0.31 in trixie-backports <a id="cite-debian-2026"></a>([Debian, 2026](#ref-debian-2026)) | `package` |
| Debian 12 bookworm | `bookworm-backports` | 0.27 | `package` from backports, with `default_release` |
| Ubuntu 24.04 noble and later | `universe` | 0.22 for noble, by archive dates, to verify with `apt-cache policy` <a id="cite-ubuntu-2026"></a>([Ubuntu, 2026](#ref-ubuntu-2026)) | `package` |
| Ubuntu 22.04 jammy | not packaged | none | `source`, pinned |
| Arch Linux | AUR | 0.28 at last check <a id="cite-aur-2026"></a>([Arch User Repository, 2026](#ref-aur-2026)) | `aur` via a non-root build user, or `source` |
| Alpine | not packaged, and no systemd | none | Out of scope for now |

Two consequences. First, the fleet will run anything from 0.22 to 0.34, so the role's own defaults must use only syntax that 0.22 accepts, and anything newer must come from the user's own rule strings. Second, 0.34 and 0.35 change iplist behaviour (`|missing-ok` replaced by `missing_ok=yes`, and `overwrite` defaulting to `no` from 0.35) <a id="cite-foobaroy-2026c"></a>([FoobarOy, 2026c](#ref-foobaroy-2026c)), so the role should record the installed version as a fact and let assertions refuse version-specific settings on older hosts.

### Configuration loading

Foomuuri reads every `*.conf` under `/etc/foomuuri`, alphabetically and including subdirectories, after the shipped files in `/usr/share/foomuuri`, and includes every `*.nft` file as raw nftables <a id="cite-foobaroy-2026f"></a>([FoobarOy, 2026f](#ref-foobaroy-2026f)). This allows the role to write several files (one for globals and zones, one per zone pair group, one per raw `.nft`) and lets an operator drop in a hand-written file without the role removing it, provided the role only manages files it owns.

### Validation and apply

`foomuuri --set=etc_dir=DIR check` validates a whole configuration directory that is not the live one, and the check includes a real `nft --check` of the generated ruleset (tested). It writes `next.fw` into the state directory, so staging checks should also pass `--set=state_dir=` pointing at a scratch directory.

`foomuuri reload` (or `systemctl reload foomuuri`) replaces the ruleset atomically and refreshes iplists. `foomuuri try-reload` adds a confirmation with automatic rollback, but it waits for an interactive answer, so it is not directly usable from Ansible.

`foomuuri-boot.service` loads the last good ruleset (`/var/lib/foomuuri/good.fw`) early in boot, before the configuration is parsed.

### Forwarding and sysctl

The package ships `50-foomuuri.conf` as a sysctl file with ARP and redirect hardening. It does not enable forwarding; upstream asks for a separate `/etc/sysctl.d/` file. A router role setting therefore belongs in `ansible.posix.sysctl`, not in the `foomuuri {}` section.

### firewalld emulation

`foomuuri-firewalld` lets NetworkManager assign interfaces to zones over D-Bus. It conflicts with a real `firewalld`. It is useful on desktops and laptops, not needed on servers with fixed interface names.

## Proposed design for ansible-role-foomuuri

### Principles

Robert's skeleton, Foomuuri's language. Lookup tables, assertions, container awareness and Molecule come from Robert. Rule syntax is Foomuuri's own, passed through as strings, because the Foomuuri rule language already handles every matcher, rate limit, log option and statement, and a YAML re-encoding would always lag behind it.

Fail loudly. Unknown zones, policies with misplaced terminal rules, missing SSH access and unsupported platforms are assertion failures, not silent omissions.

Never install an unchecked configuration. Render to a staging directory, check it, then copy.

Never open the firewall to apply a change. Reload, do not restart.

Distribution packages first. Source builds are a pinned, checksummed fallback.

### Layout

```text
ansible-role-foomuuri/
├── defaults/main.yml
├── vars/main.yml               # _foomuuri_* lookup tables, resolved names
├── meta/main.yml               # platforms, dependencies: []
├── meta/argument_specs.yml     # types and choices for every public variable
├── requirements.yml            # bootstrap, epel, ansible.posix, community.general
├── handlers/main.yml           # Reload foomuuri
├── tasks/
│   ├── main.yml
│   ├── assert.yml
│   ├── install_package.yml
│   ├── install_aur.yml
│   ├── install_source.yml
│   ├── conflicts.yml
│   ├── sysctl.yml
│   └── configure.yml           # stage, check, deploy, enable
├── templates/
│   ├── foomuuri.conf.j2        # globals, zones, macros, templates, iplists, targets
│   └── policies.conf.j2        # zone-zone, any-zone and other rule sections
└── molecule/default/
    ├── prepare.yml             # robertdebock.bootstrap, robertdebock.epel
    ├── converge.yml
    └── verify.yml              # foomuuri check, ruleset contains expected chains
```

### Platform resolution

```yaml
# vars/main.yml
_foomuuri_install_method:
  default: package
  Ubuntu-22: source
  Archlinux: aur

foomuuri_install_method_resolved: >-
  {{ foomuuri_install_method
     if foomuuri_install_method != 'auto'
     else _foomuuri_install_method[ansible_facts['distribution'] ~ '-' ~ ansible_facts['distribution_major_version']]
          | default(_foomuuri_install_method[ansible_facts['distribution']]
          | default(_foomuuri_install_method['default'])) }}

_foomuuri_apt_default_release:
  Debian-12: bookworm-backports

_foomuuri_conflicting_services:
  default: [firewalld, ufw, nftables, netfilter-persistent, iptables, ip6tables, ferm, firehol, shorewall, shorewall6]

_foomuuri_source_requirements:
  Debian: [make, nftables, python3, python3-requests, python3-dbus, python3-gi, python3-systemd]
  RedHat: [make, nftables, python3, python3-requests, python3-dbus, python3-gobject, python3-systemd]
  Archlinux: [make, nftables, python, python-requests, python-dbus, python-gobject, python-systemd]
```

The binary path is resolved once (`/usr/sbin/foomuuri` for packages and `make install`, which defaults `BINDIR` to `/usr/sbin`) and used everywhere, so Arch and source paths cannot drift.

### EPEL on Red Hat family systems

The role does not install EPEL. On `os_family == "RedHat"` (excluding Fedora), the first install task checks that the `foomuuri` package is available (`dnf list foomuuri`) and, if not, fails with a message that names `robertdebock.epel` and the CRB repository. Molecule's `prepare.yml` runs `robertdebock.epel`, which mirrors how uc-gradus runs it before this role in production.

### Variable model

```yaml
# defaults/main.yml (abridged)

foomuuri_install_method: auto          # auto, package, aur, source
foomuuri_source_version: "0.34"
foomuuri_source_checksum: "sha256:..."
foomuuri_firewalld: false              # install foomuuri-firewalld
foomuuri_disable_conflicting: true
foomuuri_ip_forward: false             # writes /etc/sysctl.d/60-foomuuri-forward.conf
foomuuri_require_ssh: true             # assert that some *-localhost policy allows ssh

# foomuuri {} section, keys as Foomuuri spells them
foomuuri_config: {}

# zone {} section; localhost is added by the role
foomuuri_zones:
  public:
    - "{{ ansible_facts['default_ipv4']['interface'] | default('eth0') }}"

# macro {} section; value is a Foomuuri string, or a protocol mapping
foomuuri_macros: {}
#  admin-nets: "192.0.2.0/24 2001:db8::/32"
#  dns-full: {tcp: [53], udp: [53]}      # renders "tcp 53; udp 53"

# template NAME {} sections
foomuuri_templates: {}

# Every rule section, keyed by its Foomuuri name:
# zone-zone, any-zone, zone-any, any-any, snat, dnat, zonemap,
# prerouting, postrouting, invalid, rpfilter, smurfs, hook, ...
foomuuri_rules:
  public-localhost:
    - dhcp-client
    - dhcpv6-client
    - ping
    - ssh
    - drop log
  localhost-public:
    - accept

# iplist {} section
foomuuri_iplists: {}
foomuuri_iplist_settings: {}

# target NAME {} and group NAME {} sections, lines as written in Foomuuri
foomuuri_targets: {}
foomuuri_groups: {}

# Extra files under /etc/foomuuri, owned by the role (.conf or .nft)
foomuuri_files: []
#  - name: 90-docker.nft
#    content: |
#      ...
#    state: present
```

Why one `foomuuri_rules` dictionary instead of a list of `from_zone`, `to_zone`, `rules` items plus separate SNAT, DNAT, mangle and special-chain variables: every one of those Foomuuri sections has the same shape (a name, then lines of rules), and the dictionary key is exactly the section name written in the file. A new Foomuuri section needs no role change. Duplicate keys across inventory layers are merged by concatenating their lists and removing exact duplicates while keeping first-seen order.

The defaults match upstream's own host firewall example: SSH, DHCP and ping in, everything out. A host with only the defaults is reachable and can update.

### Layering across inventory

Tubby1981's `foomuuri_x` plus `foomuuri_extra_x` pairs allow exactly two layers. To allow `group_vars/all`, a role group and `host_vars` to each contribute without overwriting each other, the role also merges every variable whose name starts with the main name and a double underscore:

```yaml
# group_vars/all.yml
foomuuri_rules__base:
  public-localhost: [ssh saddr admin-nets]

# group_vars/webservers.yml
foomuuri_rules__web:
  public-localhost: [http, https]
```

```yaml
# vars/main.yml
_foomuuri_rules_layers: "{{ query('ansible.builtin.varnames', '^foomuuri_rules__.+') | sort | map('extract', vars) | list }}"
```

Layers are applied in name order, so `__10_base` comes before `__20_web`. The same mechanism is offered for `foomuuri_macros`, `foomuuri_zones`, `foomuuri_iplists` and `foomuuri_files`. Whether to keep `_extra_` names as well, for users coming from Tubby1981's role, is an open question below.

### Ordering and terminal rules

The role never reorders rules. Instead, `assert.yml` fails if a bare terminal rule (`accept`, `drop`, `reject`, with or without `log` and a message) appears anywhere but last in a merged section. Because layers concatenate, a base layer must not end with `drop log`; the role's documentation states that terminal rules belong in a dedicated last layer, or are left to Foomuuri's automatic final rule.

### Assertions

`assert.yml` plus `meta/argument_specs.yml` check:

- every zone named in a zone-zone key of `foomuuri_rules` exists in `foomuuri_zones` or is `localhost` or `any`
- each zone has a list of interfaces (empty is allowed, for NetworkManager)
- `foomuuri_config` keys are in the set Foomuuri documents, so `ip_forward` fails with a pointer to `foomuuri_ip_forward`
- when `foomuuri_require_ssh` is true, at least one `*-localhost` or `any-localhost` section contains a rule beginning with `ssh`, or a `tcp 22` rule
- the platform is in the supported table
- iplist settings such as `missing_ok` or `overwrite` are refused when the installed Foomuuri is older than 0.34

### Validate, then apply

```yaml
# tasks/configure.yml, outline
- name: Create staging directories
  ansible.builtin.tempfile: {state: directory, suffix: foomuuri}
  register: _foomuuri_stage            # also a state directory for the check

- name: Render configuration into staging
  ansible.builtin.template: {src: "{{ item }}.j2", dest: "{{ _foomuuri_stage.path }}/etc/{{ item }}", mode: "0644"}
  loop: [foomuuri.conf, policies.conf]

- name: Copy unmanaged operator files into staging, so the check sees the full set
  # find *.conf and *.nft in /etc/foomuuri not owned by the role, copy with remote_src

- name: Check staged configuration
  ansible.builtin.command:
    argv:
      - "{{ foomuuri_binary }}"
      - "--set=etc_dir={{ _foomuuri_stage.path }}/etc"
      - "--set=state_dir={{ _foomuuri_stage.path }}/state"
      - check
  changed_when: false

- name: Deploy configuration
  ansible.builtin.template: ...      # same templates, now to /etc/foomuuri
  notify: Reload foomuuri

- name: Remove staging directory
  ansible.builtin.file: {path: "{{ _foomuuri_stage.path }}", state: absent}
```

The handler is `ansible.builtin.systemd: {name: foomuuri, state: reloaded}`. The first run, when the service is not yet active, uses `state: started` instead. Both are skipped in containers unless `foomuuri_apply_in_container` is true, following Robert's pattern; the check task always runs, so containers still test the generated configuration.

An optional safety net for remote changes, to decide: before reloading, schedule a one-shot `systemd-run --on-active=120 foomuuri start` with the previous configuration restored, and cancel it from a second task that runs only if Ansible can still reach the host after the reload. This gives `try-reload` behaviour without an interactive prompt.

### Conflicting firewalls

When `foomuuri_disable_conflicting` is true, `conflicts.yml` stops, disables and masks every service in `_foomuuri_conflicting_services` that exists on the host, before the first start. Packages are left installed, unlike Robert's role, because removing `iptables` or `nftables` packages can remove tools other software depends on. `firewalld` is skipped from the list only when it is not installed; if `foomuuri_firewalld` is true and `firewalld` is installed, the assertion fails.

### Forwarding

`foomuuri_ip_forward: true` writes `net.ipv4.conf.all.forwarding = 1` and `net.ipv6.conf.all.forwarding = 1` to `/etc/sysctl.d/60-foomuuri-forward.conf` with `ansible.posix.sysctl`, which is what Foomuuri's own sysctl file asks for.

### Testing

Molecule with Robert's images: Debian 13 and 12, Ubuntu 24.04 and 22.04, Fedora latest, EL 9 and 10 (with EPEL from `prepare.yml`), Arch latest. `converge.yml` uses a two-zone configuration with a macro, a template, an iplist with static addresses, SNAT and DNAT. `verify.yml` runs `foomuuri check` and, where nftables works in the container, `foomuuri ruleset list public-localhost` and asserts the expected rules. A second scenario renders the role's defaults only and asserts that SSH is accepted and `localhost-public` accepts, guarding against the lockout found above. Final acceptance is on a VM or bare metal, since containers share the host's kernel netfilter.

## Side by side

| Concern | robertdebock.firewall | tubby1981 foomuuri | Proposed ansible-role-foomuuri |
|---|---|---|---|
| Rule expressiveness | Ports and service names only | Most Foomuuri sections, structured | Every Foomuuri section, rules as Foomuuri strings |
| Platforms | Alpine, Debian, Ubuntu, Fedora, EL 9 | Debian, Ubuntu, Arch (EL declared, not enabled) | Debian 12 and 13, Ubuntu 22.04 and 24.04+, Fedora, EL 9 and 10, Arch |
| Platform logic | Lookup tables in `vars/` | Inline `set_fact` expression | Lookup tables in `vars/` |
| Input checks | `assert.yml` | None | `assert.yml` and `argument_specs.yml` |
| Safe defaults | SSH open | SSH and outgoing blocked | SSH, DHCP, ping in; all out |
| Validation | `iptables-restore --test` on Alpine | After install, in a handler | Before install, in staging |
| Apply | Service reload | Service restart | Service reload |
| Conflicting firewalls | Removes packages | Ignored | Stops, disables, masks services |
| Inventory layering | Single list | Two layers (`_extra_`) | Any number (`__` suffix) |
| Tests | Molecule, seven images | Manual playbook | Molecule, eight images, plus VM |

## Open questions and unclear points

| Question | Options or recommendation |
|---|---|
| Keep Tubby1981's variable names (`foomuuri_zone_policies`, `foomuuri_extra_*`) for compatibility? | **Option A: no, clean model only**. Option B: accept them and translate, with a deprecation warning. Recommended: A, since the role has no users yet and the old shapes encode the defects above. |
| One generated file, or one per section group? | Option A: `foomuuri.conf` plus `policies.conf`. **Option B: one file per zone pair.** Recommended: A, split only when a config passes about 200 lines, as upstream suggests. |
| Add the remote rollback timer? | Option A: yes, off by default. Option B: rely on `foomuuri-boot` and console access. **Recommended: A, since uc-gradus manages nodes remotely.** |
| Should `foomuuri_disable_conflicting` remove packages, as Robert does? | **Recommended**: no, mask services only. |
| EL 8 support | **Recommended**: out of scope unless a Foomuuri package for EL 8 is found. To verify. |
| Alpine support | Foomuuri has no OpenRC service. **Recommended: out of scope.** |
| License | Robert uses Apache-2.0, Tubby1981 MIT. Owner to decide. |
| Publish under which Galaxy namespace and as a standalone role or in a collection? | Owner to decide. Robert's convention is a standalone role per repository. |
| Verify Ubuntu 24.04's Foomuuri version and that the role's defaults pass `check` on it | Must be tested in Molecule before the first release. |
| Verify the AUR package is current, or prefer `source` on Arch | Must be checked; the AUR page showed 0.28 while upstream is at 0.34. |

## References

<a id="ref-aur-2026"></a>Arch User Repository. (2026). *foomuuri* [Package details]. https://aur.archlinux.org/packages/foomuuri
[Return to citation](#cite-aur-2026)

<a id="ref-debian-2026"></a>Debian. (2026). *Package: foomuuri* [Source package listing]. https://sources.debian.org/patches/foomuuri
[Return to citation](#cite-debian-2026)

<a id="ref-debock-2026a"></a>de Bock, R. (2026a). *ansible-role-firewall* (Commit 1f32061) [Computer software]. GitHub. https://github.com/robertdebock/ansible-role-firewall
[Return to citation](#cite-debock-2026a)

<a id="ref-debock-2026b"></a>de Bock, R. (2026b). *ansible-role-epel* [Computer software]. GitHub. https://github.com/robertdebock/ansible-role-epel
[Return to citation](#cite-debock-2026b)

<a id="ref-fedora-2026"></a>Fedora Project. (2026). *EPEL testing 10, Everything, x86_64, packages "f"* [Repository listing]. https://mirror.arizona.edu/fedora-epel/testing/10/Everything/x86_64/Packages/f/
[Return to citation](#cite-fedora-2026)

<a id="ref-foobaroy-2026a"></a>FoobarOy. (2026a). *50-foomuuri.conf* [Configuration file, Foomuuri 0.34]. GitHub. https://github.com/FoobarOy/foomuuri/blob/main/etc/50-foomuuri.conf
[Return to citation](#cite-foobaroy-2026a)

<a id="ref-foobaroy-2026b"></a>FoobarOy. (2026b). *Installation*. Foomuuri documentation. https://foomuuri.foobar.fi/latest/install/
[Return to citation](#cite-foobaroy-2026b)
[Return to second citation](#cite-foobaroy-2026b-2)

<a id="ref-foobaroy-2026c"></a>FoobarOy. (2026c). *Changelog* [Foomuuri 0.34 and 0.35 entries]. GitHub. https://github.com/FoobarOy/foomuuri/blob/main/CHANGELOG.md
[Return to citation](#cite-foobaroy-2026c)

<a id="ref-foobaroy-2026d"></a>FoobarOy. (2026d). *zone-zone*. Foomuuri documentation. https://foomuuri.foobar.fi/latest/config/section/zonezone/
[Return to citation](#cite-foobaroy-2026d)

<a id="ref-foobaroy-2026e"></a>FoobarOy. (2026e). *Monitor*. Foomuuri documentation. https://foomuuri.foobar.fi/latest/tool/monitor/
[Return to citation](#cite-foobaroy-2026e)

<a id="ref-foobaroy-2026f"></a>FoobarOy. (2026f). *Basics*. Foomuuri documentation. https://foomuuri.foobar.fi/latest/config/basic/
[Return to citation](#cite-foobaroy-2026f)

<a id="ref-ubuntu-2026"></a>Ubuntu. (2026). *Index of /ubuntu/pool/universe/f/foomuuri* [Archive mirror listing]. https://mirror.seas.harvard.edu/ubuntu/pool/universe/f/foomuuri/
[Return to citation](#cite-ubuntu-2026)

## Changelog

| Version | Status | Notes |
|---|---|---|
| 0.1.0 | Draft | First analysis: Robert's firewall role reviewed and consultation finding recorded, Tubby1981's role reviewed with six rendered configurations checked by `foomuuri check`, Foomuuri packaging and behaviour summarised, design proposed for `ansible-role-foomuuri`, open questions listed |
