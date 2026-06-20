# Apache Superset Templates

A collection of dashboard CSS themes, Handlebars chart templates, and Claude Code skills for working with Apache Superset.

---

## Repository Structure

```
superset_templates/
├── Templates/          # Dashboard CSS themes (paste into Edit Dashboard → CSS)
├── skills/             # Claude Code agent skills for Superset tasks
└── install/            # Docker Compose files and quick-start commands
```

---

## Dashboard CSS Themes

Apply a theme by opening a dashboard, clicking **Edit Dashboard → CSS**, and pasting the contents of the chosen file.

### Transport Blue (`Templates/Transport Blue.txt`)

Grain logistics and transport style. Deep maritime blues (`#1E3A8A`), barley gold (`#EAB308`), silo grays, and cream background (`#FDFBF7`). Compact layout with reduced padding — optimized for dense production dashboards.

**Palette highlights:**
| Role | Color |
|---|---|
| Background | `#FDFBF7` |
| Primary / Headers | `#1E3A8A` |
| Accent / Buttons | `#EAB308` |
| Text | `#0F172A` |

---

### Provence Light (`Templates/Provence Light Theme Segoe UI.txt`)

Warm Mediterranean style. Cream background (`#FDF6E3`), sunflower gold (`#E8B84B`), lavender accents (`#9B89B0`), and soft terracotta. Uses Georgia/Palatino serif fonts.

**Palette highlights:**
| Role | Color |
|---|---|
| Background | `#FDF6E3` |
| Cards | `#FFF8ED` |
| Primary Accent | `#E8B84B` |
| Link / Button | `#C9891A` |
| Text | `#3D2B1F` |

---

### Brutalist Light Black-White (`Templates/Brutalist Light Black-White Segoe UI.txt`)

Minimal competitive style. Pure grays (`#f5f5f5` background, `#ffffff` cards), no rounded corners, no shadows. Clean and distraction-free for analytical dashboards.

**Palette highlights:**
| Role | Color |
|---|---|
| Background | `#f5f5f5` |
| Cards | `#ffffff` |
| Buttons | `#6f6f6f` |
| Text | `#1f1f1f` |

---

## Claude Code Skills

### Handlebars Table (`skills/superset-handlebars-table.md`)

A step-by-step skill for building production-ready metrics tables using the Superset **Handlebars** chart type. Covers:

- Virtual Dataset SQL with fiscal year/month logic and plan-vs-actual deviation
- Chart configuration (dimensions, metrics, avoiding string columns as metrics)
- A complete ready-to-paste Handlebars + CSS Grid template
- All 7 critical rules that prevent raw HTML from rendering in the browser

**Key rules enforced by this skill:**
1. Use CSS Grid divs — never `<table>` (DOMPurify strips table elements)
2. No HTML comments (`<!-- -->` breaks the parser)
3. No inline `style=""` on container divs (stripped by sanitizer)
4. Put `<style>` inside the Handlebars Template field, not the CSS Styles field
5. Set `HTML_SANITIZATION = False` in `superset_config.py`
6. Use bracket notation for aggregated columns: `{{this.[SUM(feed_tons)]}}`
7. Use HTML entities for special characters: `&#9650;` `&#9660;` `&#8212;`

**Trigger phrases:** "Handlebars table in Superset", "metrics table", "plan vs actual", "Abweichung", "color-coded cells in Superset dashboard"

---

## Installation

### Quick Start — Pre-built Image (recommended)

Uses the official Apache Superset Docker image. No build required.

```bash
# Start
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml up -d

# Stop (keeps data)
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml stop

# Start again
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml start

# Remove containers (keeps volumes)
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml down
```

Superset runs on `http://localhost:8088`. PostgreSQL is exposed on port `5434`.

---

### Development Build

For local development with live frontend reloading:

```bash
# Clone the repo
git clone https://github.com/apache/superset
cd superset
git checkout tags/4.1.2   # or the desired version

# Disable example data loading
echo "SUPERSET_LOAD_EXAMPLES=no" > docker/.env-local

# Start all services (builds from source)
docker compose up -d
```

Services: Superset (`8088`), Nginx (`80`), Redis (`6379`), PostgreSQL (`5433`), Webpack dev server (`9000`).

---

### Configuration (`superset_config.py`)

Located at `superset/docker/pythonpath_dev/superset_config.py`:

```python
APP_NAME = "SUPERSET"
LOGO_RIGHT_TEXT = "Betrieb"
APP_ICON = "/static/assets/images/lewens_logo.png"
SUPERSET_TIMEZONE = "Europe/Berlin"

# Required for Handlebars <style> tags to render correctly
HTML_SANITIZATION = False
```

**Copy a custom logo into the container:**

```bash
docker cp lewens_logo.png superset_app:/app/superset/static/assets/images/lewens_logo.png
```

**Restart after config changes:**

```bash
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml down
TAG=6.1.0 docker compose -f docker-compose-image-tag.yml up -d
```

---

### Network Route (Taktstrasse Gateway)

```bash
sudo ip route add 192.168.11.0/24 via 192.168.0.21
```

For a persistent route, add to `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:
      dhcp4: true
      routes:
        - to: 192.168.11.0/24
          via: 192.168.0.21
```

```bash
netplan apply
```
