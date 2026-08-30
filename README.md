# Ansible Role: Kibana

Kibana 9.x, pointed at an authenticated, TLS-protected Elasticsearch cluster.

## Provenance

| Part | Upstream | Synced at |
| --- | --- | --- |
| `handlers/main.yml` (adapted) | [geerlingguy/ansible-role-kibana](https://github.com/geerlingguy/ansible-role-kibana) | `41ea27d2` |
| `kibana.yml` settings | [telekom-security/tpotce](https://github.com/telekom-security/tpotce), `docker/elk/kibana/Dockerfile` | `8a228130` |

`handlers/main.yml` is not verbatim — it uses `ansible.builtin.systemd` with
`daemon_reload: true` in place of upstream's free-form `service:` module, even
at the commit cited above. `.github/workflows/upstream-sync.yml` diffs it
against upstream on a schedule and flags (never applies) any change for
review.

## Changes required for 9.x

The previous `kibana.yml.j2` ended with a block of feature switches:

    xpack.infra.enabled: false
    xpack.logstash.enabled: false
    xpack.canvas.enabled: false
    xpack.spaces.enabled: false
    xpack.apm.enabled: false
    xpack.security.enabled: false
    xpack.uptime.enabled: false
    xpack.securitySolution.enabled: false
    xpack.ml.enabled: false

**Every one of these is gone in Kibana 9**, and Kibana refuses to start on an
unrecognised setting rather than ignoring it. They have been removed, not
commented out. T-Pot sets none of them either. The render test in this
repository asserts that no `xpack.*.enabled` key can reappear in the output.

Other 9.x requirements now handled:

* Kibana will not connect as the `elastic` superuser. It authenticates as the
  built-in `kibana_system` account, whose password the `elasticsearch` role
  sets over the API.
* `xpack.encryptedSavedObjects.encryptionKey`, `xpack.reporting.encryptionKey`
  and `xpack.security.encryptionKey` are required, at 32 characters or more.
  Without them Kibana generates random keys at every start, which invalidates
  encrypted saved objects and drops every session. The role asserts on all
  three.
* The "Remove Enterprise Search" task is gone — Enterprise Search was removed
  from the stack entirely in 9.0, so the path it deleted no longer exists.

## Carried over from T-Pot

`elasticsearch.requestTimeout` and `elasticsearch.shardTimeout` at 60s, the
`unifiedSearch.autocomplete.valueSuggestions` timeout and `terminateAfter`
bumps, and telemetry and newsfeed disabled. The honeypot dashboards issue wide
terms aggregations and time out on the stock values.

## Configuration

Everything is built into a dict in `templates/kibana.yml.j2` and merged with
`kibana_extra_config`, so arbitrary settings can be added without editing the
template:

    kibana_extra_config:
      server.maxPayload: 4194304

## Debian only

`tasks/setup-RedHat.yml` and `templates/kibana.repo.j2` have been dropped. They
added the yum repository but never installed the package once version pinning
moved into the Debian path, so the RedHat route silently did nothing. The role
now fails fast on a non-Debian `os_family`.

## Molecule

`molecule test` converges the role on `geerlingguy/docker-debian13-ansible`
against a throwaway CA generated on the controller (standing in for the
certificate the `elasticsearch` role would otherwise hand over), and verifies
that Kibana starts clean, serves `/api/status`, and that none of the settings
removed in 9.x reappear in the rendered config. It replaces the old scenario,
which converged `geerlingguy.elasticsearch` + `geerlingguy.kibana` and could
not exercise a role that now requires real TLS credentials.

## Example Playbook

    - hosts: kibana
      roles:
        - role: honeynet.kibana
      vars:
        elastic_version: "9.3.5"
        elastic_repo_channel: "9.x"
        kibana_elasticsearch_hosts:
          - https://elasticsearch-01.example.org:9200
        kibana_elasticsearch_password: "{{ vault_kibana_system_password }}"
        kibana_elasticsearch_ca_file: /etc/kibana/certs/ca.crt
        kibana_encrypted_saved_objects_key: "{{ vault_kibana_encrypted_saved_objects_key }}"
        kibana_reporting_key: "{{ vault_kibana_reporting_key }}"
        kibana_security_key: "{{ vault_kibana_security_key }}"

See `defaults/main.yml` for the full set of variables, and
`molecule/default/converge.yml` for a complete working configuration.

## License

MIT

## Author Information

The Honeynet Project. Role scaffolding originally by
[Jeff Geerling](https://github.com/geerlingguy).
