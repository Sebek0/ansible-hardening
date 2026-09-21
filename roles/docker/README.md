# ansible-role-docker

Installs Docker CE, the Compose plugin, and manages `daemon.json` across
Debian/Ubuntu and RHEL/Rocky/CentOS hosts. Built for real fleet provisioning:
version pinning, proxy support, group membership, and optional scheduled
`docker system prune`.

## Requirements

- Ansible >= 2.14
- Target hosts running Ubuntu 20.04+/Debian 11+ or RHEL/Rocky/CentOS 8+
- `community.general` collection is **not** required — everything here uses
  `ansible.builtin`, deliberately, to keep the role dependency-free.

## Role Variables

See `defaults/main.yml` for the full, commented list. The ones you'll touch
most often:

| Variable | Default | Purpose |
|---|---|---|
| `docker_version` | `latest` | Pin a specific Docker CE version string |
| `docker_users` | `[]` | Users added to the `docker` group |
| `docker_daemon_log_opts` | 10m x3 rotation | Prevents container logs from filling disk |
| `docker_daemon_storage_driver` | `overlay2` | Rarely needs changing on modern kernels |
| `docker_daemon_insecure_registries` | `[]` | For internal/self-hosted registries without valid TLS |
| `docker_daemon_data_root` | `""` | Relocate `/var/lib/docker` to a larger volume |
| `docker_use_proxy` | `false` | Set true + `docker_http_proxy`/`docker_https_proxy` behind a corporate proxy |
| `docker_prune_enabled` | `false` | Adds a systemd timer for automatic cleanup |

## Example Playbook

```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: docker
      vars:
        docker_version: "5:24.0.7-1~ubuntu.22.04~jammy"
        docker_users:
          - deploy
          - ci-runner
        docker_daemon_data_root: "/data/docker"
        docker_daemon_insecure_registries:
          - "registry.internal.example.com:5000"
        docker_prune_enabled: true
```

### Behind a corporate proxy

```yaml
    docker_use_proxy: true
    docker_http_proxy: "http://proxy.example.com:3128"
    docker_https_proxy: "http://proxy.example.com:3128"
    docker_no_proxy: "localhost,127.0.0.1,.internal.example.com"
```

## Notes on idempotency

- Package installation branches on `docker_version == "latest"` vs a pinned
  string; pinned installs are followed by a package hold (`apt`) or
  `dnf versionlock` so an unrelated `apt upgrade`/`dnf upgrade` elsewhere in
  your pipeline can't silently move Docker versions.
- `daemon.json` is rendered from a dict built in the template, so hosts that
  don't set optional keys (mirrors, DNS, bip, etc.) get a minimal, clean file
  rather than a template full of empty strings.
- The `validate:` parameter on the `template` task runs `python3 -m json.tool`
  against the rendered file before it's put in place, so a bad Jinja
  substitution fails the play instead of leaving Docker unable to start.
- Docker only restarts (via handler) when `daemon.json` or the proxy drop-in
  actually changes — not on every run.
