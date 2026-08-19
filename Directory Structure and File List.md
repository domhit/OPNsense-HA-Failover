# Repository Structure

```
opnsense-ha-failover/
├── README.md                           # Main documentation
├── INSTALLATION.md                     # Detailed installation guide
├── LICENSE                             # MIT License
├── scripts/                           # Main script files
│   ├── 10-failover.php                # Main failover logic
│   ├── 99-ha_passive_enforcer.sh      # Boot-time enforcer
│   └── validate_ha_config.php         # Configuration validator
├── config/                            # Configuration files
│   ├── ha_failover.conf               # Example configuration
```

## File Descriptions

### Core Script Files

| File | Location | Purpose | Permissions |
|------|----------|---------|-------------|
| `10-failover.php` | `/usr/local/etc/rc.syshook.d/carp/` | Main CARP event handler | `755` |
| `99-ha_passive_enforcer.sh` | `/usr/local/etc/rc.d/` | Boot-time backup node enforcer | `755` |
| `validate_ha_config.php` | `/usr/local/etc/` | Configuration validator | `755` |
| `ha_failover.conf` | `/usr/local/etc/` | Central configuration file | `600` |

### Installation Locations

```bash
# Main configuration (secure permissions)
/usr/local/etc/ha_failover.conf                    # 600 (-rw-------)

# Validation utility  
/usr/local/etc/validate_ha_config.php              # 755 (-rwxr-xr-x)

# CARP event handler
/usr/local/etc/rc.syshook.d/carp/10-failover.php   # 755 (-rwxr-xr-x)

# Boot-time service
/usr/local/etc/rc.d/99-ha_passive_enforcer.sh      # 755 (-rwxr-xr-x)

# Runtime files (created automatically)
/tmp/carp_failover.lock                             # Lock file
/tmp/carp_failover.state                            # State tracking
/tmp/carp_failover.failures                         # Failure counter
/var/log/ha_enforcer.log                           # Boot enforcer log
```

### Repository File Manifest

#### Documentation
- `README.md` - Main project documentation and quick start
- `INSTALLATION.md` - Detailed step-by-step installation guide

#### Configuration Templates
- `config/ha_failover.conf` - Complete working example

#### Core Scripts
- `scripts/10-failover.php` - Main failover logic (15.6KB)
- `scripts/99-ha_passive_enforcer.sh` - Boot enforcer (3.8KB)
- `scripts/validate_ha_config.php` - Config validator (2.1KB)

### File Dependencies

```
ha_failover.conf
├── 10-failover.php (reads config)
├── 99-ha_passive_enforcer.sh (reads config)
└── validate_ha_config.php (validates config)

10-failover.php
├── Requires: config.inc, interfaces.inc, util.inc, system.inc
├── Creates: /tmp/carp_failover.* files
└── Logs to: syslog (LOG_LOCAL4)

99-ha_passive_enforcer.sh
├── Reads: ha_failover.conf
├── Logs to: /var/log/ha_enforcer.log
└── Enabled by: /etc/rc.conf.local
```

### Checksum Verification

After installation, verify file integrity:

```bash
#!/bin/bash
# Generate checksums for verification
echo "Generating checksums for HA Failover files..."

md5sum /usr/local/etc/ha_failover.conf > /tmp/ha_checksums.txt
md5sum /usr/local/etc/validate_ha_config.php >> /tmp/ha_checksums.txt  
md5sum /usr/local/etc/rc.syshook.d/carp/10-failover.php >> /tmp/ha_checksums.txt
md5sum /usr/local/etc/rc.d/99-ha_passive_enforcer.sh >> /tmp/ha_checksums.txt

echo "Checksums saved to /tmp/ha_checksums.txt"
echo "Compare checksums between primary and backup firewalls"
```

### Version Information

- **Current Version**: 15.6.1 (Production Release)
- **PHP Version Required**: 7.4+ (OPNsense native)
- **OPNsense Version**: 26.7.2+ (tested)
- **Dependencies**: OPNsense Core, CARP, jq (for shell scripts)

### File Size Reference

| File | Approximate Size |
|------|------------------|
| 10-failover.php | ~35.8 KB |
| 99-ha_passive_enforcer.sh | ~4.27 KB |
| validate_ha_config.php | ~2.1 KB |
| ha_failover.conf | ~1.4 KB |
| **Total** | **~44 KB** |
