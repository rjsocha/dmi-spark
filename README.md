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

- `dmi-spark.service` (enabled, oneshot) runs `dmi-spark start` at boot: decode +
  apply. Idempotent via markers under `/var/opt/dmi-spark/`.
- `dmi-spark-dns@<sec>.timer` -> `dmi-spark-dns.service` re-registers the host
  with the DNS registry every `<sec>` seconds (started by `dmi-spark start`).

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
