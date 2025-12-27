# Simple docker exporter

A lightweight Prometheus exporter for Docker container statistics, ported from JavaScript to Go for maximum efficiency and minimal resource footprint.

## Motivation

In modern containerized environments, monitoring is crucial. However, popular solutions like **cAdvisor** can sometimes consume significant CPU resources. 

I was looking for a lighter alternative and found the excellent [docker_stats_exporter](https://github.com/wywywywy/docker_stats_exporter) by **wywywywy**. While the original Node.js implementation worked perfectly, it inspired me to create this **Go port**. By moving to Go, the exporter becomes a single static binary with a tiny memory footprint (~10MB RAM) and significantly lower CPU overhead.

## Features

- **Resource Efficient:** Written in Go, minimal overhead compared to Node.js or cAdvisor.
- **Accurate CPU Metrics:** Manages internal state to calculate precise CPU usage deltas.
- **Fail-Fast:** Validates Docker connection on startup and exits if the socket is missing.
- **Flexible:** Supports both Unix Socket and TCP connections to Docker.
- **Clean Metrics:** Automatically cleans up data for removed containers.

## Metrics

The exporter provides the following metrics (prefixed with `dockerstats_`):

| Metric | Description |
| :--- | :--- |
| `cpu_usage_ratio` | CPU usage percentage (0-100%) |
| `memory_usage_bytes` | Current memory usage in bytes |
| `memory_usage_rss_bytes` | Memory RSS usage in bytes |
| `memory_limit_bytes` | Container memory limit |
| `memory_usage_ratio` | Memory usage percentage (0-100%) |
| `network_received_bytes` | Network bytes received |
| `network_transmitted_bytes` | Network bytes transmitted |
| `blockio_read_bytes` | Block IO read bytes |
| `blockio_written_bytes` | Block IO written bytes |

## Usage

### Using Docker

```bash
docker run -d \
  --name simple-docker-exporter \
  --restart always \
  -p 9487:9487 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  callmetigro/simple-docker-exporter:latest
```

### Using Docker Compose

```yaml
services:
  exporter:
    image: callmetigro/simple-docker-exporter:latest
    container_name: docker-stats-exporter
    restart: always
    ports:
      - "9487:9487"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # command: ["-interval", "10", "-workers", "15"] # Optional flags
```

### Running as a systemd Service

If you want to run the exporter on a Linux host without Docker, you can set it up as a systemd service.

**1. Download the binary**

Download the latest binary for your architecture from the [Releases](https://github.com/callmetigro/simple-docker-exporter/releases) page:
```bash
mkdir docker-exporter
cd docker-exporter
wget [https://github.com/callmetigro/simple-docker-exporter/releases/download/v0.1.0/simple-docker-exporter_linux_amd64.tar.gz](https://github.com/callmetigro/simple-docker-exporter/releases/download/v0.1.0/simple-docker-exporter_linux_amd64.tar.gz)
tar -xzf simple-docker-exporter_linux_amd64.tar.gz
sudo mv simple-docker-exporter /usr/local/bin/
```

**2. Create a dedicated user**

For security, it is recommended to run the exporter under a non-root user. This user needs to be part of the `docker` group to access the Unix socket:
```bash
sudo useradd -rs /bin/false docker_exporter
sudo usermod -aG docker docker_exporter
```

**3. Create the systemd service unit**

Create a file at `/etc/systemd/system/docker-exporter.service`:
```
[Unit]
Description=Simple Docker Prometheus Exporter
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
User=docker_exporter
Group=docker
# Adjust flags as needed: -port, -interval, etc.
ExecStart=/usr/local/bin/simple-docker-exporter -port 9487
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**4. Enable and start the service**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now docker-exporter

# Check status
sudo systemctl status docker-exporter
```

### Configuration Flags

| Flag | Default | Description |
| :--- | :--- | :--- |
| `-port` | 9487 | Port to expose Prometheus metrics |
| `-interval` | 15 | Polling interval in seconds (min: 3) |
| `-workers` | 10 | Max concurrent calls to Docker API |
| `-hostip` | "" | Docker host IP (for TCP) |
| `-hostport` | 0 | Docker host port (for TCP) |
| `-v`, `--version` | | Show version and exit |

## Grafana Dashboard

To visualize collected metrics, you can use the following Grafana dashboard: [Docker Stats Dashboard](https://grafana.com/grafana/dashboards/24609-docker-stats/).

<img width="1902" alt="Grafana dashboard" src="https://github.com/user-attachments/assets/7b31711e-f95c-45ce-8b3d-1c23db46d436" />

### Quick setup

1. Copy the dashboard ID: 24609.
1. In your Grafana instance, go to Dashboards -> Import.
1. Paste the ID and select your data source.

## Credits

- Original Node.js project: [wywywywy/docker_stats_exporter](https://github.com/wywywywy/docker_stats_exporter)
- Inspiration: Seeking a lightweight alternative to cAdvisor.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

This work is a Go port of the original [docker_stats_exporter](https://github.com/wywywywy/docker_stats_exporter) by @wywywywy, which is also licensed under
