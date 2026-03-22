# LocalWiki: Self-Hosted Wikipedia Mirror — Implementation Plan

## Goal

Run a local, LAN-accessible copy of Wikipedia in Docker, with automated updates.

---

## Method Comparison

| Method | Docker Support | Storage (EN) | Setup Time | Update Effort | Images | Status |
|--------|---------------|-------------|------------|---------------|--------|--------|
| **Kiwix** | Official (`ghcr.io/kiwix/kiwix-serve`) | 115 GB (full) / 48 GB (text-only) | ~5 min | Re-download ZIM (~6 months) | Yes (maxi variant) | Active |
| **MediaWiki + Dumps** | Official (`mediawiki` on Docker Hub) | 250–600 GB text + images separate | Days–weeks | Re-import dumps monthly | Separate download | Active |
| **XOWA** | Community only | Hundreds of GB | Hours | Manual re-import | Yes | Dormant (2021) |
| **Nginx Caching Proxy** | Standard Nginx | On-demand | ~30 min | Automatic (live proxy) | Yes | N/A |
| **WikiTaxi** | None (Windows only) | ~8 GB compressed | Easy | Manual | No | Discontinued |

### Recommendation: Kiwix

Kiwix is the clear winner for this use case:
- Official Docker image, actively maintained
- Single container + single ZIM file = minimal complexity
- Built-in full-text search
- Runs on minimal hardware (even a Raspberry Pi)
- Can serve additional content (Stack Exchange, Wiktionary, etc.)

The only trade-off is that updates require re-downloading the entire ZIM file (~115 GB for full English Wikipedia with images), and content is a periodic snapshot (rebuilt roughly every 6 months).

---

## Implementation Steps

### Phase 1: Initial Setup

#### Step 1 — Choose a ZIM variant

Pick one based on available disk space:

| Variant | File | Size | Content |
|---------|------|------|---------|
| Full + images | `wikipedia_en_all_maxi_YYYY-MM.zim` | ~115 GB | All articles + images |
| Text only | `wikipedia_en_all_nopic_YYYY-MM.zim` | ~48 GB | All articles, no images |
| Mini | `wikipedia_en_all_mini_YYYY-MM.zim` | ~12 GB | Introductions only + images |

Download from: `https://download.kiwix.org/zim/wikipedia/`

#### Step 2 — Create the directory structure

```bash
mkdir -p /path/to/localwiki/zims
mkdir -p /path/to/localwiki/scripts
```

#### Step 3 — Download the ZIM file

```bash
# Full English Wikipedia with images (~115 GB)
cd /path/to/localwiki/zims
# Check https://download.kiwix.org/zim/wikipedia/ for the latest filename
wget -c https://download.kiwix.org/zim/wikipedia/wikipedia_en_all_maxi_YYYY-MM.zim
```

> The `-c` flag enables resume if the download is interrupted.

**Expected download times:**

| Connection | ~115 GB (maxi) | ~48 GB (nopic) | ~12 GB (mini) |
|------------|---------------|----------------|---------------|
| 100 Mbps | ~2.5 hours | ~1 hour | ~16 min |
| 50 Mbps | ~5 hours | ~2 hours | ~32 min |
| 25 Mbps | ~10 hours | ~4 hours | ~64 min |
| 10 Mbps | ~25 hours | ~10 hours | ~2.7 hours |

> Kiwix's download server may throttle speeds. Consider using a torrent if available for faster downloads of the larger variants.

#### Step 3b — Verify the download

Kiwix publishes SHA-256 checksums alongside each ZIM file:

```bash
cd /path/to/localwiki/zims
wget https://download.kiwix.org/zim/wikipedia/wikipedia_en_all_maxi_YYYY-MM.zim.sha256
sha256sum -c wikipedia_en_all_maxi_YYYY-MM.zim.sha256
```

> For a 115 GB file, corruption during download is a real risk. Always verify before deleting an older working copy.

#### Step 4 — Create `docker-compose.yml`

```yaml
services:
  kiwix:
    image: ghcr.io/kiwix/kiwix-serve:3.8.2  # Pin version for predictability
    container_name: localwiki
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./zims:/data
    command: '*.zim'
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080"]
      interval: 60s
      timeout: 5s
      retries: 3
```

#### Step 5 — Start the container

```bash
cd /path/to/localwiki
docker compose up -d
```

Wikipedia is now accessible at `http://<your-lan-ip>:8080`.

#### Step 6 — Verify LAN access

From another device on your network, open `http://<host-ip>:8080` in a browser.

If it doesn't load, check:
- **Firewall**: Ensure port 8080 is open on the host (`sudo ufw allow 8080/tcp` on Ubuntu, or check macOS firewall settings under System Settings > Network > Firewall)
- **Docker binding**: To restrict access to a specific interface, change the port mapping to `"192.168.x.x:8080:8080"`
- **Container health**: Run `docker ps` and check the container status and health

---

### Phase 2: Automated Updates

ZIM files for English Wikipedia are rebuilt roughly every 6 months. Two options for keeping current:

#### Option A — Cron script (simple)

Create `scripts/update-zim.sh`:

```bash
#!/bin/bash
set -euo pipefail

# Downloads the latest English Wikipedia ZIM file
# Run via cron: 0 3 1 */3 * /path/to/localwiki/scripts/update-zim.sh

# ── Configuration ──────────────────────────────────────────────
ZIM_DIR="/path/to/localwiki/zims"
BASE_URL="https://download.kiwix.org/zim/wikipedia"
VARIANT="wikipedia_en_all_maxi"  # Change to nopic or mini if preferred
CONTAINER_NAME="localwiki"
MIN_FREE_GB=250  # Minimum free disk space required (2x ZIM size + buffer)

# Notifications (optional — leave empty to disable)
# For email: set NOTIFY_CMD="mail -s 'LocalWiki Update' you@example.com"
# For ntfy:  set NOTIFY_CMD="curl -s -d @- https://ntfy.sh/your-topic"
# For Slack: set NOTIFY_CMD="curl -s -X POST -d @- https://hooks.slack.com/..."
NOTIFY_CMD=""

# ── Helper functions ──────────────────────────────────────────
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

notify() {
  log "$1"
  if [ -n "$NOTIFY_CMD" ]; then
    echo "$1" | eval "$NOTIFY_CMD" >/dev/null 2>&1 || true
  fi
}

# ── Pre-flight: disk space check ──────────────────────────────
FREE_GB=$(df -BG "$ZIM_DIR" | awk 'NR==2 {print int($4)}')
if [ "$FREE_GB" -lt "$MIN_FREE_GB" ]; then
  notify "SKIP: Only ${FREE_GB}GB free on disk (need ${MIN_FREE_GB}GB). Skipping update."
  exit 1
fi

# ── Find the latest available ZIM ─────────────────────────────
LATEST=$(curl -sf "$BASE_URL/" \
  | grep -oP "${VARIANT}_\d{4}-\d{2}\.zim" \
  | sort -V | tail -1)

if [ -z "$LATEST" ]; then
  notify "ERROR: Could not determine latest ZIM file from $BASE_URL"
  exit 1
fi

if [ -f "$ZIM_DIR/$LATEST" ]; then
  log "Already have latest: $LATEST"
  exit 0
fi

# ── Download ──────────────────────────────────────────────────
log "Downloading $LATEST..."
wget -c "$BASE_URL/$LATEST" -O "$ZIM_DIR/$LATEST.part"

if [ $? -ne 0 ]; then
  notify "ERROR: Download of $LATEST failed"
  # Keep .part file so wget -c can resume next run
  exit 1
fi

mv "$ZIM_DIR/$LATEST.part" "$ZIM_DIR/$LATEST"

# ── Verify checksum ───────────────────────────────────────────
log "Verifying checksum..."
wget -q "$BASE_URL/$LATEST.sha256" -O "$ZIM_DIR/$LATEST.sha256"

if ! (cd "$ZIM_DIR" && sha256sum -c "$LATEST.sha256"); then
  notify "ERROR: Checksum verification failed for $LATEST. Keeping old ZIM."
  rm -f "$ZIM_DIR/$LATEST" "$ZIM_DIR/$LATEST.sha256"
  exit 1
fi
rm -f "$ZIM_DIR/$LATEST.sha256"

# ── Validate: restart container and confirm it starts ─────────
log "Restarting $CONTAINER_NAME with new ZIM..."
docker restart "$CONTAINER_NAME"
sleep 5

if ! docker exec "$CONTAINER_NAME" wget -q --spider http://localhost:8080 2>/dev/null; then
  notify "ERROR: Container failed health check after update. Rolling back."
  rm -f "$ZIM_DIR/$LATEST"
  docker restart "$CONTAINER_NAME"
  exit 1
fi

# ── Cleanup: remove old ZIM files only after validation ───────
OLD_ZIMS=$(find "$ZIM_DIR" -name "${VARIANT}_*.zim" ! -name "$LATEST" -type f)
if [ -n "$OLD_ZIMS" ]; then
  echo "$OLD_ZIMS" | xargs rm -f
  log "Removed old ZIM files"
fi

notify "SUCCESS: LocalWiki updated to $LATEST"
```

```bash
chmod +x scripts/update-zim.sh

# Run quarterly at 3 AM on the 1st of the month
crontab -e
# Add: 0 3 1 */3 * /path/to/localwiki/scripts/update-zim.sh >> /path/to/localwiki/update.log 2>&1
```

#### Option B — kiwix-zim-updater (community tool)

The [kiwix-zim-updater](https://github.com/jojo2357/kiwix-zim-updater) project automates checking and downloading newer ZIM files. It supports checksum verification and can manage multiple ZIM files.

```bash
git clone https://github.com/jojo2357/kiwix-zim-updater.git scripts/kiwix-zim-updater
# Run: ./scripts/kiwix-zim-updater/kiwix-zim-updater.sh /path/to/localwiki/zims
```

---

### Phase 3: Optional Enhancements

#### Serve additional content

Kiwix can serve multiple ZIM files simultaneously. Add more to the `zims/` directory:

- **Stack Overflow**: `stackoverflow.com_en_all_YYYY-MM.zim`
- **Wiktionary**: `wiktionary_en_all_YYYY-MM.zim`
- **WikiBooks**: `wikibooks_en_all_YYYY-MM.zim`

Browse available ZIMs at `https://download.kiwix.org/zim/`.

#### Reverse proxy with HTTPS (optional)

If you want a clean URL like `https://wiki.local`, put it behind Traefik, Caddy, or Nginx:

```yaml
# Example addition to docker-compose.yml using Caddy
services:
  caddy:
    image: caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
    depends_on:
      - kiwix

volumes:
  caddy_data:
```

```
# Caddyfile
wiki.local {
    reverse_proxy kiwix:8080
    tls internal
}
```

---

## Storage & Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| Disk (full + images) | 130 GB (one copy) | 250 GB (room for download + old copy during update) |
| Disk (text only) | 55 GB | 110 GB |
| RAM | 256 MB | 1 GB |
| CPU | Any (single core fine) | Any |

Kiwix-serve is extremely lightweight. The bottleneck is disk space and download bandwidth.

---

## Quick Start Checklist

- [ ] Decide on ZIM variant (maxi / nopic / mini)
- [ ] Provision disk space (2x ZIM size for update headroom)
- [ ] Download the ZIM file
- [ ] Verify download checksum
- [ ] Create `docker-compose.yml` (with pinned image version + healthcheck)
- [ ] `docker compose up -d`
- [ ] Verify at `http://<lan-ip>:8080`
- [ ] Verify access from another device on the LAN (check firewall if needed)
- [ ] Set up update cron job (with disk check, checksum, rollback, notifications)
- [ ] (Optional) Add reverse proxy for clean URL
- [ ] (Optional) Download additional ZIM files
