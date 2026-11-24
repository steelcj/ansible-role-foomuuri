---
title: Foomuuri and cloud-init Form a systemd Ordering Cycle
description: Why a Debian host with cloud-init never finishes booting once Foomuuri is installed, how to confirm it, the workarounds, and what remains for upstream.
<<<<<<< HEAD
version: 0.1.0
=======
version: 0.1.1
>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)
status: provisional
date: 2026-11-23
language: en-CA
---

# Foomuuri and cloud-init Form a systemd Ordering Cycle

On a Debian host with cloud-init, installing Foomuuri creates an ordering cycle between Foomuuri's units and cloud-init's network stage. At the next boot, systemd breaks the cycle by dropping one of the jobs in it. The host stays in `starting` for good, and anything that waits for it to finish booting hangs.

The cycle comes from the Foomuuri package's unit files, not from this role, so it affects any host where the two meet. The role installs those units unchanged.

## Symptom

- The first run of the role, before any reboot, succeeds.
- After the next reboot, `systemctl is-system-running` reports `starting` and never changes, and `systemctl is-system-running --wait` never returns. In an Ansible build this shows as a hang in the first task that waits for boot.
- `cloud-init status` waits forever.
- Network and SSH still come up, from the configuration cloud-init wrote at first boot, so the host looks healthy from outside.

## How to confirm it

On the host:

```sh
<<<<<<< HEAD
journalctl -b | grep -i 'ordering cycle'
=======
sudo journalctl -b SYSLOG_IDENTIFIER=systemd | grep -i 'ordering cycle'
>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)
systemctl list-jobs
systemctl is-system-running
```

The journal shows lines like `Found ordering cycle on ...` and `Job cloud-init-network.service/start deleted to break ordering cycle`. `systemctl list-jobs` shows jobs still waiting.

<<<<<<< HEAD
=======
Read the journal as root. An ordinary user only sees their own messages, so without `sudo` the search finds nothing even when there is a cycle. Filtering on `SYSLOG_IDENTIFIER=systemd` keeps out logged commands that contain the search text, such as an Ansible task that runs the same search. Without the filter they can make a count of matching lines non-zero on a host with no cycle.

>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)
To see the ordering without rebooting:

```sh
systemd-analyze verify default.target 2>&1 | grep -i cycle
systemctl cat foomuuri.service foomuuri-boot.service cloud-init-network.service
```

## Cause

The unit files involved, as read from upstream on 2026-11-23:

| Unit | Ordering | Source |
| :-- | :-- | :-- |
| `foomuuri-boot.service` | `After=local-fs.target`, and systemd's default dependencies, which add `After=sysinit.target` | Foomuuri `systemd/foomuuri-boot.service` |
| `foomuuri.service` | `After=foomuuri-boot.service`, `Before=network-pre.target`, and the default dependencies, so also `After=sysinit.target` | Foomuuri `systemd/foomuuri.service` |
| `systemd-networkd.service` | `After=network-pre.target` | systemd |
| `systemd-networkd-wait-online.service` | `After=systemd-networkd.service` | systemd |
| `cloud-init-network.service` | `DefaultDependencies=no`, `After=systemd-networkd-wait-online.service` and `After=networking.service`, `Before=sysinit.target` on the Debian variant | cloud-init `systemd/cloud-init-network.service.tmpl` |

Following the `After=` links from cloud-init's network stage leads back to itself:

```text
cloud-init-network.service
  after systemd-networkd-wait-online.service
  after systemd-networkd.service
  after network-pre.target
  after foomuuri.service
  after foomuuri-boot.service
  after sysinit.target
  after cloud-init-network.service
```

The conflict is between two promises. Foomuuri promises to load the firewall before the network comes up, `Before=network-pre.target`, but keeps systemd's default dependencies, which put it after `sysinit.target`. On the Debian variant, cloud-init promises to finish its network stage, which runs after the network is up, before `sysinit.target`. Both cannot hold.

On a host using ifupdown instead of systemd-networkd, the same cycle should form through `networking.service`, which is also ordered after `network-pre.target`. This has not been seen on a host.

### Why it matters beyond the hang

When systemd finds an ordering cycle, it deletes one job in the cycle to break it. On the host where this was found, it deleted `cloud-init-network.service`. Nothing in the unit files guarantees it will always choose that job. If it chooses `foomuuri.service` or `foomuuri-boot.service` instead, the host comes up with no firewall. Treat the cycle as a firewall problem, not only a boot delay.

## Where it happens

| Host | Status |
| :-- | :-- |
<<<<<<< HEAD
| Debian 13 VPS with cloud-init 25 and systemd-networkd | Seen: the host stays in `starting` after the first reboot |
=======
| Debian 13 VPS with cloud-init 25 and systemd-networkd | Seen: the host stays in `starting` after the first reboot. With cloud-init disabled before the reboot, no cycle forms |
>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)
| Debian with ifupdown and cloud-init | Expected from the unit files, not yet seen |
| Ubuntu with cloud-init | Probably not affected: on the Ubuntu variant, cloud-init's network stage is not ordered after `systemd-networkd-wait-online.service`, and `networking.service` is usually absent. Not tested |
| Incus `images:debian/13`, as in the Molecule scenario | Not affected: the image has no cloud-init, which is why the scenario never showed the cycle |
| Debian installed without cloud-init | Not affected |

The units on a host come from its distribution's packages, which may be older than upstream. Check them on the host with `systemctl cat` before relying on the table above.

## Workarounds

### Disable cloud-init after first boot

Create `/etc/cloud/cloud-init.disabled`. cloud-init's systemd generator then leaves its units out of the boot, so the cycle never forms. The network configuration cloud-init wrote at first boot stays in place, and cloud-init stays installed, so removing the file turns it back on.

The cost: cloud-init no longer applies changes the provider makes later, such as a new address or a change to the network after a resize. On a host managed by Ansible from then on, that is usually acceptable.

<<<<<<< HEAD
This is what uc-node does for hosts on its ssh platform.
=======
This is what uc-node does for hosts on its ssh platform. On 2026-11-24, uc-node adopted a Debian 13 VPS that had Foomuuri installed but had not rebooted since. The build disabled cloud-init before rebooting, the host reached `running`, and systemd's journal had no ordering cycle.
>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)

### Remove cloud-init

The same effect as disabling it, but it cannot be turned back on without reinstalling, and it removes software the provider installed. Disabling is preferred.

### Change the boot ordering

A systemd drop-in can remove one side of the conflict, for example by giving Foomuuri's units `DefaultDependencies=no` with explicit ordering of their own. This keeps cloud-init running at every boot, but it changes when the firewall starts relative to the rest of early boot, and Foomuuri may depend on things that run during `sysinit.target`, such as its tmpfiles entry. Until upstream decides how its units should be ordered, a local drop-in is riskier than disabling cloud-init.

## For upstream

The cycle should be raised with Foomuuri. Questions for that report:

- Can `foomuuri.service` and `foomuuri-boot.service` run with `DefaultDependencies=no`, as early-boot firewall units commonly do, with explicit ordering for what they need from `sysinit.target`?
- If not, is `Before=network-pre.target` compatible with cloud-init's `Before=sysinit.target` at all, or should the documentation say to disable one of them?

Status: not yet reported.

## What the role could do

None of these are done yet. They are recorded in the ROADMAP once chosen.

- Detect an active cloud-init, `/etc/cloud/cloud-init.disabled` absent and the `cloud-init` package installed, and warn, or fail unless the caller says how to handle it.
- Offer an opt-in variable that disables cloud-init, leaving the choice with the caller.
- Add a Molecule scenario on an image with cloud-init, such as `images:debian/13/cloud`, that reboots and checks that the journal has no ordering cycle.

## Changelog

| Version | Date | Change |
| :-- | :-- | :-- |
| 0.1.0 | 2026-11-23 | First version, from the first build of a uc-node raw node on a Debian 13 VPS |
<<<<<<< HEAD
=======
| 0.1.1 | 2026-11-24 | Read the journal as root, filtered on systemd's messages. Record that disabling cloud-init before the reboot worked on a real VPS |
>>>>>>> bea325f (added en/docs/known-issues/cloud-init-ordering-cycle.md)
