# syntax=docker/dockerfile:1.7-labs
# use buildx syntax definition to use secrets

# ============================================================================
# base
# ============================================================================
FROM ubuntu:24.04 AS base

ARG \
  CONTAINER_VERSION="unknown"
ENV \
  LANG="C.UTF-8"

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# hadolint ignore=DL3008
RUN <<EOT
  export DEBIAN_FRONTEND=noninteractive
# install minimal base packages
  apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl wget \
    dialog gnupg sq \
    tzdata locales-all \
    procps apt-utils net-tools iputils-ping lsb-release
# switch to https repos
  sed -i '/^[[:space:]]*#/!s/http/https/g' /etc/apt/sources.list.d/ubuntu.sources
# add additional repos
  install -dm 755 /etc/apt/keyrings
#  mise (https://mise.jdx.dev)
  # wget -qO - https://mise.jdx.dev/gpg-key.pub | gpg --dearmor | tee /etc/apt/keyrings/mise-archive-keyring.gpg 1> /dev/null
  curl -fsSL https://mise.jdx.dev/gpg-key.pub | gpg --dearmor -o /etc/apt/keyrings/mise-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/mise-archive-keyring.gpg] https://mise.jdx.dev/deb stable main" | tee /etc/apt/sources.list.d/mise.list > /dev/null
# install base tooling
  apt-get update && apt-get install -y --no-install-recommends \
    mise \
    build-essential \
    sudo htop less bash-completion \
    bash zsh fish \
    zip mc ncdu tmux \
    git rsync openssh-client \
    ripgrep fzf fd-find jq eza
# clean up after package installation
  apt-get autoremove -y
  apt-get clean -y
  rm -rf /var/lib/apt/lists/*
EOT

# Set up system ==============================================================
# hadolint ignore=SC2016
RUN <<EOT
# try to remove ubuntu user (if present)
  userdel --force --remove ubuntu || true
# create user and group
  groupadd devs --gid 2000
  useradd --create-home --groups devs,sudo --shell /bin/bash user --uid 1000;
# sudo without password
  echo '%sudo ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/sudo_group
# keep some env vars after sudo
  echo 'Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy "' >> /etc/sudoers.d/sudo_group
  chmod 440 /etc/sudoers.d/sudo_group
# activate mise also for root
  echo 'eval "$(mise activate bash)"' >> /root/.bashrc
# git system-wide configuration
  git config --system advice.detachedHead false
  git config --system init.defaultBranch main
  git config --system core.autocrlf input
  git config --system core.safecrlf warn
# use mise shims for all non-interactive shells
  echo 'eval "$(mise activate bash --shims)"' > /etc/profile.d/mise.sh
  chmod 644 /etc/profile.d/mise.sh
EOT

# ============================================================================
# devcontainer
# ============================================================================
FROM base AS devcontainer

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

RUN <<EOT
# restore manpages
yes | unminimize || true
# append version and build date to /etc/motd
 cat > /etc/motd <<EOF
 devcontainer-ubuntu [${CONTAINER_VERSION}], build $(LANG=C.UTF-8 date +%Y-%m-%dT%H:%M:%S%z)
EOF
EOT

# user context ===============================================================
USER user
WORKDIR /home/user
COPY --chown=user:user config/ /home/user/.config/

# user: install mise base tooling
RUN --mount=type=secret,id=mise_github_token,uid=1000 <<EOT
  MISE_GITHUB_TOKEN=$(cat /run/secrets/mise_github_token)
  export MISE_GITHUB_TOKEN
  eval "$(mise activate bash)"
  export MISE_VERBOSE=true
# install (and use) user-global packages
  mise --version
  mise use --global --yes \
    node@lts \
    uv@latest \
    neovim@latest \
    starship@latest \
    lazygit@latest \
    github-cli@latest \
    delta@latest \
    task@latest
  mise list
  mise cache clear --yes
EOT

# user: set up shell env
# hadolint ignore=SC2016
RUN <<EOT
  eval "$(mise activate bash)"
# user-global git configuration
  git config --global core.pager delta
  git config --global interactive.diffFilter 'delta --color-only'
  git config --global delta.navigate true
  git config --global merge.conflictStyle zdiff3
# user-global git aliases
  git config --global alias.co "checkout"
  git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative"
  git config --global alias.pcb "push -u origin HEAD"
  git config --global alias.undo-last-commit "reset --soft HEAD^"
  git config --global alias.up '!git remote update -p; git merge --ff-only @{u}'
  git config --global alias.uprebase "rebase --rebase-merges @{u}"
# starship prompt (https://starship.rs)
  starship config add_newline true
  starship config command_timeout 2000
  starship config hostname.ssh_only false
  starship config username.show_always true
  starship config username.format '[$user]($style)@'
  starship config container.disabled true
# bash shell
  echo 'eval "$(mise activate bash)"' >> /home/user/.bashrc
  echo 'eval "$(starship init bash)"' >> /home/user/.bashrc
# zsh shell
  echo 'eval "$(mise activate zsh)"' >> /home/user/.zshrc
  echo 'eval "$(starship init zsh)"' >> /home/user/.zshrc
# fish shell
  mkdir -p /home/user/.config/fish
  echo 'mise activate fish | source' >> /home/user/.config/fish/config.fish
  echo 'starship init fish | source' >> /home/user/.config/fish/config.fish
EOT

ENV \
  LANG=de_DE.UTF-8 \
  LANGUAGE=de_DE.UTF-8 \
  LC_ALL=de_DE.UTF-8 \
  TZ=Europe/Berlin \
  EDITOR=nvim \
  DOCKER_BUILDKIT=1

# use mise shims in path, interactive shells with activated mise will override this
ENV PATH=/home/user/.local/share/mise/shims:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Mount for docker-in-docker
VOLUME [ "/var/lib/docker" ]

CMD [ "sleep", "infinity" ]

# ============================================================================
# ci
# ============================================================================
FROM base AS ci

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# hadolint ignore=DL3008
RUN <<EOT
# append version and build date to /etc/motd
  cat > /etc/motd <<EOF
  ci-ubuntu [${CONTAINER_VERSION}], build $(LANG=C.UTF-8 date +%Y-%m-%dT%H:%M:%S%z)
EOF
  export DEBIAN_FRONTEND=noninteractive
# install docker-ce-cli (needed to build images in ci pipelines)
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  apt-get update && apt-get install -y --no-install-recommends \
    docker-ce-cli
# add specific ghrunner user for github-actions (must have UID 1001!)
# we do not need to do any special customizations for this user,
# because the home directory will get overridden by the github runner anyways
  useradd --create-home --home /github/home --groups devs,sudo --shell /bin/bash ghrunner --uid 1001;
# clean up after package installations
  apt-get autoremove -y
  apt-get clean -y
  rm -rf /var/lib/apt/lists/*
EOT

USER ghrunner

CMD [ "sleep", "infinity" ]

