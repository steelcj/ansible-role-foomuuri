# [Ansible role foomuuri](#ansible-role-foomuuri)

Build stable nftables firewalls with Foomuuri, from one or more additive passes.

|GitHub|Version|
|------|-------|
|[steelcj/ansible-role-foomuuri](https://github.com/steelcj/ansible-role-foomuuri)|0.1.0, first iteration, untested|

## [Example Playbook](#example-playbook)

This example is taken from [`molecule/default/converge.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/molecule/default/converge.yml).

```yaml
---
- name: Converge
  hosts: all
  become: true
  gather_facts: true

  roles:
    # Step one: the base fragment only. It allows SSH on port 22, as a raw
    # node does at handoff.
    - role: steelcj.foomuuri
      foomuuri_fragment: 10-base
```

The machine needs to be prepared. In testing this is done using [`molecule/default/prepare.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/molecule/default/prepare.yml):

```yaml
---
- name: Prepare
  hosts: all
  become: true
  gather_facts: false

  tasks:
    - name: Install Python, which Incus images may not include
      ansible.builtin.raw: test -x /usr/bin/python3 || (apt-get update && apt-get install -y python3)
      register: _python
      changed_when: "'Setting up python3' in _python.stdout"

    - name: Install ufw, so the role has a conflicting firewall to disable
      ansible.builtin.apt:
        name: ufw
        state: present
        update_cache: true

    # The role has to take over from a firewall that is running, not only
    # installed. SSH is allowed first, as on a raw node at handoff.
    - name: Allow SSH through ufw
      community.general.ufw:
        rule: allow
        port: "22"
        proto: tcp

    - name: Enable ufw, so the role takes over from an active firewall
      community.general.ufw:
        state: enabled
```

## [How it works](#how-it-works)

Each run of the role writes one fragment, a file in `/etc/foomuuri/ansible/` named by `foomuuri_fragment`. Foomuuri reads configuration files in name order and appends the rules of a section that appears in more than one file, so a later run adds to earlier ones.

Every run then:

1. Copies the live configuration, with this run's changes, into a staging directory.
2. Checks the whole staged configuration with `foomuuri check`, which includes an nftables syntax check.
3. Writes the fragment to `/etc/foomuuri` only if the check passed.
4. Reloads Foomuuri, which replaces its `inet foomuuri` table in one step. Tables owned by other software, such as Incus, are not touched. The role never restarts Foomuuri, because a restart leaves the host unfiltered for a moment.

### [Fragments](#fragments)

- **Base fragment.** `foomuuri_zones`, `foomuuri_config` and `foomuuri_base_rules` are written only to the fragment named in `foomuuri_base_fragment` (default `10-base`). Other fragments ignore them, so they can be set in inventory for all hosts.
- **Declarative per fragment.** Each run writes its fragment whole, so removing a rule from that run's variables removes it from the host. Other fragments are left alone.
- **Removing a fragment.** Run the role with `foomuuri_fragment_state: absent`.
- **Terminal rules.** A bare `accept`, `drop` or `reject`, with or without `log`, may only be the last rule of a section, and closes that section. The role refuses a run that would add rules after a section another fragment has closed, because those rules would never be reached. Without a terminal rule, Foomuuri adds its own: `drop log`, or `reject log` for sections leaving `localhost`.
- **SSH.** With `foomuuri_require_ssh: true`, the role refuses any run after which no fragment allows SSH towards `localhost`.

### [Purge](#purge)

| `foomuuri_purge` | Removes |
|------------------|---------|
| `unmanaged` (default) | Every `.conf` or `.nft` file under `/etc/foomuuri` that the role did not write |
| `fragments` | Also every managed fragment other than this run's and those in `foomuuri_keep_fragments`. Use only when one run declares the whole firewall. |
| `none` | Nothing |

Purge only affects files in `/etc/foomuuri`. It never touches nftables tables other than Foomuuri's.

### [Inventory layers](#inventory-layers)

Variables named `foomuuri_rules__<layer>` are merged after `foomuuri_rules`, in name order, into the fragment being written:

```yaml
# group_vars/all.yml
foomuuri_rules__10_base:
  public-localhost:
    - ping

# group_vars/webservers.yml
foomuuri_rules__20_web:
  public-localhost:
    - http
    - https
```

### [D-Bus handler](#d-bus-handler)

Foomuuri starts a D-Bus handler, `foomuuri-dbus.service`, so that NetworkManager can move interfaces between zones. Hosts whose zones are set in `foomuuri_zones` do not need it, so with `foomuuri_dbus: false` (the default) the role stops and masks it before Foomuuri starts.

Foomuuri older than 0.31 lets any local user change zones through this handler (CVE-2025-67603, CVE-2025-67858) unless the distribution has patched its package. Ubuntu 24.04 has the patch only through Ubuntu Pro. Check the installed package before setting `foomuuri_dbus: true`.

## [Role Variables](#role-variables)

The default values for the variables are set in [`defaults/main.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/defaults/main.yml):

```yaml
---
# defaults file for foomuuri

foomuuri_fragment: 10-base
foomuuri_fragment_state: present
foomuuri_base_fragment: 10-base
foomuuri_purge: unmanaged
foomuuri_keep_fragments: []
foomuuri_purge_exclude: []
foomuuri_disable_conflicting: true
foomuuri_dbus: false
foomuuri_require_ssh: true
foomuuri_minimum_version: "0.22"
foomuuri_skip_connections:
  - container
  - docker
  - community.docker.docker
foomuuri_config: {}
foomuuri_zones:
  localhost: []
  public:
    - "*"
foomuuri_base_rules:
  public-localhost:
    - dhcp-client
    - dhcpv6-client
    - ping
    - ssh
  localhost-public:
    - accept
foomuuri_rules: {}
foomuuri_macros: {}
foomuuri_iplists: {}
foomuuri_templates: {}
foomuuri_files: []
```

See [`defaults/main.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/defaults/main.yml) for a description of each variable, and [`meta/argument_specs.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/meta/argument_specs.yml) for types.

## [Requirements](#requirements)

- pip packages listed in [requirements.txt](https://github.com/steelcj/ansible-role-foomuuri/blob/main/requirements.txt).
- Collections listed in [requirements.yml](https://github.com/steelcj/ansible-role-foomuuri/blob/main/requirements.yml), for testing with Incus.
- Foomuuri must be available from the distribution's package repositories. On Ubuntu and Linux Mint it is in `universe`.

## [Context](#context)

This role follows the structure and conventions of Robert de Bock's roles, such as [robertdebock.firewall](https://github.com/robertdebock/ansible-role-firewall), but uses [Foomuuri](https://github.com/FoobarOy/foomuuri) to manage nftables on every platform instead of ufw or firewalld. On each host it stops, disables and masks ufw, firewalld, nftables and other firewall services, and runs `ufw disable` first so that ufw's rules leave the kernel. It does this only after Foomuuri's own table is loaded, so the host is never left without a firewall, and afterwards it reloads Foomuuri if one of those services removed its table.

## [Compatibility](#compatibility)

This role is written for:

|Distribution|Version|
|------------|-------|
|Debian|13|
|Ubuntu|24.04|
|Linux Mint|22|

Tests run in Incus system containers and VMs, see [`molecule/default/molecule.yml`](https://github.com/steelcj/ansible-role-foomuuri/blob/main/molecule/default/molecule.yml). This first iteration has not been tested yet.

The minimum version of Ansible required is 2.15.

If you find issues, please register them on [GitHub](https://github.com/steelcj/ansible-role-foomuuri/issues).

## [License](#license)

[Apache-2.0](https://github.com/steelcj/ansible-role-foomuuri/blob/main/LICENSE).

## [Author Information](#author-information)

[steelcj](https://github.com/steelcj)
