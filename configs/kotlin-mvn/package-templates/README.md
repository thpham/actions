# Native Package Templates (DEB/RPM)

This directory contains example templates for creating native Linux packages (`.deb` and `.rpm`) for Spring Boot/Kotlin applications using **nfpm** (a lightweight Go-based packaging tool).

## Why nfpm?

nfpm is the recommended approach for packaging Spring Boot fat JARs because:

| Aspect          | nfpm                      | jpackage                          |
| --------------- | ------------------------- | --------------------------------- |
| JRE bundling    | NOT required              | REQUIRED (fundamental limitation) |
| Package size    | ~48MB (JAR only)          | ~150MB+ (includes JRE)            |
| Configuration   | Simple YAML               | Complex JReleaser config          |
| Dependencies    | Single Go binary          | Requires JDK 16+                  |
| Both deb/rpm    | Same config               | Same config                       |
| Spring Boot fit | Perfect (uses system JRE) | Overkill for fat JARs             |

Spring Boot fat JARs are self-contained and can run on any JRE. Bundling a JRE (as jpackage requires) is unnecessary overhead.

## Directory Structure

In your consumer repository (e.g., `gha-mvn-rflow`), create:

```
your-repo/
├── nfpm.yaml                         # nfpm package configuration
├── jreleaser.yml                     # JReleaser (SBOM disabled, release only)
└── packaging/                        # Package support files
    ├── myapp.service                 # Systemd unit file
    ├── myapp.conf                    # Environment defaults
    ├── application.yml               # Default app config
    └── scripts/
        ├── preinstall.sh             # Pre-install script
        ├── postinstall.sh            # Post-install script
        ├── preremove.sh              # Pre-remove script
        └── postremove.sh             # Post-remove script
```

## Template Files

| File                      | Description                                             |
| ------------------------- | ------------------------------------------------------- |
| `nfpm.yaml.example`       | nfpm configuration for both deb and rpm                 |
| `preinstall.sh.example`   | Pre-install script (creates user, directories)          |
| `postinstall.sh.example`  | Post-install script (sets permissions, enables service) |
| `preremove.sh.example`    | Pre-remove script (stops service)                       |
| `postremove.sh.example`   | Post-remove script (cleanup on purge)                   |
| `app.service.example`     | Systemd unit file with security hardening               |
| `app.conf.example`        | Environment variables for the service                   |
| `application.yml.example` | Default Spring Boot configuration                       |

## Usage

1. **Copy the templates** to your consumer repository
2. **Rename** `myapp` to your actual application name
3. **Customize** the nfpm.yaml for your project
4. **Enable** `build-packages: true` in your workflow call

## nfpm Configuration Highlights

```yaml
# nfpm.yaml
name: myapp
arch: all
platform: linux
version: "${VERSION}" # Set via environment variable
release: 1

contents:
  # Main JAR file
  - src: modules/api/target/api-${VERSION}.jar
    dst: /opt/myapp/myapp.jar
    file_info:
      mode: 0640
      owner: myapp
      group: myapp

  # Systemd service
  - src: packaging/myapp.service
    dst: /lib/systemd/system/myapp.service

  # Config files (preserved on upgrade)
  - src: packaging/myapp.conf
    dst: /etc/default/myapp
    type: config|noreplace

scripts:
  preinstall: packaging/scripts/preinstall.sh
  postinstall: packaging/scripts/postinstall.sh
  preremove: packaging/scripts/preremove.sh
  postremove: packaging/scripts/postremove.sh

deb:
  depends:
    - default-jre-headless (>= 17)

rpm:
  depends:
    - java-17-openjdk-headless
```

## Building Packages

### Locally

```bash
# Install nfpm
brew install goreleaser/tap/nfpm  # macOS
# or download from https://github.com/goreleaser/nfpm/releases

# Build packages
export VERSION=1.0.0
nfpm package --config nfpm.yaml --packager deb --target dist/
nfpm package --config nfpm.yaml --packager rpm --target dist/
```

### In GitHub Actions

Enable package building in your workflow call:

```yaml
uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
with:
  build-packages: true
  nfpm-config: "nfpm.yaml"
  # ... other inputs
```

## Package Installation

### Debian/Ubuntu

```bash
# Install
sudo dpkg -i myapp_1.0.0-1_amd64.deb
sudo apt-get install -f  # Fix dependencies

# Start service
sudo systemctl start myapp

# Remove
sudo apt remove myapp

# Purge (remove all config/data)
sudo apt purge myapp
```

### RHEL/Rocky/Alma

```bash
# Install
sudo dnf install ./myapp-1.0.0-1.x86_64.rpm

# Start service
sudo systemctl start myapp

# Remove
sudo dnf remove myapp
```

## Testing with Docker

```bash
# Test DEB in Ubuntu container
docker run -it --rm -v "$(pwd)/dist:/packages" ubuntu:22.04 bash
apt-get update && apt-get install -y openjdk-17-jre-headless
dpkg -i /packages/myapp_1.0.0-1_amd64.deb
dpkg -L myapp

# Test RPM in Rocky container
docker run -it --rm -v "$(pwd)/dist:/packages" rockylinux:9 bash
dnf install -y java-17-openjdk-headless
rpm -ivh /packages/myapp-1.0.0-1.x86_64.rpm
rpm -ql myapp
```

## See Also

- [nfpm Documentation](https://nfpm.goreleaser.com/)
- [nfpm Configuration Reference](https://nfpm.goreleaser.com/docs/configuration/)
- [Systemd Service Units](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
