# Design Consideration - pin one Foomuuri release across Debian and Ubuntu hosts (0.31 or later, for the D-Bus fixes)

how tricky would you guess that this would be: Version: pin one Foomuuri release across Debian and Ubuntu hosts (0.31 or later, for the D-Bus fixes), rather than taking each distribution's package.

I’d call it easy to moderate. Getting one version installed everywhere is easy. The ongoing work is keeping up with updates yourself.

**Why it’s easy:** Foomuuri is pure Python. Debian packages it as `arch:all`, so the same `.deb` should install on any Debian-based system. What it needs is ordinary: Python 3.9 or later, nftables 1.0 or later with JSON support, and optional extras like `python3-dbus`, `python3-gi`, `python3-systemd`, `python3-urllib3` and `fping`. Ubuntu 24.04 (and so Mint 22) has Python 3.12 and nftables 1.0.9, and Debian 13 is newer on both. I haven’t seen anything Ubuntu-specific that would get in the way.

**The simplest approach** is to take Debian’s trixie-backports `.deb` (and `foomuuri-firewalld` if you want it), at 0.31 or later. You’d install that same file everywhere with Ansible:

- Download it, check its SHA-256 against a value stored in the role, and install it with `apt` (`deb:` in the Ansible `apt` module). On Debian 13 you could enable trixie-backports instead, but the same file everywhere is simpler.
- Add an apt pin, or `apt-mark hold`, so a routine upgrade never swaps in Ubuntu’s 0.22. A plain `apt upgrade` won’t downgrade anyway, because the backports version number sorts higher, but the pin makes your intent explicit.
- Put the version in one role variable, so a bump is a one-line change you can review.

That’s maybe half a day to a day for the role, plus testing in a Debian 13 VM and a Mint 22 VM, which you need anyway for the assess checks.

**Where it gets trickier:**

- **Security updates become your job.** Neither Debian nor Ubuntu will patch a version you installed by hand. You’d watch the upstream changelog, oss-security and Debian’s backports, and bump the version yourself. The January D-Bus fix is the example: upstream fixed it within weeks, and you’d want that rather than waiting five months for Ubuntu.
- **Breaking changes arrive with each bump.** Pinning protects you between bumps, but every upgrade needs the changelog read and `foomuuri check` run against your templated config before it reaches remote hosts. That’s a good argument for a small CI job that runs `foomuuri check` on the rendered templates.
- **Possible packaging differences.** Debian 13 and Ubuntu 24.04 differ in polkit and systemd versions. The Polkit policy file and the `ProtectSystem=full` units should be fine on both, but that’s one of the things to confirm on Mint.
- **If a backports `.deb` ever fails on Ubuntu,** the fallback is building your own `.deb` from Debian’s packaging and hosting it in a small local apt repo. That’s a bigger step, roughly a day to set up, plus a little work for every release.

I’d avoid installing from the upstream tarball with `make install`. It works, but apt doesn’t know it’s there, which makes upgrades and clean removal messier.

The one real check before you rely on this is installing the backports `.deb` on a fresh Mint 22 VM and seeing whether `apt` resolves its dependencies without complaint. Want me to add this as a design decision with that check in the Foomuuri entry, as a 0.1.1?
