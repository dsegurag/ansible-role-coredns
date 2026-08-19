# Ansible Role: CoreDNS

Installs [CoreDNS](https://coredns.io) from the official binary release - picking the asset
that matches the host's CPU architecture and verifying its SHA-256 - and runs it as a systemd
service.

The unit follows the upstream example in
[coredns/deployment](https://github.com/coredns/deployment/tree/master/systemd), with the
deprecated `PermissionsStartOnly=` dropped, logging left to the journal instead of
`/var/log/coredns.log`, and a few hardening options added.

## Requirements

- A Debian or Ubuntu host with systemd.
- `ansible-core` 2.18 or higher. No external collections.
- systemd 247 or newer (bookworm ships 252) **only** if you use `coredns_credentials`;
  everything else works on older systemd.

## Quick start

The role ships a minimal forwarding config, so this alone gives you a working resolver:

```yaml
- hosts: dns
  become: true
  roles:
    - role: coredns
```

## Configuration

Three sources, in order of precedence - the first one set wins:

| Variable | Description | Looked up in | Default |
| - | - | - | - |
| `coredns_config_content` | Corefile written from inline content. | - | `""` |
| `coredns_config_src` | Raw Corefile, copied verbatim. | `files/` | `""` |
| `coredns_config_template` | Jinja template rendered to the Corefile. | `templates/` | `Corefile.default.j2` |

The config that ships with the role is therefore a fallback, not a competing source: setting
one of the other two overrides it, no need to blank `coredns_config_template` first. The role
only fails if all three are empty.

To use your own template, drop it in the playbook's `templates/` and name it:

```yaml
coredns_config_template: Corefile.j2
```

A Corefile that needs no Jinja at all - one that gets its values from `{$VAR}` environment
variables, say - is better copied verbatim, and must be, if it contains `{{ ... }}` (the
*template* plugin's `answer "{{ .Name }} ..."` is not valid Jinja):

```yaml
coredns_config_src: Corefile      # files/Corefile
```

> A role's `templates/` takes precedence over the playbook's, so a template named
> `Corefile.j2` *inside this role* would silently shadow yours. That is why the built-in one
> is called `Corefile.default.j2` - any name of your own that is not that one is safe.

There is no `validate:` step: the CoreDNS binary has no config-check flag, so a broken Corefile
surfaces as a failing restart.

`coredns_environment` is written to `/etc/default/coredns` and read by the unit through
`EnvironmentFile=`; the Corefile expands entries with the `{$VAR}` syntax:

```yaml
coredns_environment:
  UPSTREAM: 10.0.0.1
```

```
forward . {$UPSTREAM}
```

## Files the service user cannot read

Optional, and only relevant when CoreDNS serves DoT (`tls://.:853`), DoH (`https://.:443`) or
gRPC: the *tls* plugin has to open a private key, and a key is usually `0600 root:root` -
unreadable by an unprivileged `coredns`.

List such files in `coredns_credentials`. systemd reads them **as root** when the unit starts
and exposes copies under `$CREDENTIALS_DIRECTORY` (`/run/credentials/coredns.service/`, a
tmpfs) with mode `0400` owned by the service user. Nothing on disk is chmod'ed and no key is
copied into a second permanent location:

```yaml
coredns_credentials:
  fullchain.pem: /path/to/fullchain.pem
  privkey.pem: /path/to/privkey.pem
```

```
tls {$CREDENTIALS_DIRECTORY}/fullchain.pem {$CREDENTIALS_DIRECTORY}/privkey.pem
```

Two things to know:

- The unit **fails to start** if a source file is missing, so whatever issues the certificate
  has to run before this role.
- CoreDNS reads the key pair once, at startup, and `reload`/`SIGUSR1` does not pick up a new
  one ([coredns#4994](https://github.com/coredns/coredns/issues/4994)). Renewal therefore has
  to **restart** the service, not reload it.

Running as root (`coredns_user: root`, `coredns_create_user: false`) sidesteps the question
entirely and lets the Corefile point at the real paths, at the cost of privilege separation on
a service listening on three public ports.

### Example: certificates from certbot

With [dsegurag.ssl](https://galaxy.ansible.com/ui/standalone/roles/dsegurag/ssl/), which issues
a Let's Encrypt certificate and renews it unattended:

```yaml
- hosts: dns
  become: true
  roles:
    - role: dsegurag.ssl
      vars:
        ssl_domains: ["dns.example.com"]
        ssl_cloudflare_token: "{{ lookup('ansible.builtin.env', 'CLOUDFLARE_API_TOKEN') }}"
        # restart, not reload: the key pair is read at startup
        ssl_deploy_hook_command: systemctl restart coredns

    - role: coredns
      vars:
        coredns_config_template: Corefile.j2
        coredns_credentials:
          fullchain.pem: "{{ ssl_fullchain }}"
          privkey.pem: "{{ ssl_privatekey }}"
```

Any other source works the same way - a certificate copied in by another role, a key from a
vault, an internal CA - as long as the file exists on the host before the unit starts.

## Role Variables

### Installation

| Variable | Description | Default |
| - | - | - |
| `coredns_version` | Release to install. | `1.14.6` |
| `coredns_arch_map` | Ansible architecture fact to release asset suffix. | `aarch64`/`arm64` → `arm64`, `x86_64` → `amd64`, `armv6l`/`armv7l` → `arm`, plus `riscv64`, `s390x`, `ppc64le` |
| `coredns_arch` | Asset suffix for this host. | Looked up in `coredns_arch_map`; an unmapped architecture fails the run |
| `coredns_download_url`, `coredns_checksum_url` | Where the tarball and its checksum come from. | GitHub releases |
| `coredns_download_dir` | Where the tarball is cached on the host. | `/var/cache/coredns` |
| `coredns_bin_dir` | Where the binary is unpacked. | `/usr/local/bin` |

The binary is downloaded only when `coredns --version` disagrees with `coredns_version`, so
re-running the role is a no-op, and changing the version - up or down - reinstalls and
restarts.

### Configuration

| Variable | Description | Default |
| - | - | - |
| `coredns_config_template`, `coredns_config_src`, `coredns_config_content` | Corefile sources, see above. | `Corefile.default.j2`, `""`, `""` |
| `coredns_config_dir`, `coredns_config_file` | Where it lands, `0640 root:<group>`. | `/etc/coredns`, `/etc/coredns/Corefile` |
| `coredns_environment` | Dict written to `/etc/default/coredns`. | `{}` |

### Service

| Variable | Description | Default |
| - | - | - |
| `coredns_user`, `coredns_group` | Service identity. | `coredns` |
| `coredns_create_user` | Create that system user and group. | `true` |
| `coredns_state_dir` | Working directory; must live under `/var/lib` (its basename becomes `StateDirectory=`). | `/var/lib/coredns` |
| `coredns_credentials` | Files passed in via `LoadCredential=`, as `name: source path`. | `{}` |
| `coredns_listen_port` | Default port for zones that carry none. Empty keeps 53. | `""` |
| `coredns_extra_args` | Extra command line arguments, e.g. `["-quiet"]`. | `[]` |
| `coredns_service_enabled`, `coredns_service_state` | Passed to `systemd_service`. | `true`, `started` |

The unit keeps `CAP_NET_BIND_SERVICE` (needed to bind 53, 443 and 853 as a non-root user) and
adds `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`,
`ProtectKernelTunables`, `ProtectControlGroups` and `RestrictSUIDSGID`.

## License

MIT License

## Author Information

This role was created by Daniel Segura.
