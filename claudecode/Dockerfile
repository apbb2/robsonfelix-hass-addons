ARG BUILD_FROM
FROM ${BUILD_FROM}

# Labels
LABEL maintainer="Robson Felix"
LABEL org.opencontainers.image.title="Claude Code for Home Assistant"
LABEL org.opencontainers.image.description="Claude Code CLI with web terminal and HA integration"
LABEL org.opencontainers.image.source="https://github.com/robsonfelix/robsonfelix-hass-addons"
LABEL org.opencontainers.image.licenses="MIT"

# Environment setup
ENV \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TERM=xterm-256color \
    DISABLE_AUTOUPDATER=1

# Install system dependencies (ttyd installed separately as static binary)
RUN apk add --no-cache \
    bash \
    curl \
    git \
    github-cli \
    jq \
    nodejs \
    npm \
    openssh-client \
    libstdc++ \
    ncurses \
    vim \
    nano \
    tmux \
    coreutils \
    findutils \
    grep \
    sed \
    gawk \
    ca-certificates \
    openssl \
    ripgrep \
    p7zip \
    socat \
    docker-cli \
    && update-ca-certificates

# Install ttyd static binary (avoids libwebsockets issues)
ARG BUILD_ARCH
RUN case "${BUILD_ARCH}" in \
        amd64) TTYD_ARCH="x86_64" ;; \
        aarch64) TTYD_ARCH="aarch64" ;; \
        armv7|armhf) TTYD_ARCH="armhf" ;; \
        i386) TTYD_ARCH="i686" ;; \
        *) TTYD_ARCH="x86_64" ;; \
    esac && \
    curl -fsSL "https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.${TTYD_ARCH}" -o /usr/bin/ttyd && \
    chmod +x /usr/bin/ttyd

# Provide a passthrough sudo so Claude Code's postinstall script doesn't fail
# (container runs as root so sudo is unnecessary, but HA's security profile blocks the real binary)
RUN printf '#!/bin/sh\nexec "$@"\n' > /usr/local/bin/sudo && chmod +x /usr/local/bin/sudo

# Install Claude Code via npm
# BUILD_VERSION is passed automatically by HA and busts the Docker cache on every version bump
ARG BUILD_VERSION=dev
RUN echo "Building add-on ${BUILD_VERSION} - installing latest Claude Code" && \
    npm install -g @anthropic-ai/claude-code@latest --no-fund --no-audit

# Install hass-mcp for Home Assistant integration and pymodbus for Modbus operations
RUN pip3 install --no-cache-dir --break-system-packages hass-mcp pymodbus pyserial

# Install mbpoll (Modbus command line tool) from source
RUN apk add --no-cache --virtual .build-deps \
    build-base \
    cmake \
    libmodbus-dev \
    && git clone https://github.com/epsilonrt/mbpoll.git /tmp/mbpoll \
    && cd /tmp/mbpoll \
    && mkdir build && cd build \
    && cmake .. \
    && make \
    && make install \
    && cd / \
    && rm -rf /tmp/mbpoll \
    && apk del .build-deps \
    && apk add --no-cache libmodbus

# Pre-cache Playwright MCP server (npx cache avoids /usr/local/bin/mcp conflict with hass-mcp)
ENV NPM_CONFIG_CACHE=/root/.npm
RUN npx @playwright/mcp --version || true

# Install Home Assistant CLI (ha command)
RUN case "${BUILD_ARCH}" in \
        amd64) HA_ARCH="amd64" ;; \
        aarch64) HA_ARCH="aarch64" ;; \
        armv7|armhf) HA_ARCH="armhf" ;; \
        i386) HA_ARCH="i386" ;; \
        *) HA_ARCH="amd64" ;; \
    esac && \
    curl -fsSL "https://github.com/home-assistant/cli/releases/latest/download/ha_${HA_ARCH}" -o /usr/local/bin/ha && \
    chmod +x /usr/local/bin/ha

# Create necessary directories (NOT /root/.claude - will be symlinked at runtime)
RUN mkdir -p \
    /homeassistant \
    /root/.config \
    /root/.local/bin

# Configure tmux with large scrollback buffer and scroll support
RUN cat > /root/.tmux.conf << 'EOF'
set -g history-limit 20000
# Disable alternate screen buffer to allow terminal scrollback
set -ga terminal-overrides ',xterm*:smcup@:rmcup@'
# Enable mouse support for scrolling (but may affect copy/paste)
set -g mouse on
# Bind mouse wheel to scroll through history in copy mode
bind -n WheelUpPane if-shell -F -t = "#{mouse_any_flag}" "send-keys -M" "if -Ft= '#{pane_in_mode}' 'send-keys -M' 'select-pane -t=; copy-mode -e; send-keys -M'"
bind -n WheelDownPane select-pane -t= \; send-keys -M
EOF

# Configure bash with aliases and prompt
RUN cat > /root/.bashrc << 'EOF'
export TERM=xterm-256color
export LANG=C.UTF-8
export COLUMNS=220
PS1='\[\033[1;36m\]claude-code\[\033[0m\]:\[\033[1;34m\]\w\[\033[0m\]\$ '

# Function to update MCP token before starting Claude
update_mcp_token() {
  local SETTINGS_FILE=/root/.claude/settings.json
  if [ -f "$SETTINGS_FILE" ] && [ -n "$SUPERVISOR_TOKEN" ]; then
    jq ".mcpServers.homeassistant.env.HASS_TOKEN = \"$SUPERVISOR_TOKEN\"" "$SETTINGS_FILE" > /tmp/settings.tmp 2>/dev/null && mv /tmp/settings.tmp "$SETTINGS_FILE"
  fi
}

# Aliases
alias ll='ls -la'
alias c='update_mcp_token && stty cols 220 2>/dev/null; claude'
alias cc='update_mcp_token && stty cols 220 2>/dev/null; claude --continue'
alias ha-config='cd /homeassistant'
alias ha-logs='cat /homeassistant/home-assistant.log 2>/dev/null || echo "Log not found"'
alias claude-update='echo "Updating Claude Code..." && claude update 2>&1; echo "Done: $(claude --version 2>/dev/null)"'
EOF

# Create update notification script and wire into .bashrc
RUN echo '[ -f /homeassistant/.claudecode/.update_notice ] && echo -e "\033[1;33m[UPDATE AVAILABLE]\033[0m Claude Code $(cat /homeassistant/.claudecode/.update_notice) available - restart the add-on or run: claude-update"' > /root/.claude-notify.sh
RUN echo 'source /root/.claude-notify.sh' >> /root/.bashrc
# Set PTY width so Node.js apps (like Claude Code) read 220 cols via ioctl, not just the COLUMNS shell var
RUN echo 'stty cols 220 rows 50 2>/dev/null || true' >> /root/.bashrc

# Ensure .bashrc is sourced for login shells too
RUN echo 'source ~/.bashrc' > /root/.profile

# Set up PATH
ENV PATH="/root/.local/bin:${PATH}"

# Set working directory
WORKDIR /homeassistant

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -sf http://localhost:7681/ || exit 1

# Startup script that reads config and persists Claude auth
# Use /homeassistant/.claudecode which is always persistent
ENTRYPOINT []
CMD ["/bin/bash", "-c", "\
  export HA_TOKEN=\"$SUPERVISOR_TOKEN\" && \
  export HA_URL=\"http://supervisor/core\" && \
  PERSIST_DIR=/homeassistant/.claudecode && \
  mkdir -p $PERSIST_DIR/config /root/.config && \
  printf '%s\n' \
    '# Claude Code - Home Assistant Add-on' \
    '' \
    '## Path Mapping' \
    '' \
    'In this add-on container, paths are mapped differently than HA Core:' \
    '- \\`/homeassistant\\` = HA config directory (equivalent to \\`/config\\` in HA Core)' \
    '- \\`/config\\` does NOT exist - always use \\`/homeassistant\\`' \
    '' \
    'When users mention \\`/config/...\\`, translate to \\`/homeassistant/...\\`' \
    '' \
    '## Available Paths' \
    '' \
    '| Path | Description | Access |' \
    '|------|-------------|--------|' \
    '| \\`/homeassistant\\` | HA configuration | read-write |' \
    '| \\`/share\\` | Shared folder | read-write |' \
    '| \\`/media\\` | Media files | read-write |' \
    '| \\`/ssl\\` | SSL certificates | read-only |' \
    '| \\`/backup\\` | Backups | read-only |' \
    '' \
    '## Home Assistant Integration' \
    '' \
    'Use the \\`homeassistant\\` MCP server to query entities and call services.' \
    '' \
    '## Reading Home Assistant Logs' \
    '' \
    '**Log levels (from most to least verbose):**' \
    '- \\`debug\\` - Only shown if explicitly enabled in configuration.yaml' \
    '- \\`info\\` - General information, shown by default' \
    '- \\`warning\\` - Warnings, always shown' \
    '- \\`error\\` - Errors, always shown' \
    '' \
    '**Commands to read logs:**' \
    '\\`\\`\\`bash' \
    '# View recent logs (ha CLI)' \
    'ha core logs 2>&1 | tail -100' \
    '' \
    '# Filter by keyword' \
    'ha core logs 2>&1 | grep -i keyword' \
    '' \
    '# Filter errors only' \
    'ha core logs 2>&1 | grep -iE \"(error|exception)\"' \
    '' \
    '# Alternative: read log file directly' \
    'tail -100 /homeassistant/home-assistant.log' \
    '\\`\\`\\`' \
    '' \
    '**To enable debug logging for an integration**, add to \\`configuration.yaml\\`:' \
    '\\`\\`\\`yaml' \
    'logger:' \
    '  default: info' \
    '  logs:' \
    '    custom_components.YOUR_INTEGRATION: debug' \
    '\\`\\`\\`' \
    '' \
    '**Key insight:** \\`_LOGGER.debug()\\` calls are invisible unless the logger level is set to debug. Use \\`_LOGGER.info()\\` or \\`_LOGGER.warning()\\` for logs that should always appear.' \
    > $PERSIST_DIR/CLAUDE.md && \
  if [ ! -L /root/.claude ]; then rm -rf /root/.claude; ln -s $PERSIST_DIR /root/.claude; fi && \
  if [ ! -L /root/.config/claude-code ]; then rm -rf /root/.config/claude-code; ln -s $PERSIST_DIR/config /root/.config/claude-code; fi && \
  if [ ! -L /root/.claude.json ]; then touch $PERSIST_DIR/.claude.json; rm -f /root/.claude.json; ln -s $PERSIST_DIR/.claude.json /root/.claude.json; fi && \
  mkdir -p $PERSIST_DIR/local-bin && \
  if [ ! -L /root/.local/bin ]; then rm -rf /root/.local/bin; ln -s $PERSIST_DIR/local-bin /root/.local/bin; fi && \
  FONT_SIZE=$(jq -r '.terminal_font_size // 14' /data/options.json) && \
  THEME=$(jq -r '.terminal_theme // \"dark\"' /data/options.json) && \
  SESSION_PERSIST=$(jq -r '.session_persistence // true' /data/options.json) && \
  ENABLE_MCP=$(jq -r '.enable_mcp // true' /data/options.json) && \
  ENABLE_PLAYWRIGHT=$(jq -r '.enable_playwright_mcp // false' /data/options.json) && \
  PLAYWRIGHT_HOST=$(jq -r '.playwright_cdp_host // \"\"' /data/options.json) && \
  if [ -z \"$PLAYWRIGHT_HOST\" ] && [ \"$ENABLE_PLAYWRIGHT\" = \"true\" ]; then \
    echo '[INFO] Auto-detecting Playwright Browser hostname...' && \
    PLAYWRIGHT_HOST=$(curl -s -H \"Authorization: Bearer $SUPERVISOR_TOKEN\" http://supervisor/addons | jq -r '.data.addons[] | select(.slug | endswith(\"playwright-browser\") or endswith(\"_playwright-browser\")) | .hostname' | head -1) && \
    if [ -n \"$PLAYWRIGHT_HOST\" ] && [ \"$PLAYWRIGHT_HOST\" != \"null\" ]; then \
      echo \"[INFO] Found Playwright Browser: $PLAYWRIGHT_HOST\"; \
    else \
      echo '[WARN] Playwright Browser add-on not found, using default hostname'; \
      PLAYWRIGHT_HOST=\"playwright-browser\"; \
    fi; \
  fi && \
  AUTO_UPDATE=$(jq -r '.auto_update_claude // true' /data/options.json) && \
  if [ \"$AUTO_UPDATE\" = \"true\" ]; then \
    CURRENT_VER=$(claude --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1) && \
    LATEST_VER=$(npm show @anthropic-ai/claude-code version 2>/dev/null) && \
    if [ -n \"$LATEST_VER\" ] && [ -n \"$CURRENT_VER\" ] && [ \"$CURRENT_VER\" != \"$LATEST_VER\" ]; then \
      echo \"[INFO] Updating Claude Code from $CURRENT_VER to $LATEST_VER...\"; \
      yes 2>/dev/null | timeout 120 claude update 2>&1 || true; \
      echo \"[INFO] Claude Code update complete: $(claude --version 2>/dev/null)\"; \
    else \
      echo \"[INFO] Claude Code $CURRENT_VER is up to date\"; \
    fi; \
  fi && \
  MODEL=$(jq -r '.model // \"claude-sonnet-4-6\"' /data/options.json) && \
  export ANTHROPIC_MODEL=\"$MODEL\" && \
  echo \"[INFO] Using Claude model: $MODEL\" && \
  claude mcp remove homeassistant -s user 2>/dev/null || true && \
  claude mcp remove playwright -s user 2>/dev/null || true && \
  if [ \"$ENABLE_MCP\" = \"true\" ]; then \
    claude mcp add-json homeassistant '{\"command\":\"hass-mcp\"}' -s user && \
    SETTINGS_FILE=/root/.claude/settings.json && \
    ALLOWED_TOOLS='[\"mcp__homeassistant__get_version\",\"mcp__homeassistant__get_entity\",\"mcp__homeassistant__list_entities\",\"mcp__homeassistant__search_entities_tool\",\"mcp__homeassistant__domain_summary_tool\",\"mcp__homeassistant__list_automations\",\"mcp__homeassistant__get_history\",\"mcp__homeassistant__get_error_log\",\"Read(/homeassistant/**)\",\"Read(/config/**)\",\"Read(/share/**)\",\"Read(/media/**)\",\"Glob(/homeassistant/**)\",\"Glob(/config/**)\",\"Grep(/homeassistant/**)\",\"Grep(/config/**)\"]' && \
    jq --argjson tools \"$ALLOWED_TOOLS\" '.permissions.allow = ($tools + (.permissions.allow // []) | unique)' $SETTINGS_FILE > /tmp/settings.tmp && mv /tmp/settings.tmp $SETTINGS_FILE && \
    jq \".mcpServers.homeassistant.env.HASS_TOKEN = \\\"$SUPERVISOR_TOKEN\\\"\" $SETTINGS_FILE > /tmp/settings.tmp && mv /tmp/settings.tmp $SETTINGS_FILE && \
    echo '[INFO] MCP configured with Home Assistant integration'; \
    echo '[INFO] Pre-authorized read-only MCP tools'; \
  else \
    echo '[INFO] MCP disabled'; \
  fi && \
  if [ \"$ENABLE_PLAYWRIGHT\" = \"true\" ]; then \
    claude mcp add-json playwright \"{\\\"command\\\":\\\"npx\\\",\\\"args\\\":[\\\"--no-install\\\",\\\"@playwright/mcp\\\",\\\"--cdp-endpoint\\\",\\\"http://${PLAYWRIGHT_HOST}:9222\\\"]}\" -s user && \
    echo \"[INFO] Playwright MCP enabled (CDP: http://${PLAYWRIGHT_HOST}:9222)\"; \
    echo '[INFO] Make sure the Playwright Browser add-on is installed and running'; \
  else \
    echo '[INFO] Playwright MCP disabled'; \
  fi && \
  if [ \"$THEME\" = \"dark\" ]; then \
    COLORS='background=#1e1e2e,foreground=#cdd6f4,cursor=#f5e0dc'; \
  else \
    COLORS='background=#eff1f5,foreground=#4c4f69,cursor=#dc8a78'; \
  fi && \
  if [ \"$SESSION_PERSIST\" = \"true\" ]; then \
    SHELL_CMD='tmux new-session -A -s claude'; \
  else \
    SHELL_CMD='bash --login'; \
  fi; \
  (while true; do \
    IV=$(claude --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1); \
    LV=$(npm show @anthropic-ai/claude-code version 2>/dev/null); \
    if [ -n \"$LV\" ] && [ -n \"$IV\" ] && [ \"$IV\" != \"$LV\" ]; then \
      echo \"$LV\" > $PERSIST_DIR/.update_notice; \
      echo \"[INFO] Claude Code update available: $LV (installed: $IV)\"; \
      printf '{\"title\":\"Claude Code Update Available\",\"message\":\"Version %s is available (installed: %s). Restart the add-on to update.\",\"notification_id\":\"claude_code_update\"}' \"$LV\" \"$IV\" \
        | curl -sf -X POST -H \"Authorization: Bearer $SUPERVISOR_TOKEN\" -H \"Content-Type: application/json\" -d @- http://supervisor/core/api/services/persistent_notification/create 2>/dev/null || true; \
    else \
      rm -f $PERSIST_DIR/.update_notice 2>/dev/null; \
      printf '{\"notification_id\":\"claude_code_update\"}' \
        | curl -sf -X POST -H \"Authorization: Bearer $SUPERVISOR_TOKEN\" -H \"Content-Type: application/json\" -d @- http://supervisor/core/api/services/persistent_notification/dismiss 2>/dev/null || true; \
    fi; \
    sleep 3600; \
  done) & \
  cd /homeassistant && \
  exec ttyd --port 7681 --writable --ping-interval 30 --max-clients 5 \
    -t fontSize=$FONT_SIZE \
    -t fontFamily=Monaco,Consolas,monospace \
    -t scrollback=20000 \
    -t \"theme=$COLORS\" \
    $SHELL_CMD \
"]
