[English](README.md) | [فارسی](README-persian.md)

# Set up Blackbox Exporter

The **Prometheus Blackbox Exporter** allows you to perform "blackbox" probing of endpoints over various protocols like HTTP, HTTPS, DNS, TCP, and ICMP. 

While tools like Node Exporter look *inside* your servers (whitebox monitoring), Blackbox Exporter looks from the *outside*. It answers questions like:
- Is my website up and responding with a `200 OK`?
- How long does it take for my API to respond?
- Is the SSL certificate going to expire soon?
- Is my DNS resolving correctly?
- Can I ping this server (ICMP)?

---

## Install Blackbox Exporter as a System Service (Binary)

If you want to run Blackbox Exporter directly on your Linux server, you can set it up as a systemd service.

### 1. Download and Install the Binary

```bash
# Important Note
# If your system architecture is not amd64 the command below will not work for you.
# For example if it is arm64, replace all `amd64` with `arm64` in the commands below:

VERSION=$(curl -s https://api.github.com/repos/prometheus/blackbox_exporter/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
wget -O blackbox_exporter.tar.gz https://github.com/prometheus/blackbox_exporter/releases/download/v${VERSION}/blackbox_exporter-${VERSION}.linux-amd64.tar.gz
tar xvfz blackbox_exporter.tar.gz
```

Move the extracted binary to the system's bin path:
```bash
sudo mv blackbox_exporter-${VERSION}.linux-amd64/blackbox_exporter /usr/local/bin/
rm -rf blackbox_exporter-${VERSION}.linux-amd64 blackbox_exporter.tar.gz
```

Create a dedicated system user and a directory for its configuration:
```bash
sudo useradd --no-create-home --shell /bin/false blackbox_exporter
sudo mkdir -p /etc/blackbox_exporter
```

### 2. Create a configuration file

Create `/etc/blackbox_exporter/blackbox.yml`:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []  # Defaults to 2xx
      method: GET

  icmp_ping:
    prober: icmp
    timeout: 5s
```

> [!NOTE]
> The configuration defines "modules" (like `http_2xx` or `icmp_ping`) which specify *how* to probe a target. You can find many more examples (like DNS or TCP) in the [official documentation](https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md).

Set the permissions:
```bash
sudo chown -R blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
```

### 3. Create a systemd service

Create `/etc/systemd/system/blackbox_exporter.service`:
```ini
[Unit]
Description=Prometheus Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml

Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
```

---

## Set up Blackbox Exporter with Docker Compose (Recommended)

Running Blackbox Exporter using Docker is highly recommended and very clean.

Create a `blackbox.yml` file in your directory:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
```

Create a `docker-compose.yml`:
```yaml
services:
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox_exporter
    restart: unless-stopped
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml:ro
    command:
      - "--config.file=/etc/blackbox_exporter/config.yml"
```

Run it:
```bash
docker compose up -d
```

---

## How to Configure Prometheus to Scrape Blackbox Exporter

> [!IMPORTANT]
> Blackbox Exporter works differently from Node Exporter! You don't just scrape the exporter itself. You have to tell Prometheus to send a `target` parameter (the website or IP you want to check) and a `module` parameter to the Blackbox Exporter.

Add this job to your `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe
        - https://google.com      # Target to probe
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: {IP_ADDRESS}:9115  # The Blackbox Exporter's real hostname:port
```
*Note: Change `{IP_ADDRESS}` to the actual IP of the server where Blackbox Exporter is running. If both are on the same server but in Docker, use the host IP or docker networking.*

---

## Grafana Dashboard

To visualize the uptime, ping times, and SSL expiration of your probed endpoints, you can use community dashboards. 
Dashboard ID `7587` or `13659` are great starting points. Just go to Grafana -> Dashboards -> Import and enter the ID!
