# SHIELD Deployment Guide: Building and Deploying with Genesis

This guide explains how the three components in this repository work together to deploy SHIELD, and provides step-by-step instructions for building from source and deploying with Genesis.

## Understanding the Components

### 1. **shield/** - The SHIELD Source Code
This directory contains the SHIELD application source code written in Go. It includes:
- Core SHIELD daemon (`shieldd`)
- SHIELD agent (`shield-agent`)
- CLI tool (`shield`)
- Various plugins for backing up different data stores (PostgreSQL, MySQL, Redis, etc.)
- Storage plugins for different backends (S3, Azure, WebDAV, etc.)

**Purpose**: Build the actual SHIELD binaries that will run on your VMs.

### 2. **shield-boshrelease/** - The BOSH Release
This is a BOSH release that packages SHIELD for deployment via BOSH. It contains:
- Job definitions (how SHIELD runs on VMs)
- Package specifications (dependencies and compilation instructions)
- Release metadata and versioning
- Source code references (usually as git submodules or vendored code)

**Purpose**: Package SHIELD into a BOSH-deployable format with all dependencies, configuration templates, and lifecycle scripts.

### 3. **shield-genesis-kit/** - The Genesis Kit
This is a Genesis deployment kit that provides:
- Base manifest templates
- Feature flags for different deployment configurations
- Environment-specific parameter definitions
- Hooks for deployment automation
- Cloud-provider-specific configurations

**Purpose**: Provide a user-friendly, opinionated way to deploy SHIELD using Genesis, which generates the final BOSH manifest.

## How They Work Together

```
┌─────────────────┐
│  shield/        │  1. Build SHIELD binaries from source
│  (Source Code)  │     (Go compilation)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ shield-         │  2. Create BOSH release with compiled binaries
│ boshrelease/    │     (BOSH packaging)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ shield-genesis- │  3. Generate deployment manifest using Genesis
│ kit/            │     (Genesis templating)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  BOSH Director  │  4. Deploy to infrastructure
│                 │     (BOSH orchestration)
└─────────────────┘
```

## Prerequisites

Before you begin, ensure you have:

1. **Development Tools** (for building SHIELD):
   - Go 1.21+ installed
   - Make
   - Git

2. **BOSH Tools**:
   - BOSH CLI v7+ installed
   - Access to a BOSH Director
   - Appropriate cloud config set up on your BOSH Director

3. **Genesis** (v3.1.0+):
   - Genesis CLI installed
   - Vault for secrets management
   - Access to your target environment

4. **Infrastructure**:
   - Cloud provider credentials (AWS, vSphere, GCP, etc.)
   - Network and VM resources available in your BOSH cloud-config

## Step-by-Step Deployment Process

### Phase 1: Build SHIELD from Source

```bash
# Navigate to the SHIELD source directory
cd shield/

# Set the version for your build
export VERSION=8.7.0  # Use the version you're building

# Build all SHIELD components
make release

# This creates binaries in artifacts/ directory:
# - artifacts/shield-server-linux-amd64/
#   - daemon/shieldd
#   - daemon/shield-schema
#   - agent/shield-agent
#   - agent/shield-report
#   - plugins/*
#   - webui/
```

**What this does**: Compiles all SHIELD components (daemon, agent, CLI, plugins) for Linux AMD64 architecture and packages them with the web UI.

### Phase 2: Create a BOSH Release

Now you'll incorporate your built binaries into a BOSH release.

```bash
# Navigate to the BOSH release directory
cd ../shield-boshrelease/

# Option A: Create a dev release (for testing)
# This creates a local, non-finalized release
bosh create-release --name=shield --force

# Option B: Create a final release (for production)
# This creates a versioned, finalized release
bosh create-release --name=shield --version=8.7.0 --final --force

# Upload the release to your BOSH Director
bosh upload-release
```

**Important Notes**:
- **Dev releases** are great for iteration and testing but aren't suitable for production
- **Final releases** are versioned, tracked, and suitable for production
- You may need to update the `src/` directory in the BOSH release to include your built artifacts
- Check `packages/shield/spec` to understand how the release expects sources to be structured

### Phase 3: Prepare Genesis Deployment

```bash
# Navigate to a directory where you keep your deployments
cd ~/deployments/

# Initialize a new SHIELD deployment repository using Genesis
genesis init -k shield shield-deployments

# Or, if you want to use your local Genesis kit:
cd ../shield-genesis-kit/
genesis init -k . ~/deployments/shield-deployments

# Navigate to your deployment repo
cd ~/deployments/shield-deployments/
```

### Phase 4: Create an Environment

```bash
# Create a new environment (e.g., 'prod')
genesis new prod

# This will prompt you for:
# - BOSH environment name
# - Vault path
# - IP addresses
# - Cloud provider details
# - Features to enable
```

**Example environment file** (`prod.yml`):
```yaml
---
kit:
  name:    shield
  version: 2.0.0
  features:
    - secure        # Auto-generate admin credentials
    - postgres-addon  # Use PostgreSQL instead of SQLite

genesis:
  env: prod

params:
  # Network Configuration
  shield_static_ip: 10.0.1.50
  shield_network:   shield-net
  shield_disk_pool: shield-disks
  shield_vm_type:   medium
  
  # Optional: Custom domain
  external_domain: shield.mycompany.com
  
  # PostgreSQL addon version
  postgres-addon-version: 11
  
  # Certificate validity
  ca_validity_period: 3650    # 10 years
  cert_validity_period: 365   # 1 year
```

### Phase 5: Use Your Local BOSH Release

To use your locally built BOSH release instead of a public one, you need to modify the Genesis kit's manifest to reference your uploaded release.

**Option A: Modify the base manifest**

Edit `shield-genesis-kit/manifests/shield.yml`:

```yaml
releases:
- name:    shield
  version: 8.7.0  # Your version
  # Remove or comment out the 'url' and 'sha1' fields
  # BOSH will use the uploaded release from the director
```

**Option B: Use an environment-specific override**

Create `prod-local.yml` in your deployment repo:

```yaml
releases:
- name:    shield
  version: 8.7.0  # Your custom version
```

Then deploy with:
```bash
genesis deploy prod -y prod-local.yml
```

### Phase 6: Deploy!

```bash
# Generate secrets (if using 'secure' feature)
genesis add-secrets prod

# Review the generated manifest (optional)
genesis manifest prod > manifest.yml
cat manifest.yml

# Deploy to BOSH
genesis deploy prod

# Watch the deployment
bosh -e <env-name> -d prod-shield tasks --recent=1
```

## Iterative Development Workflow

When making changes and testing:

```bash
# 1. Make code changes in shield/
cd shield/
# ... edit code ...

# 2. Rebuild
make release

# 3. Create new dev release
cd ../shield-boshrelease/
bosh create-release --force

# 4. Upload to BOSH
bosh upload-release

# 5. Redeploy
cd ~/deployments/shield-deployments/
genesis deploy prod
```

## Common Deployment Scenarios

### Scenario 1: Testing with Local Changes

```bash
# Build a dev release with timestamp
cd shield-boshrelease/
bosh create-release --force --timestamp-version

# This creates something like: shield-8.7.0+dev.1.234567890
# Upload and deploy as usual
```

### Scenario 2: Air-Gapped Deployment

```bash
# 1. Build release tarball
bosh create-release --final --tarball=/tmp/shield-8.7.0.tgz

# 2. Transfer to air-gapped environment
scp /tmp/shield-8.7.0.tgz user@jumpbox:/tmp/

# 3. On the air-gapped BOSH Director
bosh upload-release /tmp/shield-8.7.0.tgz

# 4. Deploy using Genesis (Genesis kit may need to be transferred too)
```

### Scenario 3: Multi-Environment Deployment

```bash
# Create multiple environments
genesis new dev
genesis new staging  
genesis new prod

# Each can have different configurations
# Deploy them independently
genesis deploy dev
genesis deploy staging
genesis deploy prod
```

## Understanding BOSH Release Structure

To properly integrate your built binaries, understand the BOSH release structure:

```
shield-boshrelease/
├── jobs/
│   ├── core/          # SHIELD daemon job
│   ├── shield-agent/  # SHIELD agent job
│   └── store/         # WebDAV storage job
├── packages/
│   ├── shield/        # Main SHIELD package
│   ├── golang/        # Go compiler (if building from source)
│   └── ...
├── src/               # Source code (often git submodules)
└── config/
    └── blobs.yml      # External dependencies
```

### Integrating Your Built Binaries

You have two options:

**Option 1: Replace source and rebuild**
- Copy your `artifacts/` to `src/shield/artifacts/`
- Update `packages/shield/packaging` script to use pre-built artifacts
- Create release as usual

**Option 2: Use blobs**
- Create a tarball of your artifacts
- Add it as a blob: `bosh add-blob artifacts.tar.gz shield/artifacts.tar.gz`
- Upload blobs: `bosh upload-blobs`
- Update packaging script to extract from blob

## Troubleshooting

### Issue: BOSH can't find the release

```bash
# List releases on director
bosh releases

# Re-upload if needed
bosh upload-release --fix
```

### Issue: Deployment fails due to missing stemcell

```bash
# Check required stemcell version in manifest
genesis manifest prod | grep stemcell

# Upload stemcell
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-<iaas>-<os>-go_agent?v=<version>
```

### Issue: Genesis can't find the kit

```bash
# Use absolute path to local kit
genesis init -k /path/to/shield-genesis-kit ~/deployments/shield-deployments

# Or compile the kit
cd shield-genesis-kit/
genesis compile-kit --name shield --version 2.0.0-dev
# Then use the compiled kit
```

### Issue: Compilation fails in BOSH release

Check the compilation logs:
```bash
bosh tasks --recent=5
bosh task <task-id> --debug
```

Common issues:
- Missing dependencies in `packages/*/spec`
- Incorrect paths in `packages/*/packaging` scripts
- Architecture mismatches (ARM vs AMD64)

## Advanced: Building a Complete Workflow

Here's a complete script for automated builds:

```bash
#!/bin/bash
set -e

VERSION=${1:-8.7.0}
BOSH_ENV=${2:-my-bosh}
DEPLOYMENT=${3:-prod}

echo "Building SHIELD version $VERSION"

# Build from source
cd shield/
export VERSION=$VERSION
make release

# Create BOSH release
cd ../shield-boshrelease/
bosh -e $BOSH_ENV create-release \
  --name=shield \
  --version=$VERSION \
  --final \
  --force

bosh -e $BOSH_ENV upload-release

# Deploy with Genesis
cd ~/deployments/shield-deployments/
genesis deploy $DEPLOYMENT

echo "Deployment complete!"
echo "Access SHIELD at: https://$(genesis lookup $DEPLOYMENT params.shield_static_ip)"
```

## Additional Resources

- **BOSH Documentation**: https://bosh.io/docs/
- **SHIELD GitHub**: https://github.com/shieldproject/shield
- **Genesis Kit Development**: https://github.com/genesis-community/shield-genesis-kit

## Quick Reference Commands

```bash
# Build SHIELD
cd shield/ && make release

# Create BOSH release
cd shield-boshrelease/ && bosh create-release --force

# Upload release
bosh upload-release

# Init Genesis deployment
genesis init -k shield shield-deployments

# Create environment
genesis new <env-name>

# Generate secrets
genesis add-secrets <env-name>

# Deploy
genesis deploy <env-name>

# Check status
bosh vms
bosh instances --ps

# Access SHIELD
genesis do <env-name> -- visit
```

## Next Steps

After successful deployment:

1. **Access SHIELD**: `genesis do prod -- visit` (opens web UI)
2. **Get admin credentials**: `genesis lookup prod params.admin_username` and `genesis lookup prod secret/prod/shield/failsafe:admin_password`
3. **Configure backups**: Use the SHIELD web UI or CLI to set up stores, targets, and jobs
4. **Deploy agents**: Use the `runtime-config` addon to deploy agents across your infrastructure

Good luck with your SHIELD deployment!
