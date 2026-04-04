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

### First-time setup

Set these variables before running the commands below:

```sh
REPO_URL="https://git.example.com/youruser/alloy-log-collection-config.git"
LOKI_HOST="loki.example.com"
```

Clone the repository as the `alloy` user and symlink the config files into `/etc/alloy/`:

```sh
sudo -u alloy git clone "$REPO_URL" ~alloy/config
sudo mkdir -p /etc/alloy
sudo bash -c 'for f in ~alloy/config/*.alloy; do ln -s "$f" /etc/alloy/; done'
```

The RPM package configures Alloy via `/etc/sysconfig/alloy`. Set the Loki push URL and point `CONFIG_FILE` at the directory:

```sh
sudo sed -i 's|CONFIG_FILE=.*|CONFIG_FILE="/etc/alloy/"|' /etc/sysconfig/alloy
echo "LOKI_URL=\"https://${LOKI_HOST}/loki/api/v1/push\"" | sudo tee -a /etc/sysconfig/alloy
```

Alloy loads all `*.alloy` files in the directory and merges them. Components can reference each other across files.

Enable and start (first time):

```sh
sudo systemctl enable --now alloy
```

### UID map

`loki-journal.alloy` contains a generated block that maps numeric owner UIDs to
usernames (for the `owner` label in Loki). Run the script to populate it after
the initial clone, then again whenever service users are added, removed, or
renamed:

```sh
sudo -u alloy ~alloy/config/scripts/generate-uid-map.sh
sudo systemctl reload alloy
```

The script edits `loki-journal.alloy` in the repo clone directly, so the file
will show as locally modified in git.

### Updating

Because the UID map leaves a local modification in the repo, stash it before
pulling, then regenerate instead of restoring the stash:

```sh
sudo -u alloy git -C ~alloy/config stash && \
sudo -u alloy git -C ~alloy/config pull && \
sudo -u alloy ~alloy/config/scripts/generate-uid-map.sh && \
sudo systemctl reload alloy
```

## Files

| File | Purpose |
|------|---------|
| `config.alloy` | Node metrics: `node_exporter` via `prometheus.exporter.unix` + self-scrape |
| `loki-journal.alloy` | Reads the systemd journal, relabels entries, parses and normalizes log levels |
| `loki-write.alloy` | Loki push endpoint — URL set via `LOKI_URL` env var in `/etc/sysconfig/alloy` |

## Log level normalization

Levels are derived in two passes:

1. **Relabel** (`loki.relabel.journal`): maps `__journal_priority_keyword` (syslog severity) to a normalized label — `notice` → `info`, `warning` → `warn`, `err` → `error`.
2. **Pipeline** (`loki.process.parse_level`): extracts `level=<value>` from the log line content and normalizes aliases (`warning`→`warn`, etc.). Falls back to `unknown` if nothing is detected.

### Audit logs

Linux kernel audit records (`AUDIT<TYPE> … res=…`) are recognized automatically:

- `res=failed` → `level=warn`
- `res=success` → `level=info`
- The `audit_res` label is set for filtering in Loki (e.g. `{audit_res="failed"}`)

## Grafana dashboards

Pre-built dashboards are in `dashboards/`. Import them via Grafana → Dashboards → Import → Upload JSON file.

| File | Description |
|------|-------------|
| `dashboards/podman-healthchecks.json` | Health check logs filtered by level; defaults to warn/error/fatal/unknown |
| `dashboards/system-journal.json` | System journal logs for a selectable host; shows warn and above, with SSH brute-force/scanner noise filtered out |

