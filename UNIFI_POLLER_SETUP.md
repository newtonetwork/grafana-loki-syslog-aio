# UniFi Poller Integration - Complete Setup

## What Was Added

### 1. UniFi Poller Service
- **Container**: `unifi-poller`
- **Port**: 9130 (Prometheus metrics)
- **Config**: `config/unifi-poller.yml`
- **Connects to**: Your UniFi Controller at https://10.0.2.1
- **SSL**: Configured to ignore self-signed certificate

### 2. Dashboards Created

#### UniFi Network Overview
**File**: `config/grafana/dashboards/no_folder/unifi_network_overview.json`

Shows:
- Total devices (APs, switches, gateways)
- Connected client count
- Device network traffic (RX/TX)
- Connected clients per device
- Device status table (CPU, memory, firmware, IP)
- CPU & memory usage graphs
- Device load averages
- Client traffic by hostname
- **Connected Clients Table** with:
  - Hostname, IP, MAC
  - Connected to which device
  - Connection type (wired/wireless)
  - WiFi signal strength (with gauge)
  - Uptime
  - Satisfaction score

#### Security Monitoring
**File**: `config/grafana/dashboards/no_folder/security_monitoring.json`

Shows:
- Failed authentication attempts (last 1h)
- Successful logins (last 1h)
- New network connections (last 1h)
- Critical events (last 1h)
- Failed auth attempts over time (by device type)
- Network client connections graph
- Failed auth by source IP table
- Recent successful logins
- SSH activity by device
- Recent critical events
- Recently connected devices (sorted by uptime)

### 3. Alert Rules Added

#### UniFi Prometheus Alerts
**File**: `config/rules/unifi_prometheus_alerts.yml`

**Device Health Alerts:**
- `UniFiDeviceOffline` - Device has been offline for >3 minutes (critical)
- `UniFiDeviceHighCPU` - CPU usage >85% for 10 minutes (warning)
- `UniFiDeviceHighMemory` - Memory usage >90% for 10 minutes (warning)
- `UniFiDeviceRecentReboot` - Device uptime <10 minutes (info)

**Wireless Health Alerts:**
- `UniFiAPHighClientCount` - More than 30 clients on one AP (warning)
- `UniFiLowSignalClients` - 3+ clients with signal <-75 dBm (warning)
- `UniFiLowClientSatisfaction` - 2+ clients with satisfaction <50% (warning)

**Network Issues:**
- `UniFiHighDisconnectionRate` - High client roaming rate (warning)
- `UniFiDeviceTransmitErrors` - High packet drops on transmit (warning)
- `UniFiDeviceReceiveErrors` - High packet drops on receive (warning)

**Anomaly Detection:**
- `UniFiUnusualClientCount` - Client count 50% above 1h average (info)
- `UniFiRapidClientConnections` - 10+ new clients in 2 minutes (info)
- `UniFiClientRetryStorm` - High WiFi retry rate, interference suspected (warning)

#### Existing Loki Alerts (Already Configured)
**File**: `config/rules/loki_alerts.yml`

Includes SSH brute force, disk space, service crashes, RAID degradation, and more.

## Configuration Changes Made

### 1. docker-compose-filesystem.yml
Added `unifi-poller` service and mounted Prometheus rules directory:
```yaml
unifi-poller:
  container_name: unifi-poller
  image: golift/unifi-poller:latest
  ports:
    - 9130:9130
  volumes:
    - ./config/unifi-poller.yml:/etc/unifi-poller/up.yml:ro

prometheus:
  volumes:
    - ./config/rules:/etc/prometheus/rules:ro  # NEW
```

### 2. prometheus.yml
Added rule files configuration:
```yaml
rule_files:
  - "/etc/prometheus/rules/*.yml"
```

Added UniFi Poller scrape target:
```yaml
- job_name: 'unifi-poller'
  static_configs:
    - targets: ['unifi-poller:9130']
```

## How to Apply Changes

### Restart the Stack
```bash
cd /Users/enderx/Github/grafana-loki-syslog-aio
docker-compose -f docker-compose-filesystem.yml down
docker-compose -f docker-compose-filesystem.yml up -d
```

### Verify Services
```bash
# Check UniFi Poller is running and collecting metrics
curl http://localhost:9130/metrics | grep unifi_device

# Check Prometheus is scraping UniFi Poller
# Visit: http://localhost:9090/targets
# Look for "unifi-poller" - should show as UP

# Check alert rules are loaded
# Visit: http://localhost:9090/rules
```

### Access Dashboards
- UniFi Network Overview: http://localhost:3000/d/unifi_network_overview
- Security Monitoring: http://localhost:3000/d/security_monitoring

## Metrics Being Collected

UniFi Poller is collecting 1474 metrics every 30 seconds, including:

**Device Metrics:**
- Uptime, CPU, memory, temperature
- Network traffic (bytes/packets in/out)
- Load averages
- Device info (model, firmware, IP, MAC)

**Client Metrics:**
- Connected clients count
- Uptime per client
- WiFi signal strength (RSSI)
- Satisfaction scores
- Transmit/receive rates
- Roaming counts
- Packet retry counts

**Access Point Specific:**
- Client counts per radio
- Channel utilization
- Noise levels

## Troubleshooting

### UniFi Poller Not Collecting Data
```bash
# Check logs
docker logs unifi-poller

# Should see: "UniFi Measurements Exported. Site: 1, Client: 34, UAP: 2..."
```

### Dashboards Show No Data
1. Verify Prometheus is scraping:
   - Go to http://localhost:9090/targets
   - Check "unifi-poller" shows as UP

2. Test a query in Prometheus:
   - Go to http://localhost:9090
   - Run: `unifi_device_info`
   - Should show your devices

3. In Grafana, check datasource:
   - Settings → Data Sources → Prometheus
   - Click "Test" - should be successful

### Alerts Not Firing
1. Check Prometheus rules loaded:
   - Go to http://localhost:9090/rules
   - Should see "unifi_device_health", "unifi_wireless_health", etc.

2. Check alert evaluation:
   - Rules page shows "Pending" or "Firing" states
   - Check "for" duration hasn't been met yet

## Your Current Network

Based on metrics being collected:
- **1 Site**: Default
- **34 Clients**: Connected devices
- **2 Access Points**: U6 Mesh, U6 LR
- **1 Gateway**: UDM Pro (Home-Garcia)
- **2 Switches**: USW Flex, USW Lite 8 PoE

## Next Steps (Optional)

1. **Set up Alertmanager**: Configure email/Slack notifications for alerts
2. **Add more exporters**:
   - Pi-hole v6 Exporter for DNS metrics
   - NAS exporter for storage metrics
   - Proxmox exporter for VM/container metrics
3. **Fine-tune alert thresholds**: Adjust based on your environment
4. **Create custom views**: Build dashboards for specific needs

## Credentials

UniFi Controller user (read-only):
- Username: `unifipoller1828`
- Password: (stored in `config/unifi-poller.yml` and docker-compose environment)

## Files Modified/Created

**New Files:**
- `config/unifi-poller.yml`
- `config/grafana/dashboards/no_folder/unifi_network_overview.json`
- `config/grafana/dashboards/no_folder/security_monitoring.json`
- `config/rules/unifi_prometheus_alerts.yml`
- `UNIFI_POLLER_SETUP.md` (this file)

**Modified Files:**
- `docker-compose-filesystem.yml`
- `config/prometheus.yml`

**Removed Files:**
- Old dashboard with incorrect metrics (already deleted)
