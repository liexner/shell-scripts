#!/bin/bash
set -euo pipefail

# ─────────────────────────────────────────────
#  Ubuntu VM Setup Script
#  - System update & upgrade
#  - Docker (official install)
#  - Unattended upgrades (security + auto-reboot at 03:00)
#  - Post-setup: docker group, service restarts, reboot if needed
# ─────────────────────────────────────────────

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

info()  { echo -e "${GREEN}[INFO]${NC}  $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC}  $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*"; exit 1; }

# Must be run as root
[[ $EUID -ne 0 ]] && error "Run as root: sudo bash setup-ubuntu.sh"

# ─────────────────────────────────────────────
#  CONFIGURATION — edit before running
# ─────────────────────────────────────────────

# Extra users to add to the docker group (space-separated), e.g. "alice bob"
# The current sudo user is always added automatically
DOCKER_USERS=""


# ─── 1. UPDATE & UPGRADE ──────────────────────
info "Updating package lists..."
apt-get update -y

info "Upgrading installed packages..."
DEBIAN_FRONTEND=noninteractive apt-get upgrade -y \
  -o Dpkg::Options::="--force-confdef" \
  -o Dpkg::Options::="--force-confold"

info "Removing unused packages..."
apt-get autoremove -y
apt-get autoclean -y

# ─── 2. INSTALL DOCKER ────────────────────────
info "Installing Docker (official method)..."

# Remove old/conflicting packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  apt-get remove -y "$pkg" 2>/dev/null || true
done

# Dependencies
apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl enable --now docker

info "Docker installed: $(docker --version)"

# ─── 3. UNATTENDED UPGRADES ───────────────────
info "Configuring unattended-upgrades..."

apt-get install -y unattended-upgrades apt-listchanges

cat > /etc/apt/apt.conf.d/20auto-upgrades <<'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Download-Upgradeable-Packages "1";
EOF

cat > /etc/apt/apt.conf.d/50unattended-upgrades <<'EOF'
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

Unattended-Upgrade::Package-Blacklist {
    "docker-ce";
    "docker-ce-cli";
    "containerd.io";
    "docker-buildx-plugin";
    "docker-compose-plugin";
};

Unattended-Upgrade::DevRelease "false";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";

Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

// Unattended-Upgrade::Mail "you@example.com";
// Unattended-Upgrade::MailReport "on-change";

Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::Verbose "false";
EOF

systemctl enable unattended-upgrades
systemctl restart unattended-upgrades

info "Verifying unattended-upgrades config (dry-run)..."
unattended-upgrades --dry-run --debug 2>&1 | tail -5

# ─── 4. POST-SETUP ────────────────────────────
info "Running post-setup tasks..."

# -- Docker group --
# Add the invoking sudo user automatically
SUDO_USER_NAME="${SUDO_USER:-}"
if [[ -n "$SUDO_USER_NAME" && "$SUDO_USER_NAME" != "root" ]]; then
  usermod -aG docker "$SUDO_USER_NAME"
  info "Added '$SUDO_USER_NAME' to docker group."
fi

# Add any extra users defined in DOCKER_USERS
for u in $DOCKER_USERS; do
  if id "$u" &>/dev/null; then
    usermod -aG docker "$u"
    info "Added '$u' to docker group."
  else
    warn "User '$u' does not exist — skipping."
  fi
done

# -- Service restarts --
info "Restarting services..."
systemctl daemon-reload
systemctl restart docker
systemctl restart unattended-upgrades
info "Services restarted."

# ─── DONE ─────────────────────────────────────
echo ""
echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "${GREEN}  Setup complete!${NC}"
echo -e "${GREEN}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo ""
echo "  Docker:              $(docker --version)"
echo "  Unattended upgrades: active (security only, reboot at 03:00)"
echo "  Docker group users:  ${SUDO_USER_NAME:-root} ${DOCKER_USERS}"
echo ""

warn "Rebooting in 1 minute. Cancel with: sudo shutdown -c"
shutdown -r +1 "Setup complete — rebooting"
