# OPNsense Single WAN High-Availability Failover

A production-ready, active/passive OPNsense cluster solution that shares a single public IP address (static or DHCP). This system prevents split-brain conditions and IP conflicts through a multi-script architecture with comprehensive logging and safety mechanisms.

## 🚀 Features

- **Single WAN IP Sharing**: Supports both static IP and DHCP configurations
- **Active/Passive Clustering**: Prevents split-brain scenarios with intelligent failover
- **Service Management**: Automatic start/stop of critical services during transitions
- **Health Monitoring**: Multi-stage health checks with configurable retry logic
- **Circuit Breaker**: Prevents rapid failover loops with failure tracking
- **Structured Logging**: JSON-formatted logs for modern observability
- **Dry Run Support**: Safe testing without system changes
- **IPv6 Support**: Optional IPv6 tunnel management

## 📜 Project History

This project builds upon earlier single-script approaches to OPNsense high availability:

**Original Foundation**: [spali's single script solution](https://gist.github.com/spali/2da4f23e488219504b2ada12ac59a7dc) provided the initial framework for CARP-based failover with basic WAN IP management. This pioneering work demonstrated the feasibility of automated failover for single WAN IP scenarios.

**Iterative Development**: [lavacano's enhanced versions](https://gist.github.com/lavacano/a678e65d31df9bec344e572461ed3e10) expanded on the original concept with improved error handling, additional service management, and better logging capabilities. These iterations identified key pain points and reliability issues in production environments.

**MSZ345 Solution (base of this fork)**: This multi-script architecture represents a complete redesign addressing the limitations discovered in single-script approaches:

- **Separation of Concerns**: Distinct scripts for different phases (boot enforcement, live failover, route management)
- **Production Hardening**: Comprehensive error handling, circuit breakers, and failure tracking
- **Operational Excellence**: Structured logging, dry-run testing, and configuration validation
- **Maintainability**: Centralized configuration and modular design for easier customization

The evolution from single-script to multi-script architecture reflects lessons learned from real-world deployments and the need for enterprise-grade reliability in critical network infrastructure.

**Current Solution**: Script was not working for me on OPNsense 26.7.2_2. Main problem was config save function that maybe changed in OPNsense since then. But I did not check the Changenlogs. I'm not a programmer at all. I never touched php code before. Debugging was done with the support auf claude and chatgpt. All code changes where manually done by me. I also added the IPv6 idea of the new gateway enable/disable logic, but this is untested, because I hoave no IPv6 setup. I don't know if the new gateway logic is better or not, but it works for my setup.

## 📋 Prerequisites

### Hardware Requirements

- Two identical OPNsense firewalls
- Dedicated sync interface between firewalls
- Shared network segments for WAN and LAN
- **Recommended**: Configure WAN interfaces with identical MAC addresses for seamless failover

### OPNsense Configuration

Before installing the scripts, configure your firewalls:

#### Primary Firewall

1. Go to **System → High Availability → Settings**
   - Enable HA by checking "Synchronize States"
   - Set the Synchronize Interface (dedicated SYNC interface)
   - Set Synchronize Peer IP to secondary firewall's SYNC IP

2. Go to **System → Settings → Tunables**
   - Create tunable: `net.inet.carp.preempt` = `1`

#### Secondary Firewall

1. Configure HA Sync settings pointing to primary firewall
2. Go to **System → High Availability → Settings**
   - Check "Disable Preemptive Mode"

#### Both Firewalls

1. Set up CARP VIPs on WAN and LAN interfaces
   - Use unique VHID for each interface
   - Primary: advskew = 0, Secondary: advskew = 100
2. Configure **System → High Availability → Settings** to sync from primary to secondary
3. **Optional but Recommended**: Configure identical MAC addresses on WAN interfaces
   - Go to **Interfaces → [WAN] → General configuration**
   - Set the same MAC address on both firewalls' WAN interfaces
   - This ensures seamless failover without ARP table updates on upstream equipment

## 🛠️ Installation

### Step 1: Download Configuration Files

Clone this repository or download the configuration files:

```bash
# Create necessary directories
mkdir -p /usr/local/etc/rc.syshook.d/carp
mkdir -p /usr/local/etc/rc.d
```

### Step 2: Install Configuration Files

Copy the following files to both firewalls:

| File                        | Location                            | Purpose                 |
| --------------------------- | ----------------------------------- | ----------------------- |
| `ha_failover.conf`          | `/usr/local/etc/`                   | Central configuration   |
| `validate_ha_config.php`    | `/usr/local/etc/`                   | Configuration validator |
| `10-failover.php`           | `/usr/local/etc/rc.syshook.d/carp/` | Main failover logic     |
| `99-ha_passive_enforcer.sh` | `/usr/local/etc/rc.d/`              | Boot-time enforcer      |

### Step 3: Set Permissions

```bash
# Set secure permissions
chmod 600 /usr/local/etc/ha_failover.conf
chmod +x /usr/local/etc/validate_ha_config.php
chmod +x /usr/local/etc/rc.syshook.d/carp/10-failover.php
chmod +x /usr/local/etc/rc.d/99-ha_passive_enforcer.sh
```

### Step 4: Enable Boot Service

```bash
echo 'ha_passive_enforcer_enable="YES"' >> /etc/rc.conf.local
```

## ⚙️ Configuration

### Basic Configuration

Edit `/usr/local/etc/ha_failover.conf` for your environment:

#### Static IP Configuration

```json
{
  "interfaces": {
    "wan_key": "wan",
    "tunnel_key": "opt1"
  },
  "network": {
    "wan_mode": "static",
    "wan_ipv4": "203.0.113.100",
    "wan_subnet_v4": 27,
    "wan_gateway_name": "WAN_GW",
    "tunnel_gateway_name": "TUNNELGW"
  }
}
```

#### DHCP Configuration

```json
{
  "interfaces": {
    "wan_key": "wan",
    "tunnel_key": "opt1"
  },
  "network": {
    "wan_mode": "dhcp",
    "wan_gateway_name": "WAN_DHCP_GW"
  }
}
```

#### PPPoE Configuration

```json
{
  "interfaces": {
    "wan_key": "wan",
    "tunnel_key": "opt1"
  },
  "network": {
    "wan_mode": "pppoe",
    "wan_gateway_name": "WAN_GW"
  }
}
```

#### For IPv6-less Setups

Remove these keys from the configuration:

- `tunnel_key`
- `tunnel_gateway_name`
- `target_v6`

### Key Configuration Sections

#### Health Check Configuration

```json
"health_check": {
  "local_target": "192.168.1.1",
  "target": ["1.1.1.1", "8.8.8.8"],
  "target_v6": ["2001:4860:4860::8888"],
  "ping_timeout": 2,
  "require_external_connectivity": false
}
```

#### Service Management

If you use Free Range Routing then add tit as core service.

```json
"ha_core_services": [
  {
    "name": "frr",
    "pid_file": "/var/run/frr/frr.pid",
    "shutdown_timeout": 20
  }
```

Change the DHCP and DNS, if you dont use ISC DHCPv4 Server or Unbound DNS. Add other services that also need to stop in BACKUP mode.

```json
"ha_controlled_services": [
  {
    "name": "dhcpd",
    "pid_file": "/var/dhcpd/var/run/dhcpd.pid"
  },
  {
    "name": "unbound",
    "pid_file": "/var/run/unbound.pid",
    "shutdown_timeout": 15
  }
]
```

#### Timeouts

```json
"timeouts": {
  "master_transition_delay": 20,
  "event_cooldown_period": 600,
  "passive_enforcer_delay": 20,
  "route_settle_delay": 3,
  "lock_wait_timeout": 90,
  "health_check_retries": 3,
  "health_check_retry_delay": 5,
  "service_verify_timeout": 15
}
```

| Timeout                    | Min | Max  | Default |
| -------------------------- | --- | ---- | ------- |
| `master_transition_delay`  | 0   | 300  | 20      |
| `event_cooldown_period"`   | 30  | 3600 | 600     |
| `passive_enforcer_delay`   |     |      | 20      |
| `route_settle_delay`       | 0   | 60   | 2       |
| `lock_wait_timeout`        | 10  | 300  | 60      |
| `health_check_retries`     | 1   | 5    | 3       |
| `health_check_retry_delay` | 1   | 20   | 5       |
| `service_verify_timeout`   | 5   | 60   | 15      |

## 🧪 Testing

### Validate Configuration

```bash
php /usr/local/etc/validate_ha_config.php
```

### Dry Run Testing

Test failover logic without making changes:

```bash
# Simulate MASTER transition
php /usr/local/etc/rc.syshook.d/carp/10-failover.php carp MASTER dry-run

# Simulate BACKUP transition
php /usr/local/etc/rc.syshook.d/carp/10-failover.php carp BACKUP dry-run

# Test boot enforcer
sh /usr/local/etc/rc.d/99-ha_passive_enforcer.sh dry-run
```

### Live Testing

1. **Boot Test**: Reboot secondary firewall, verify it stays in BACKUP state
2. **Failover Test**: Put primary in maintenance mode, verify secondary becomes MASTER
3. **Failback Test**: Remove maintenance mode, verify primary reclaims MASTER

## 📊 Monitoring

### Log Locations

- **Failover Events**: `/var/log/system/latest.log` (search for `ha_failover`)
- **Boot Enforcer**: `/var/log/ha_enforcer.log`
- **CARP Status**: **Lobby → Dashboard → CARP**

### Log Format

All logs use structured JSON format:

```json
{
  "timestamp": "2024-01-15T10:30:00+00:00",
  "event": "master_transition_complete",
  "pid": 12345,
  "context": { "transition_time": 25 }
}
```

## 🔧 Troubleshooting

### Common Issues

**Split-brain condition detected**

- Check CARP sync interface connectivity
- Verify sync interface configuration matches on both nodes

**Services not starting/stopping**

- Verify service names in configuration match OPNsense service names
- Check PID file paths are correct
- Review service-specific logs

**Health checks failing**

- Verify target IPs are reachable
- Check ping timeouts are appropriate for your network
- Review external connectivity requirements

### Debug Commands

```bash
# Check CARP status
ifconfig | grep carp

# View current routes
netstat -rn

# Check service status
service <service_name> status

# Manual service control
php /usr/local/etc/rc.syshook.d/carp/10-failover.php carp MASTER
```

## 🔒 Security Considerations

- Configuration file has restrictive permissions (600)
- All network operations are validated
- Service control uses proper escaping
- Lock files prevent concurrent execution
- Circuit breaker prevents failover loops

## 📈 Architecture

```
┌─────────────────┐    ┌─────────────────┐
│   Primary FW    │    │  Secondary FW   │
│   (MASTER)      │◄──►│   (BACKUP)      │
│                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │   WAN IP    │ │    │ │  Services   │ │
│ │  Services   │ │    │ │  Stopped    │ │
│ │  Running    │ │    │ │             │ │
│ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘
         │                       │
         └───────── LAN ─────────┘
```

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Test thoroughly with dry-run mode
4. Submit a pull request with detailed description

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🆘 Support

For issues and questions:

1. Check the troubleshooting section
2. Review log files for error details
3. Open an issue with configuration and log excerpts
4. Include OPNsense version and hardware details

---

**⚠️ Important**: Always test in a non-production environment first. Use dry-run mode extensively before deploying to production systems.
