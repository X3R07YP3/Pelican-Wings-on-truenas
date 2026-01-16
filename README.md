# Pelican Wings for TrueNAS Scale

[![License](https://img.shields.io/github/license/X3R07YP3/Pelican-Wings-on-truenas)](LICENSE)
[![GitHub last commit](https://img.shields.io/github/last-commit/X3R07YP3/Pelican-Wings-on-truenas)](https://github.com/X3R07YP3/Pelican-Wings-on-truenas/commits/main)

Complete guide and configuration files for running Pelican Wings on TrueNAS Scale using Docker Compose, with solutions for common bind mount issues.

## 📋 Overview

This repository contains everything needed to deploy Pelican Wings (game server manager) on TrueNAS Scale, including:

- Docker Compose configuration (`wing.yml`)
- Detailed installation guides in Spanish and English
- Solutions for common path mounting problems
- Best practices for directory structure and permissions

## 📚 Documentation

- 🇪🇸 [Guía en Español](./documentacion-wings-truenas.md) - Documentación completa en español
- 🇬🇧 [English Guide](./pelican-wings-truenas-en.md) - Complete documentation in English

## 🚀 Quick Start

### Requirements

- TrueNAS Scale
- Portainer (Only if you want to see the Wing logs, since Trunas doesn't work well for viewing these logs.)
- SSH access to the server
- Pelican Panel installed and configured
- Node created in the panel (provides credentials)

### Installation Steps

1. **Create ZFS Dataset**
   - In TrueNAS UI: Storage → Pools → Add Dataset
   - Name: `<your-Dataset>`
   - Configure compression and sharing options

2. **Create Directories**
   ```bash
   mkdir -p /mnt/<your-pool>/<your-Dataset>/wings/{varlib/volumes,logs,etc}
   mkdir -p /tmp/pelican
   ```

3. **Deploy Wings**
   ```bash
   docker compose -f wing.yml up -d
   ```

## 📁 Repository Structure

```
.
├── wing.yml                        # Main Docker Compose configuration
├── documentacion-wings-truenas.md  # Spanish documentation
├── pelican-wings-truenas-en.md     # English documentation
└── README.md                       # This file
```

## 🔧 Key Configuration

The critical part of this setup are the **path-through mounts** that solve bind mount issues:

```yaml
volumes:
  # Critical path-through mounts
  - /mnt/<pool>/<your-Dataset>/wings/varlib/volumes:/mnt/<pool>/<your-Dataset>/wings/varlib/volumes
  - /tmp/pelican:/tmp/pelican
```

These mounts ensure that when Wings creates directories inside its container, they're accessible from the host for subsequent container creation.

## 🛠️ Troubleshooting

### Common Issues

1. **"bind source path does not exist"**
   - Ensure all required directories exist on the host
   - Verify path-through mounts are configured correctly

2. **Connection refused to panel**
   - Check `remote:` URL in configuration
   - Verify network connectivity and firewall rules

3. **Servers fail to start**
   - Confirm `wings0` Docker network exists
   - Check directory permissions

### Need Help?

Check the full documentation for detailed troubleshooting steps.

## ⚠️ Important Notes

- `/tmp/pelican` is cleared on system reboot (normal behavior)
- ZFS dataset `<your-Dataset>` provides better performance and snapshots
- All sensitive credentials are replaced with placeholders in documentation

## 🤝 Contributing

Contributions are welcome! Feel free to:
- Report issues
- Suggest improvements
- Translate documentation
- Share your deployment experiences

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Pelican Panel](https://pelican.dev/) team for the amazing software
- TrueNAS community for Docker integration support


---

*Made with ❤️ for the gaming server community*
