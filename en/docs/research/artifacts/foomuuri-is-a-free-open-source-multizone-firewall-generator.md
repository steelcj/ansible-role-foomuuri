**Foomuuri is a free, open-source, multizone firewall generator that works well for managing native `nftables` rulesets on Ubuntu, though its smaller user base requires staying on top of security updates****.** Created in 2023 by Finnish developer Kim B. Heino, it is actively maintained and included in the official repositories for Ubuntu 24.04 LTS and newer. [[1](https://github.com/FoobarOy/foomuuri), [2](https://www.linuxlinks.com/foomuuri-multizone-bidirectional-nftables-firewall/), [3](https://ubuntu.com/security/notices/USN-8326-1), [4](https://linux-ipsec.org/slides/2025/soini-handling-ipsec-with-foomuuri.pdf), [5](https://manpages.ubuntu.com/manpages/noble/man8/foomuuri.8.html)]

https://foomuuri.foobar.fi/latest/

https://github.com/lanefu/clammy-ng

https://github.com/lanefu/ansible-collection-clammy

https://github.com/topics/foomuuri

https://github.com/tubby1981/ansible.git

https://galaxy.ansible.com/ui/repo/published/tubby1981/system/

https://github.com/FoobarOy/foomuuri/discussions/12

https://github.com/lanefu/ansible-collection-clammy/tree/main/roles/foomuuri

Ansible Role

https://github.com/mrlesmithjr



Overview & Open-Source Status

- **License and Codebase.** Foomuuri is licensed under the **GNU General Public License v2.0 (GPL-2.0)**. Its source code is publicly hosted on the [Foomuuri GitHub repository](https://github.com/FoobarOy/foomuuri) and written primarily in Python. [[1](https://github.com/FoobarOy/foomuuri), [2](https://www.linuxlinks.com/foomuuri-multizone-bidirectional-nftables-firewall/), [3](https://linux-ipsec.org/slides/2025/soini-handling-ipsec-with-foomuuri.pdf)]
- **How It Works.** Rather than running as a heavy packet-inspecting proxy, Foomuuri acts as a compiler. It reads human-readable configuration files from `/etc/foomuuri/` and translates them directly into native kernel `nftables` rulesets. [[1](https://manpages.debian.org/testing/foomuuri/foomuuri.8.en.html), [2](https://linux-ipsec.org/slides/2025/soini-handling-ipsec-with-foomuuri.pdf)]
- **Key Capabilities.** It supports bidirectional filtering (incoming, outgoing, and forwarded traffic), automatic IPv4/IPv6 splitting, dynamic DNS and IP-list refreshing, NAT/port forwarding, and `firewalld` D-Bus emulation so desktop tools like NetworkManager can switch zones automatically. [[1](https://github.com/FoobarOy/foomuuri), [2](https://packages.debian.org/trixie-backports/all/foomuuri-firewalld)]

Does It Work Well?

- **Clean, Declarative Workflow.** System administrators generally praise its configuration syntax as much easier to read and version-control (such as with Git or Ansible) than raw `nftables.conf` syntax or verbose `firewalld` XML files.
- **Atomic Ruleset Reloads.** Because it compiles straight to native `nftables`, updating rules happens atomically without dropping existing network connections.
- **Production Readiness.** Despite being younger than UFW or `firewalld` (started in 2023), it was built specifically to replace legacy corporate firewalls and handles complex multi-ISP, VPN/IPsec, and router setups reliably.
- **Smaller Community.** Its main limitation compared to UFW is ecosystem size. You will find far fewer community tutorials or third-party troubleshooting guides online, relying mostly on the official [Foomuuri documentation](https://foomuuri.foobar.fi/latest/). [[1](https://github.com/FoobarOy/foomuuri), [2](https://github.com/tubby1981/ansible), [3](https://linux-ipsec.org/slides/2025/soini-handling-ipsec-with-foomuuri.pdf), [4](https://infosecmonkey.com/nftables-firewall-the-modern-linux-packet-filter-and-every-gui-worth-using-to-manage-it/), [5](https://foomuuri.foobar.fi/latest/)]

Is It Considered Secure?

- **Strong Packet Filtering Architecture.** At the network level, Foomuuri is as secure as the Linux kernel's `nftables` engine because it compiles directly into standard kernel netfilter tables rather than implementing its own packet filter. It enforces strict zone isolation and supports rate limiting and automatic IP banning. [[1](https://github.com/FoobarOy/foomuuri), [2](https://linux-ipsec.org/slides/2025/soini-handling-ipsec-with-foomuuri.pdf), [3](https://manpages.debian.org/testing/foomuuri/foomuuri.8.en.html)]
- **Past Local Privilege Vulnerabilities.** Like any daemon running with root privileges, its management layer has required security patches. For example, in May 2026, Ubuntu published [Ubuntu Security Notice USN-8326-1](https://ubuntu.com/security/notices/USN-8326-1) to patch two local D-Bus authorization and input validation flaws (`CVE-2025-67603` and `CVE-2025-67858`) that could allow an unprivileged local user to alter firewall rules. [[1](https://ubuntu.com/security/notices/USN-8326-1)]
- **Security Best Practices.** The vulnerabilities were responsibly disclosed by SUSE security researcher Matthias Gerstner and promptly patched upstream. If you use Foomuuri on Ubuntu, keep automatic security updates enabled, and consider disabling the `foomuuri-dbus` service if you do not need dynamic NetworkManager zone switching. [[1](https://packages.debian.org/trixie-backports/all/foomuuri-firewalld), [2](https://ubuntu.com/security/notices/USN-8326-1)]

Would you like to see a **sample Foomuuri configuration file** for a standard server, or compare how its **daily workflow** feels next to **firewalld**?