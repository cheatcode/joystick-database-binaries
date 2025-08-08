# Joystick Database Binaries

This project provides automated build systems for creating portable database binaries for macOS and Linux across multiple architectures (x86_64 and arm64). The binaries are built with portable configurations and uploaded to Cloudflare R2 storage for distribution.

## Supported Databases

- **PostgreSQL 17.5** (manual builds)
- **MongoDB 8.0.12** (automated via GitHub Actions)
- **Redis 7.2.5** (automated via GitHub Actions)

## Project Structure

```
.
├── .github/workflows/
│   └── upload-binaries.yml          # GitHub Actions workflow for MongoDB/Redis
├── manual/
│   └── postgresql/
│       ├── build_macos              # macOS build script
│       └── build_linux              # Linux build script
├── config.json                      # R2 credentials (not in repo)
└── README.md
```

## Configuration

### Required Configuration File

Create a `config.json` file in the project root directory:

```json
{
  "R2_ACCESS_KEY_ID": "your-r2-access-key",
  "R2_SECRET_ACCESS_KEY": "your-r2-secret-key",
  "R2_BUCKET": "your-bucket-name",
  "R2_ENDPOINT": "https://your-account-id.r2.cloudflarestorage.com"
}
```

## PostgreSQL Manual Builds

PostgreSQL builds are handled manually due to the complexity of creating truly portable binaries with proper library linking.

### macOS Build (`manual/postgresql/build_macos`)

**Features:**
- Cross-compilation support for both arm64 and x86_64
- Portable rpath configuration using `@loader_path`
- Automatic Homebrew installation if missing
- Library path fixing for portability
- Verification of portable paths

**Usage:**
```bash
cd manual/postgresql
./build_macos arm64     # Build for Apple Silicon
./build_macos x86_64    # Build for Intel Macs
```

**Build Process:**
1. Downloads PostgreSQL 17.5 source
2. Configures with portable settings (no readline, zlib, openssl, icu)
3. Builds with architecture-specific flags
4. Fixes library paths using `install_name_tool`
5. Verifies portable rpath configuration
6. Creates tarball and uploads to R2

### Linux Build (`manual/postgresql/build_linux`)

**Features:**
- Cross-compilation support for arm64 and x86_64
- RPATH configuration using `$ORIGIN/../lib`
- Shared library bundling
- Binary stripping for size optimization
- Dependency verification

**Usage:**
```bash
cd manual/postgresql
./build_linux arm64     # Build for ARM64 Linux
./build_linux x86_64    # Build for x86_64 Linux
```

**Build Process:**
1. Installs required packages and cross-compilation tools
2. Downloads PostgreSQL 17.5 source
3. Configures with shared libraries and portable RPATH
4. Builds and installs to temporary directory
5. Copies shared libraries and sets RPATH
6. Strips binaries for size reduction
7. Creates tarball and uploads to R2

## Automated Builds (GitHub Actions)

MongoDB and Redis builds are automated through GitHub Actions workflow.

### Triggering Builds

```bash
# Manually trigger the workflow
gh workflow run "Build and Upload DB Binaries to R2"
```

### Build Matrix

The workflow builds for:
- **Platforms:** macOS, Linux
- **Architectures:** x86_64, arm64
- **Databases:** MongoDB 8.0.12, Redis 7.2.5

## Upgrading Database Versions

### PostgreSQL

1. Update the `VERSION` variable in both build scripts:
   ```bash
   VERSION="17.6"  # Update to new version
   ```

2. Update the `MAJOR_VERSION` calculation if needed (for major version changes)

3. Run the build scripts on target machines:
   ```bash
   cd manual/postgresql
   ./build_macos arm64
   ./build_macos x86_64
   ./build_linux arm64
   ./build_linux x86_64
   ```

### MongoDB/Redis

1. Update the `db_version` in `.github/workflows/upload-binaries.yml`:
   ```yaml
   - database: mongodb
     db_version: 8.0.13  # Update version
   ```

2. Update download URLs if the MongoDB/Redis release structure changes

3. Trigger the GitHub Actions workflow

## Build Output Structure

Binaries are uploaded to R2 with the following structure:

```
s3://bucket/
├── postgresql/
│   └── 17/
│       ├── macos/
│       │   ├── arm64.tar.gz
│       │   └── x86_64.tar.gz
│       └── linux/
│           ├── arm64.tar.gz
│           └── x86_64.tar.gz
├── mongodb/
│   └── 8/
│       ├── macos/
│       │   ├── arm64.tar.gz
│       │   └── x86_64.tar.gz
│       └── linux/
│           ├── arm64.tar.gz
│           └── x86_64.tar.gz
└── redis/
    └── 7/
        ├── macos/
        │   ├── arm64.tar.gz
        │   └── x86_64.tar.gz
        └── linux/
            ├── arm64.tar.gz
            └── x86_64.tar.gz
```

## Portability Features

### macOS
- Uses `@loader_path` for relative library paths
- Fixes install names with `install_name_tool`
- Sets minimum macOS versions (11.0 for arm64, 10.15 for x86_64)
- Verifies portable paths with `otool`

### Linux
- Uses `$ORIGIN/../lib` RPATH for relative library paths
- Bundles required shared libraries
- Sets RPATH with `patchelf` or `chrpath`
- Strips binaries to reduce size
- Verifies dependencies with `ldd` and `readelf`

## Prerequisites

### macOS
- Xcode Command Line Tools
- Homebrew (auto-installed if missing)

### Linux
- build-essential
- Development libraries (zlib1g-dev, libreadline-dev, etc.)
- Cross-compilation tools (for cross-arch builds)
- patchelf or chrpath for RPATH manipulation

### Both Platforms
- curl
- jq
- AWS CLI (for R2 uploads)

## Troubleshooting

### Build Failures
- Check that all prerequisites are installed
- Verify `config.json` exists and has correct R2 credentials
- Ensure sufficient disk space for builds
- Check network connectivity for downloads

### Portability Issues
- Verify RPATH settings with `otool -L` (macOS) or `readelf -d` (Linux)
- Check that all required libraries are bundled
- Test binaries on clean systems without development tools

### Upload Failures
- Verify R2 credentials and endpoint URL
- Check bucket permissions
- Ensure AWS CLI is properly configured

## Security Notes

- `config.json` is excluded from version control
- R2 credentials should be kept secure
- GitHub Actions uses encrypted secrets for credentials
- Build scripts use `set -euo pipefail` for error handling

## Future Improvements

- Add automated testing of built binaries
- Implement checksum verification
- Add support for additional PostgreSQL extensions
- Create unified build script for all databases
- Add Docker-based builds for better reproducibility
