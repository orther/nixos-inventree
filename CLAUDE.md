# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository provides a NixOS module for running [InvenTree](https://inventree.org/) (an open-source inventory management system) as a native NixOS service. The module packages InvenTree's Python backend and frontend, creates systemd services, and provides a declarative NixOS configuration interface.

**Current InvenTree Version**: See `pkgs/src.nix` for the current version
**NixOS Compatibility**: nixos-unstable (tested)
**Last Updated**: 2025-11-11

> **Note**: When updating InvenTree versions, remember to update the "Last Updated" date above.

## Architecture

### Package Structure

The flake exposes several packages under `pkgs.inventree.*`:

- **src** (`pkgs/src.nix`): Fetches InvenTree source from GitHub, pre-built frontend from releases, patches deprecated Django calls, and builds static files. Includes a patch to disable filesystem mutation tasks.
- **server** (`pkgs/server.nix`): Wrapper script that runs gunicorn WSGI server
- **cluster** (`pkgs/cluster.nix`): Wrapper script that runs Django Q2 background worker (`manage.py qcluster`)
- **invoke** (`pkgs/invoke.nix`): Wrapper for InvenTree's invoke tasks (used for migrations, etc.)
- **python** (`pkgs/python.nix`): Python interpreter with all InvenTree dependencies
- **refresh-users** (`pkgs/refresh-users.nix`): Script to declaratively manage InvenTree users from NixOS config
- **gen-secret** (`pkgs/gen-secret.nix`): Utility to generate Django secret keys
- **shell** (`pkgs/shell.nix`): Development shell
- **venv**: Virtual environment with all dependencies (for development)

### Python Dependency Management

Uses `uv2nix` and `pyproject-nix` to convert `uv.lock` into Nix packages. Python dependencies are defined in `pyproject.toml`, which tracks InvenTree's requirements.

Several packages require build system overrides in `flake.nix` (django-allauth, django-mailbox, django-xforwardedfor-middleware, dj-rest-auth, odfpy, sgmllib3k, coreschema, invoke) to add setuptools/wheel. This is necessary because these packages use legacy setuptools but don't declare it in their build dependencies, which causes build failures in the stricter Nix environment.

The `weasyprint` package uses a special hack to use the pre-built nixpkgs version instead of building from source.

**Database drivers**: The module includes `psycopg2-binary` for PostgreSQL support. InvenTree's base requirements don't include database drivers (they're optional based on database choice), but the NixOS module adds them automatically to support common deployment scenarios.

### NixOS Module

The module (defined at bottom of `flake.nix`) provides `services.inventree` options:

- Creates systemd services: `inventree-server` (gunicorn) and `inventree-cluster` (background worker)
- Manages directories via systemd's RuntimeDirectory/StateDirectory/etc
- Handles config file generation, database migrations, static file deployment, and user provisioning in `ExecStartPre`
- Provides `inventree-invoke` command-line tool (wrapped with INVENTREE_CONFIG_FILE)

### Two-Service Architecture

InvenTree requires two services running simultaneously:
1. **inventree-server**: HTTP server (gunicorn/WSGI) for web UI and API
2. **inventree-cluster**: Background worker (Django Q2) for async tasks

Both services share the same configuration file and database.

```
┌──────────────────┐     ┌──────────────────┐
│ inventree-server │────▶│   PostgreSQL     │
│   (gunicorn)     │     │ or SQLite/MySQL  │
└──────────────────┘     └──────────────────┘
         │                        ▲
         │ Enqueues tasks         │
         ▼                        │
┌──────────────────┐              │
│inventree-cluster │──────────────┘
│   (Django Q2)    │    Processes tasks
└──────────────────┘
```

## Development Commands

### Building and Testing

```bash
# Build the source package (includes frontend and static files)
nix build .#src

# Build server wrapper
nix build .#server

# Build cluster worker wrapper
nix build .#cluster

# Format Nix files
nix fmt
```

### Development Shells

```bash
# Enter default development shell (includes all InvenTree tools)
nix develop

# Enter uv development shell (for updating dependencies)
nix develop .#uv
```

### Updating InvenTree Version

Follow the process in README.md:

1. Enter uv devshell: `nix develop .#uv`
2. Update InvenTree submodule to latest release
3. Update the version and srcs targets in `pkgs/src.nix` (use dummy hashes to ensure new downloads happen)
4. Run `./update-overrides.sh` to regenerate `pyproject.toml` and `uv.lock`
5. Run `nix build .#src` to get expected hashes and verify the build succeeds

The `update-overrides.sh` script performs the following:
- Cleans existing uv-generated files (`pyproject.toml`, `uv.lock`, etc.)
- Runs `uv init` to create a new project
- Runs `uv add -r InvenTree/src/backend/requirements.txt` to add InvenTree's dependencies
- Adds custom dependencies:
  - **invoke from git**: Uses the latest version from GitHub to fix [issue #1011](https://github.com/pyinvoke/invoke/issues/1011), waiting for version 2.3.0 release
  - **pip**: Required for InvenTree's plugin system support
- Adds setuptools workaround to `pyproject.toml` to prevent setuptools from creating unwanted modules

## Key Technical Details

### Path Structure in Packages

The source package has a nested structure: `${src}/src/src/backend/InvenTree` is the actual Django project root. This is because:
- The build process creates a `src/` directory
- InvenTree's source has backend code in `src/backend/`
- The Django project itself is in `InvenTree/`

### Configuration Management

Environment variables control InvenTree behavior:
- `INVENTREE_CONFIG_FILE`: Path to config.yaml
- `INVENTREE_SITE_URL`: Base URL for the server
- `INVENTREE_ALLOWED_HOSTS`: Comma-separated list of allowed hostnames
- `INVENTREE_SRC`: Path to source code (set by wrapper scripts)
- Various database and storage paths

The module generates `config.yaml` from NixOS options and copies it to the dataDir at service startup.

### User Management

The `refresh-users` package reads a JSON file (generated from NixOS config) and creates/updates users in the InvenTree database. Users are defined declaratively in `services.inventree.users` with password files.

### Static Files

Static files are pre-built during the `src` package build phase and then copied to `static_root` by the systemd service. The Django collectstatic process runs during build, not at runtime.

### Patches Applied

`patches/disable-fs-mutation-tasks.patch`: Disables invoke tasks that would attempt to modify the immutable Nix store. Specifically disables:
- `install`: Installing Python packages (dependencies are managed by Nix)
- `static`: Collecting static files (pre-built during package build)
- `frontend_compile`: Compiling frontend code (pre-built and fetched from releases)
- `frontend_download`: Downloading frontend assets (fetched during build)

These tasks are made no-ops because Nix builds are immutable - all dependencies and assets must be declared at build time, not modified at runtime. The patch ensures InvenTree's invoke commands won't fail when trying to write to read-only Nix store paths.

## Flake Inputs

- **nixpkgs**: NixOS package repository (follows unstable channel)
- **pyproject-nix**: Library for building Python projects with Nix
- **uv2nix**: Converts uv.lock files to Nix expressions
- **pyproject-build-systems**: Provides Python build system packages
- **flake-utils**: Utilities for multi-system flakes

## Testing the Module

To test the NixOS module in a VM or system:

```nix
{
  imports = [ nixos-inventree.nixosModules.default ];

  services.inventree = {
    enable = true;
    siteUrl = "http://localhost:8000";

    # Optional network configuration
    bindIp = "127.0.0.1";  # Default: 127.0.0.1 (localhost only)
    bindPort = 8000;       # Default: 8000

    # Optional: Allowed hosts for security (defaults to siteUrl if not set)
    allowedHosts = [ "localhost" "inventree.example.com" ];

    # Optional: Increase timeout for long migrations
    serverStartTimeout = "10min";  # Default: 10min

    config = {
      # Database, media paths, etc.
      # See InvenTree docs: https://docs.inventree.org/en/stable/start/config/
    };

    users.admin = {
      email = "admin@example.com";
      is_superuser = true;
      password_file = /path/to/password;  # Secure this file appropriately
    };
  };
}
```

### Important Module Options

- **`bindIp`**: IP address to bind the server to. Use `"127.0.0.1"` for localhost only (recommended for reverse proxy setups), or `"0.0.0.0"` to bind to all interfaces.
- **`bindPort`**: Port number for the server (default: 8000).
- **`siteUrl`**: The base URL users will use to access InvenTree. Critical for correct operation.
- **`allowedHosts`**: List of allowed hostnames. If empty, defaults to allow all (`["*"]`). The `siteUrl` hostname is automatically added to this list.
- **`serverStartTimeout`**: How long systemd waits for the server to start (default: 10min). Increase for large databases with lengthy migrations.
- **`dataDir`**: Base directory for all InvenTree data (default: `/var/lib/inventree`).
- **`config`**: Arbitrary configuration passed to InvenTree's `config.yaml`. See [InvenTree configuration docs](https://docs.inventree.org/en/stable/start/config/) for available options.

## Security Considerations

### Password Files
User passwords are stored in files referenced by `password_file` paths. Ensure these files:
- Are owned by root with restricted permissions (e.g., `chmod 600`)
- Are not world-readable or in publicly accessible locations
- Are backed up securely if needed
- Use strong passwords (consider using password managers to generate them)

### Network Security
- **Localhost binding**: By default, the server binds to `127.0.0.1` (localhost only). This is the recommended configuration when using a reverse proxy.
- **Public binding**: Only use `bindIp = "0.0.0.0"` if you understand the security implications. Always use HTTPS/TLS when exposing to the internet.
- **Allowed hosts**: Configure `allowedHosts` appropriately to prevent host header injection attacks. Never use `["*"]` in production unless necessary.

### Production Deployments
For production use, it's strongly recommended to:
1. **Use a reverse proxy** (nginx, Caddy, Traefik) in front of InvenTree for:
   - TLS/SSL termination
   - Rate limiting
   - Request filtering
   - Static file serving (optional performance optimization)

2. **Use a proper database**: Configure PostgreSQL or MySQL instead of the default SQLite for better performance and reliability.

3. **Configure backups**: Use InvenTree's backup features and the `backup_dir` configuration option.

4. **Keep updated**: Regularly update to the latest InvenTree version for security patches.

Example reverse proxy configuration (nginx):
```nginx
server {
    listen 443 ssl http2;
    server_name inventree.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Troubleshooting

### Migration Failures

If database migrations fail during service startup:
1. Check logs: `journalctl -u inventree-server -n 100`
2. Verify database connection settings in `services.inventree.config`
3. Ensure database service is running and accessible
4. Check that `serverStartTimeout` is sufficient (default: 10min)

### Build Errors

**Hash mismatches**: Use dummy hashes (all zeros) in `pkgs/src.nix` to force new downloads, then get correct hashes from build output.

**Python dependency issues**: Check that `pyproject.toml` and `uv.lock` are in sync. Re-run `./update-overrides.sh` if needed.

**Missing build dependencies**: Some packages may need additional overrides in `flake.nix` - check the error for missing setuptools/wheel.

### Service Issues

**Service won't start**:
- Check logs: `journalctl -u inventree-server -xe` and `journalctl -u inventree-cluster -xe`
- Verify static files exist: `ls /var/lib/inventree/static` (or your configured `static_root`)
- Check file permissions on dataDir, static_root, media_root

**Background tasks not running**:
- Ensure `inventree-cluster` service is active: `systemctl status inventree-cluster`
- Both services must share the same database and config file

### Accessing Logs

- Server logs: `journalctl -u inventree-server -f`
- Cluster worker logs: `journalctl -u inventree-cluster -f`
- Both services: `journalctl -u inventree-* -f`

## MCP NixOS Tools

When working with this repository, use the available MCP NixOS tools for package and option lookups:

- `mcp__nixos__nixos_search`: Search NixOS packages
- `mcp__nixos__nixos_info`: Get detailed package information
- `mcp__nixos__nixhub_package_versions`: Find specific package versions and commit hashes
- `mcp__nixos__home_manager_search`: Search Home Manager options
- `mcp__nixos__darwin_search`: Search nix-darwin options (macOS)

## Documentation Maintenance

### Keeping Documentation Current

When making architectural or configuration changes to this repository, please update this CLAUDE.md file accordingly:

- **Package changes**: Update the "Package Structure" section if adding, removing, or significantly modifying packages
- **NixOS module options**: Update the "Important Module Options" section when adding new configuration options
- **Dependency changes**: Update Python dependency information if the override list changes
- **Build process changes**: Update development commands and workflow descriptions
- **Version updates**: Update the "Last Updated" date in the Project Overview section

### Contributing to Documentation

If you notice missing, outdated, or incorrect information:
1. Make the corrections directly in CLAUDE.md
2. Submit a pull request with clear descriptions of what was updated and why
3. Tag PRs with documentation changes using the `docs:` commit prefix

### External Resources

For more information about InvenTree itself, consult the official documentation:

- **InvenTree Documentation**: https://docs.inventree.org/
- **Configuration Reference**: https://docs.inventree.org/en/stable/start/config/
- **API Documentation**: https://docs.inventree.org/en/stable/api/api/
- **Plugin Development**: https://docs.inventree.org/en/stable/extend/plugins/
- **InvenTree GitHub**: https://github.com/inventree/InvenTree

For Nix-specific documentation:
- **NixOS Manual**: https://nixos.org/manual/nixos/stable/
- **Nix Pills**: https://nixos.org/guides/nix-pills/
- **pyproject-nix**: https://pyproject-nix.github.io/pyproject.nix/
- **uv2nix**: https://github.com/pyproject-nix/uv2nix
