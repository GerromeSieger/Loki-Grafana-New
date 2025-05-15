# Docker Container Log Collection with Loki and Grafana

This guide will walk you through setting up Grafana Loki to collect logs from Docker containers and visualize them using Grafana, all within a Docker Compose environment.

## Prerequisites

- Docker and Docker Compose installed on your server
- Basic understanding of Docker concepts
- Root or sudo access to your server

## Step 1: Create Project Directory

```bash
mkdir loki-grafana-logging
cd loki-grafana-logging
```

## Step 2: Create Docker Compose File

Create a file named `docker-compose.yml` with the following content:

```yaml
services:
  # Loki is the log aggregation system
  loki:
    image: grafana/loki:2.8.3
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki-network
    restart: unless-stopped

  # Promtail collects logs and forwards to Loki
  promtail:
    image: grafana/promtail:2.8.3
    container_name: promtail
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yaml
    depends_on:
      - loki
    networks:
      - loki-network
    restart: unless-stopped

  # Grafana for visualizing logs
  grafana:
    image: grafana/grafana:10.1.2
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana-provisioning:/etc/grafana/provisioning
    depends_on:
      - loki
    networks:
      - loki-network
    restart: unless-stopped

networks:
  loki-network:
    driver: bridge

volumes:
  loki-data:
  grafana-data:
```

## Step 3: Create Loki Configuration

Create a file named `loki-config.yaml` with the following content:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

analytics:
  reporting_enabled: false
```

## Step 4: Create Promtail Configuration

Create a file named `promtail-config.yaml` with the following content:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'logstream'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        target_label: 'project'
```

## Step 5: Configure Grafana Provisioning

Create the directory structure for provisioning Grafana with data sources and dashboards:

```bash
mkdir -p grafana-provisioning/datasources
mkdir -p grafana-provisioning/dashboards
```

## Step 6: Create Grafana Datasource Configuration

Create a file named `grafana-provisioning/datasources/datasource.yaml` with the following content:

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: false
```

## Step 7: Create Grafana Dashboard Configuration

Create a file named `grafana-provisioning/dashboards/dashboard.yaml` with the following content:

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: true
```

Create a basic dashboard file named `grafana-provisioning/dashboards/docker-logs-dashboard.json`:

```json
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": {
          "limit": 100,
          "matchAny": false,
          "tags": [],
          "type": "dashboard"
        },
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "loki",
        "uid": "P8E80F9AEF21F6940"
      },
      "gridPos": {
        "h": 9,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 2,
      "options": {
        "dedupStrategy": "none",
        "enableLogDetails": true,
        "prettifyLogMessage": false,
        "showCommonLabels": false,
        "showLabels": false,
        "showTime": true,
        "sortOrder": "Descending",
        "wrapLogMessage": false
      },
      "targets": [
        {
          "datasource": {
            "type": "loki",
            "uid": "P8E80F9AEF21F6940"
          },
          "editorMode": "builder",
          "expr": "{container=~\".+\"}",
          "queryType": "range",
          "refId": "A"
        }
      ],
      "title": "All Container Logs",
      "type": "logs"
    }
  ],
  "refresh": "",
  "schemaVersion": 38,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Docker Container Logs",
  "version": 0,
  "weekStart": ""
}
```

## Step 8: Start the Stack

Start all services using Docker Compose:

```bash
docker-compose up -d
```

## Step 9: Access Grafana

1. Open your web browser and navigate to `http://your-server-ip:3000`
2. Log in with the default credentials:
   - Username: `admin`
   - Password: `admin`
3. You'll be prompted to change the password on first login.

## Step 10: Explore Logs in Grafana

1. Navigate to the "Explore" section in the left sidebar.
2. Ensure "Loki" is selected as the data source.
3. In the Loki query field, you can start with a simple query:
   ```
   {container=~".+"}
   ```
   This will show logs from all containers.

4. For specific container logs, use a filter:
   ```
   {container="example-app"}
   ```

5. To search for specific text in logs:
   ```
   {container="example-app"} |= "error"
   ```

## Step 11: Using the Pre-Configured Dashboard

1. Navigate to "Dashboards" in the left sidebar.
2. Click on "Browse" and select "Docker Container Logs" dashboard.
3. The dashboard will display logs from all containers by default.

## Advanced Loki Queries

### Filter by Log Level
```
{container="example-app"} |= "ERROR"
```

### Multiple Filters
```
{container="example-app", service="app"} |= "error" != "warning"
```

### Parse JSON Logs
```
{container="example-app"} | json | level="error"
```

### Rate of Log Messages
```
rate({container="example-app"}[5m])
```

## Troubleshooting

### No Logs Appearing in Grafana
1. Check if all containers are running:
   ```bash
   docker-compose ps
   ```

2. Check Loki logs:
   ```bash
   docker-compose logs loki
   ```

3. Check Promtail logs:
   ```bash
   docker-compose logs promtail
   ```

4. Ensure the Docker socket is accessible to Promtail.

### Cannot Connect to Grafana
1. Verify Grafana container is running:
   ```bash
   docker-compose ps grafana
   ```

2. Check if the port 3000 is accessible:
   ```bash
   curl http://localhost:3000
   ```

3. Check firewall settings if accessing remotely.

## Security Considerations

1. **Change Default Credentials**: Update the default Grafana admin password.

2. **Use a Reverse Proxy**: For production environments, consider using NGINX or Traefik as a reverse proxy with SSL.

3. **Network Isolation**: Adjust the Docker network settings based on your security requirements.

4. **Access Control**: Configure Grafana's user management for proper access control.

## Conclusion

You now have a complete logging stack using Loki for collecting Docker container logs and Grafana for visualization. This setup provides a powerful way to monitor and troubleshoot your containerized applications in real-time.

For production environments, consider adding:
- Persistent storage for logs
- Alerting based on log patterns
- Additional metrics collection with Prometheus
- Load balancing for scaling
