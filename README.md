# dsb-ccb-solutions/container-images

This repository contains custom Docker container images for development environments, with a focus on devcontainers and CI environments. The images are built using multi-stage Dockerfiles and include modern development tooling managed via `mise` (formerly `rtx`).

## Structure

- **`src/ubuntu/`** - Ubuntu-based container definitions
  - **`Dockerfile`** - Multi-stage build with `base`, `devcontainer`, and `ci` targets
  - **`config/`** - User configuration files (e.g., tmux settings)

## Image Targets

### Base (`base`)
- Ubuntu 24.04 foundation
- Essential development tools (git, ripgrep, fzf, fd-find, jq, eza, tmux)
- Multiple shells (bash, zsh, fish)
- System-wide configuration and user setup
- Security: HTTPS-only APT repos, non-root user with sudo

### Devcontainer (`devcontainer`)
- Extends base with full development environment
- Tools installed via `mise`: Node.js (LTS), uv, neovim, starship, lazygit, github-cli, delta, task
- Pre-configured shells with starship prompt
- Git configuration with delta diff tool and useful aliases
- German locale (de_DE.UTF-8) and Europe/Berlin timezone
- Docker-in-docker volume mount

### CI (`ci`)
- Lightweight CI runner image
- Extends base with Docker CLI
- Includes `ghrunner` user (UID 1001) for GitHub Actions

## Building Images

### Prerequisites
- Docker with Buildx enabled
- `MISE_GITHUB_TOKEN` environment variable (required for tool installation)

### Using Taskfile for local builds

```bash
# Build all targets (default)
task

# Build specific target
task build:devcontainer:ubuntu
task build:ci:ubuntu

# Build target for single architecture
task build:all ARCHS=linux/amd64
task build:all ARCHS=linux/arm64
```
## CI/CD

Images are automatically built and pushed to GitHub Container Registry (`ghcr.io`) via GitHub Actions:

- **Triggers**: Pull requests, version tags (`v*.*.*`), manual dispatch
- **Platforms**: Multi-arch (amd64, arm64)
- **Tags**: Version tags, `latest`, `edge`, commit SHA
- **Cleanup**: Automated weekly cleanup of old SHA-tagged images

## Notable Features

- ✅ Multi-arch support (amd64, arm64)
- ✅ Modern tooling stack (mise, starship, delta, neovim)
- ✅ Security best practices (HTTPS repos, non-root users)
- ✅ Docker-in-docker support
- ✅ Automated CI/CD with caching
- ✅ Comprehensive tooling pre-installed

## License

Apache License 2.0
