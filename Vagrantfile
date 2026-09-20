Vagrant.configure("2") do |config|
  config.vm.hostname = "omp"
  config.vm.box = "debian/trixie64"
  config.vm.box_version = "13.20260519.1"
  bun_version = "v1.4.2"
  omp_version = "v18.2.6"
  skills_version = "v1.2.3"

  mem = (ENV['VAGRANT_MEM'] || 8192).to_i
  cpus = (ENV['VAGRANT_CPUS'] || 4).to_i
  projects_dir = ENV['PROJECTS_DIR'] || nil
  # determine when to require the env var
  actionable_commands = %w[up reload resume]

  invoked = ARGV.find { |a| actionable_commands.include?(a) }
  if invoked
    raise "Environment variable PROJECTS_DIR must be set (e.g. export PROJECTS_DIR=/path)" unless projects_dir
    raise "PROJECTS_DIR points to missing directory: #{projects_dir}" unless File.directory?(projects_dir)
  end

  config.vm.provider :libvirt do |lv|
    lv.memory = mem
    lv.cpus   = cpus
    lv.cpu_mode = "host-passthrough"
  end

  if invoked
    config.vm.synced_folder projects_dir,
      "/home/vagrant/" + File.basename(projects_dir),
      type: "nfs",
      nfs_version: 4
  end

  config.vm.synced_folder "~/.omp",
    "/home/vagrant/.omp",
    type: "nfs",
    nfs_version: 4

  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provision "shell", inline: <<-SHELL
    set -ex

    apt-get update -y
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
      git \
      curl \
      direnv \
      xauth \
      x11-apps \
      dbus-user-session \
      fonts-dejavu-core \
      bash-completion \
      chromium \
      podman \
      podman-compose \
      unzip \
      tmux \
      vim

    loginctl enable-linger vagrant
    # dbus-user-session's sockets.target.wants symlink only applies at user manager
    # startup; the package is installed after user@1000 is already running, so start
    # the session bus explicitly or podman falls back to cgroupfs
    su - vagrant -c 'XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user daemon-reload && XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user start dbus.socket'
    su - vagrant -c 'XDG_RUNTIME_DIR=/run/user/$(id -u) systemctl --user enable --now podman.socket'
    DIRENV_SHELLHOOK='eval "$(direnv hook bash)"'
    PODMAN_SOCKET_LOCATION="unix://$(sudo -u vagrant -- podman info --format '{{.Host.RemoteSocket.Path}}')"
    BASHRC_PATH="/home/vagrant/.bashrc"
    grep -qxF "$DIRENV_SHELLHOOK" "$BASHRC_PATH" || echo "$DIRENV_SHELLHOOK" >> "$BASHRC_PATH"
    grep -qxF "export DOCKER_HOST=\"$PODMAN_SOCKET_LOCATION\"" "$BASHRC_PATH" || echo "export DOCKER_HOST=\"$PODMAN_SOCKET_LOCATION\"" >> "$BASHRC_PATH"
    TMUX_SHELLHOOK='# Auto-attach tmux on SSH login; exiting tmux ends the SSH session
if [[ $- == *i* ]] && [[ -z "$TMUX" ]] && [[ -n "$SSH_TTY" ]] && command -v tmux >/dev/null 2>&1; then
  exec tmux new-session -A -s main
fi'
    grep -qF 'exec tmux new-session' "$BASHRC_PATH" || printf '%s\n' "$TMUX_SHELLHOOK" >> "$BASHRC_PATH"
    # tmux panes run login shells: /etc/profile resets PATH, then nix-daemon.sh's
    # per-shell guard (inherited from the server's environment) stops it re-adding
    # the nix dirs. Unset the guard so every pane's login shell gets nix on PATH.
    TMUX_NIX_GUARD='set-environment -g -u __ETC_PROFILE_NIX_SOURCED'
    TMUX_CONF_PATH="/home/vagrant/.tmux.conf"
    touch "$TMUX_CONF_PATH"
    grep -qxF "$TMUX_NIX_GUARD" "$TMUX_CONF_PATH" || echo "$TMUX_NIX_GUARD" >> "$TMUX_CONF_PATH"
    chown vagrant:vagrant "$TMUX_CONF_PATH"

    su - vagrant -c '
      curl -fsSL https://bun.com/install | bash -s "bun-#{bun_version}"
      $HOME/.bun/bin/bun install -g @oh-my-pi/pi-coding-agent@#{omp_version}
      $HOME/.bun/bin/bun install -g github:mattpocock/skills##{skills_version}
      mkdir -p ~/.omp/agent/skills
      find $HOME/.bun/install/global/node_modules/mattpocock-skills/skills -name SKILL.md -type f |
      while IFS= read -r skill; do
          dir="$(dirname "$skill")"
          name="$(basename "$dir")"
          ln -sfn "$dir" "$HOME/.omp/agent/skills/$name"
      done
    '

    # ensure SSH allows X11Forwarding (usually default)
    sed -i 's/^#X11Forwarding yes/X11Forwarding yes/' /etc/ssh/sshd_config || true
    sed -i 's/^#X11UseLocalhost yes/X11UseLocalhost yes/' /etc/ssh/sshd_config || true
    systemctl restart ssh || service ssh restart || true

    test -e "/nix" || \
      curl --proto '=https' --tlsv1.2 -sSf -L https://install.lix.systems/lix | \
      sh -s -- install linux --no-confirm

    sudo -u vagrant git config --global user.name "omp"
    sudo -u vagrant git config --global user.email "omp@local"
  SHELL
  config.vm.provision "file",
    source: "~/.ssh/known_hosts",
    destination: "/home/vagrant/.ssh/known_hosts",
    run: "always"
end
