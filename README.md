# dmi-spark

A tiny cloud-init replacement for local KVM/QEMU guests. At first boot it reads
its configuration from **SMBIOS OEM strings** (DMI type 11, injected by the
hypervisor) and applies it: hostname, users + passwords, group membership, SSH
authorized keys, and DNS self-registration. No metadata server, no network
datasource - the config rides in the VM definition.

## Config format

One OEM string, prefixed `dmi-spark::`, with `!`-separated `key:value` entries:

| key | meaning |
|-----|---------|
| `h:<fqdn>` | set hostname |
| `p:<user>@<b64>` | user; value is a base64 crypt(3) password hash (created if missing) |
| `g:<user>@<group>` | add user to group (repeatable) |
| `s:<user>@<b64>` | base64 SSH public key -> `~user/.ssh/authorized_keys` |
| `r:<host>` | DNS registry address |
| `m:<method>` | registration method (`dns`) |

Inject with QEMU/libvirt, e.g.:

```sh
qemu-system-x86_64 ... \
  -smbios type=11,value="dmi-spark::h:vm1.lab!m:dns!s:root@$(base64 -w0 key.pub)"
```

(OEM strings are root-only and visible to whoever controls the VM definition -
this is for an internal, trusted host. Passwords/keys are base64, not encrypted.)

## Runtime

Boot is split into phases so the parts that need no network run as early as
possible, and only DNS registration waits for the network:

- `dmi-spark-hostname.service` (phase 0, `sysinit.target`, `DefaultDependencies=no`)
  runs `dmi-spark hostname`: sets only the transient (runtime) hostname via the
  `hostname` syscall. No disk writes, no D-Bus - runs before logging/sshd come up
  so they see the right name from the start.
- `dmi-spark.service` (phase 1, `After=local-fs.target`, no network deps) runs
  `dmi-spark start`: static hostname (`/etc/hostname`), users + passwords, group
  membership, SSH keys. Idempotent via markers under `/var/opt/dmi-spark/`.
- `dmi-spark-register.service` (phase 2, `After=network-online.target`) runs
  `dmi-spark register`: DNS self-registration plus starting the refresh timer.
- `dmi-spark-dns@<sec>.timer` -> `dmi-spark-dns.service` re-registers the host
  with the DNS registry every `<sec>` seconds (started by `dmi-spark register`).

By default the host is **not** deregistered from DNS on shutdown (a reboot would
only churn the entry, and the stale record is overwritten on next boot anyway).
For hosts that should remove their record when they go down, an optional
`dmi-spark-unregister.service` is shipped but **not enabled**: its `ExecStop`
runs `dmi-spark unregister` while the network is still up. Enable it per host
with `systemctl enable dmi-spark-unregister.service`.

Registration uses bundled [plotka-register](https://github.com/rjsocha/plotka-register);
OEM strings are read by bundled [oem-string](https://github.com/rjsocha/oem-string).
Both ship in `/usr/libexec/dmi-spark/`.

## Build

```sh
make            # -> result/dmi-spark_<ver>_<arch>.deb
```

Vendored helper binaries live in `binary/` (release builds of oem-string and
plotka-register). Currently builds linux/amd64.

## License

Public domain. No rights reserved.
