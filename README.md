# alloy-log-collection-config

[Grafana Alloy](https://grafana.com/docs/alloy/) configuration for collecting and shipping logs and metrics from systemd hosts.

## Installation

Add the Grafana RPM repository and install:

```sh
wget -q -O gpg.key https://rpm.grafana.com/gpg.key
sudo rpm --import gpg.key
echo -e '[grafana]\nname=grafana\nbaseurl=https://rpm.grafana.com\nrepo_gpgcheck=1\nenabled=1\ngpgcheck=1\ngpgkey=https://rpm.grafana.com/gpg.key\nsslverify=1\nsslcacert=/etc/pki/tls/certs/ca-bundle.crt' | sudo tee /etc/yum.repos.d/grafana.repo
sudo dnf install alloy
```

## Deployment

Copy all `*.alloy` files to `/etc/alloy/`:

```sh
sudo cp *.alloy /etc/alloy/
```

The RPM package configures Alloy via `/etc/sysconfig/alloy`. Change `CONFIG_FILE` to point at the directory instead of a single file:

```sh
sudo sed -i 's|CONFIG_FILE=.*|CONFIG_FILE="/etc/alloy/"|' /etc/sysconfig/alloy
```

Alloy loads all `*.alloy` files in the directory and merges them. Components can reference each other across files.

Enable and start (first time):

```sh
sudo systemctl enable --now alloy
```

Reload after config changes:

```sh
sudo systemctl reload alloy
```

## Files

| File | Purpose |
|------|---------|
| `config.alloy` | Node metrics: `node_exporter` via `prometheus.exporter.unix` + self-scrape |
| `loki-journal.alloy` | Reads the systemd journal, relabels entries, parses and normalizes log levels |
| `loki-write.alloy` | Loki push endpoint — update the URL here |

## Log level normalization

Levels are derived in two passes:

1. **Relabel** (`loki.relabel.journal`): maps `__journal_priority_keyword` (syslog severity) to a normalized label — `notice` → `info`, `warning` → `warn`, `err` → `error`.
2. **Pipeline** (`loki.process.parse_level`): extracts `level=<value>` from the log line content and normalizes aliases (`warning`→`warn`, etc.). Falls back to `unknown` if nothing is detected.

### Audit logs

Linux kernel audit records (`AUDIT<TYPE> … res=…`) are recognized automatically:

- `res=failed` → `level=warn`
- `res=success` → `level=info`
- The `audit_res` label is set for filtering in Loki (e.g. `{audit_res="failed"}`)

## Configuration

Update `loki-write.alloy` with your Loki push URL before deploying.
