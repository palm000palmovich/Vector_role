vector
======

Installs and runs [Vector](https://vector.dev/) — an observability data pipeline — as a systemd
service. By default it collects host metrics (CPU, disk, filesystem, load, memory, network) and
ships them to a ClickHouse sink, matching the ClickHouse + Vector + LightHouse stack this role is
part of.

The role downloads a prebuilt Vector binary release, installs it under a dedicated system user,
and renders its pipeline configuration (`sources` / `sinks`) from two plain variables, so the
whole pipeline can be replaced without touching the role itself.

Requirements
------------

- Target host: Debian/Ubuntu (systemd-based).
- Internet access from the target host to `packages.timber.io`, to download the Vector release.
- Only `ansible.builtin` modules are used.

Role Variables
---------------

Defined in `defaults/main.yml` (safe to override from inventory / group_vars / host_vars):

| Variable              | Default                        | Description                                           |
|------------------------|---------------------------------|---------------------------------------------------------|
| `vector_version`       | `0.41.1`                        | Vector release version to install                       |
| `vector_arch`          | `x86_64-unknown-linux-musl`     | Release archive architecture/target triple               |
| `vector_download_url`  | computed from the two above     | Full URL of the release tarball                          |
| `vector_install_dir`   | `/opt/vector`                   | Where the Vector binary is unpacked                       |
| `vector_config_dir`    | `/etc/vector`                   | Where `vector.yaml` is written                            |
| `vector_data_dir`      | `/var/lib/vector`                | Vector's working/data directory                           |
| `vector_user` / `vector_group` | `vector`                 | Dedicated system account Vector runs as                   |
| `vector_sources`       | `host_metrics` collector        | Dict rendered as-is into the `sources:` section            |
| `vector_sinks`         | `clickhouse` sink to `localhost:8123` | Dict rendered as-is into the `sinks:` section         |

To ship metrics to a real ClickHouse host, override `vector_sinks` (typically from
`group_vars/vector.yml` in the playbook that consumes this role), e.g.:

```yaml
vector_sinks:
  clickhouse:
    type: clickhouse
    inputs:
      - host_metrics
    endpoint: "http://<clickhouse-host>:8123"
    database: default
    table: vector_metrics
    skip_unknown_fields: true
```

Defined in `vars/main.yml` (internal, derived from the defaults above):

| Variable                     | Value                                    |
|-------------------------------|---------------------------------------------|
| `vector_archive_path`         | `/tmp/vector.tar.gz`                         |
| `vector_binary_path`          | `{{ vector_install_dir }}/bin/vector`        |
| `vector_config_path`          | `{{ vector_config_dir }}/vector.yaml`        |
| `vector_systemd_unit_path`    | `/etc/systemd/system/vector.service`         |

Tags
----

| Tag        | Tasks                                                              |
|------------|-----------------------------------------------------------------------|
| `packages` | Create the `vector` system user/group                                 |
| `deploy`   | Create directories, download and unpack the Vector binary             |
| `config`   | Render `vector.yaml` from `vector_sources`/`vector_sinks`             |
| `systemd`  | Render the systemd unit, reload systemd, start/enable the service     |

Dependencies
------------

None. A `clickhouse` group/host reachable at the address used in `vector_sinks` is expected at
the infrastructure level, but this role does not depend on the `clickhouse` role directly.

Example Playbook
-----------------

```yaml
- hosts: vector
  become: true
  roles:
    - role: vector
      vector_sinks:
        clickhouse:
          type: clickhouse
          inputs: [host_metrics]
          endpoint: "http://51.250.11.245:8123"
          database: default
          table: vector_metrics
          skip_unknown_fields: true
```

License
-------

MIT

Author Information
-------------------

https://github.com/palm000palmovich
