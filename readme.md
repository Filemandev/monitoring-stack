# 📊 Monitoring Stack

Een self-hosted monitoring stack uitgerold via Docker Compose. De stack biedt inzicht in systeemgezondheid, containermetrics en uptime van services.

De instructies zijn geschreven voor Debian/Ubuntu, maar Docker draait op elke Linux-distributie.
Gebruik je Windows of macOS? Dan kun je [Docker Desktop](https://docs.docker.com/desktop/) gebruiken.

---

## Docker installeren en omgeving voorbereiden

Installeer Docker en Docker Compose via de [officiële documentatie](https://docs.docker.com/compose/).
Ga naar **Install → Plugin → Install using the Repository → Ubuntu** (of jouw OS), of ga direct naar [deze pagina](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

Voer de commando's uit die daar staan, vergelijkbaar met:

```bash
# Voeg Docker's officiële GPG-sleutel toe:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Voeg de repository toe aan de apt-bronnen:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Installeer daarna de Docker-pakketten en controleer of alles werkt:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run hello-world
```

Controleer of Docker Compose correct is geïnstalleerd:

```bash
docker compose version
```

### Mappenstructuur aanmaken

Maak de benodigde mappen aan voor de monitoring stack:

```bash
sudo mkdir -p /opt/monitoring/{prometheus,grafana,uptime-kuma}
sudo chown -R 1000:1000 /opt/monitoring
sudo chmod -R a=,a+rX,u+w,g+w /opt/monitoring
ls -ln /opt/monitoring
```

### Docker Compose bestand

Maak de projectmap aan en kopieer het `docker-compose.yml` bestand:

```bash
mkdir -p ~/monitoring
cd ~/monitoring
sudo nano docker-compose.yml
```

---

## Projectstructuur

```
monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml          # Scrape configuratie
├── grafana/                    # Grafana data (persistent volume)
└── uptime-kuma/                # Uptime Kuma data (persistent volume)
```

---

## Eerste keer opstarten

Start alle services met één commando:

```bash
sudo docker compose up -d
```

Controleer of alle containers draaien:

```bash
docker compose ps
```

---

## Componenten

| Service | Image | Poort | Beschrijving |
|---|---|---|---|
| **Prometheus** | `prom/prometheus:latest` | `9090` | Metrics scraping & opslag |
| **Grafana** | `grafana/grafana:latest` | `3000` | Visualisatie & dashboards |
| **Node Exporter** | `prom/node-exporter:latest` | host network | Host-level systeemmetrics (CPU, RAM, disk) |
| **cAdvisor** | `gcr.io/cadvisor/cadvisor:latest` | `8080` | Container-level resource metrics |
| **Uptime Kuma** | `louislam/uptime-kuma:latest` | `3001` | Uptime monitoring & alerting UI |

> Alle services starten automatisch opnieuw op bij een crash of reboot (`restart: unless-stopped`).

---

## Services configureren

### Prometheus

Maak het configuratiebestand aan op `prometheus/prometheus.yml`:

```bash
sudo nano ~/monitoring/prometheus/prometheus.yml
```

Voeg de volgende scrape targets toe:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']   # host network mode

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

> Node Exporter draait in `network_mode: host` en is daarom bereikbaar via `localhost:9100` in plaats van de containernaam.

Herstart Prometheus na een configuratiewijziging:

```bash
docker compose restart prometheus
```

---

### Grafana

Open: `http://<host-ip>:3000`

- **Standaard gebruiker:** `admin`
- **Standaard wachtwoord:** `admin` *(wijzig dit direct na eerste login)*

Data wordt opgeslagen in `./grafana` op de host.

#### Prometheus koppelen

Ga naar **Connections → Data Sources → + Add new data source → Prometheus** en stel in:

- **URL**: `http://prometheus:9090`

Klik op **Save & test**.

#### Aanbevolen dashboards

Importeer via **Dashboards → + Import** met de volgende IDs van [grafana.com/dashboards](https://grafana.com/dashboards):

| Dashboard | ID |
|---|---|
| Node Exporter Full | `1860` |
| cAdvisor | `14282` |

---

### Node Exporter

Node Exporter draait in `network_mode: host` en heeft geen webinterface. Metrics zijn beschikbaar op:

- `http://localhost:9100/metrics`

Controleer of de metrics correct binnenkomen in Prometheus via **Status → Targets** op `http://<host-ip>:9090`.

---

### cAdvisor

Open: `http://<host-ip>:8080`

cAdvisor heeft geen verdere configuratie nodig. Het leest automatisch de Docker-socket en host-mounts uit en exposeert containermetrics aan Prometheus.

---

### Uptime Kuma

Open: `http://<host-ip>:3001`

Maak bij eerste gebruik een admin account aan. Voeg daarna monitors toe voor je services via **+ Add New Monitor**:

- **Monitor Type**: `HTTP(s)` voor webinterfaces, `TCP Port` voor interne services
- **Friendly Name**: herkenbare naam (bijv. `Grafana`)
- **URL**: bijv. `http://<host-ip>:3000`

Data wordt opgeslagen in `./uptime-kuma` op de host.

---

## Volumes

| Service | Host pad | Container pad | Doel |
|---|---|---|---|
| Prometheus | `./prometheus/prometheus.yml` | `/etc/prometheus/prometheus.yml` | Scrape configuratie |
| Grafana | `./grafana` | `/var/lib/grafana` | Dashboards & instellingen |
| Uptime Kuma | `./uptime-kuma` | `/app/data` | Monitors & instellingen |
| cAdvisor | `/`, `/var/run`, `/sys`, `/var/lib/docker` | diverse paden (read-only) | Toegang tot Docker & host metrics |

---

## Services herstarten

Start alle services opnieuw om te controleren of alles correct opstart:

```bash
sudo docker compose down
sudo docker compose up -d
```

---

## 🔒 Beveiliging

- Stel sterke wachtwoorden in voor Grafana en Uptime Kuma
- Bind poorten niet aan `0.0.0.0` in productie — gebruik een reverse proxy (bijv. Nginx of Traefik)
- Gebruik een `.env` bestand voor gevoelige waarden en voeg dit toe aan `.gitignore`

---

## Probleemoplossing

### Containers controleren

```bash
# Status van alle containers
docker compose ps

# Logs bekijken van een specifieke service
docker compose logs -f prometheus

# Specifieke container inspecteren
docker inspect prometheus
```

### Prometheus targets controleren

Ga naar `http://<host-ip>:9090/targets` en controleer of alle targets de status **UP** hebben. Een **DOWN** status geeft aan dat Prometheus de betreffende exporter niet kan bereiken.

### Node Exporter niet bereikbaar

Node Exporter draait in `network_mode: host`. Controleer of poort `9100` luistert:

```bash
ss -tlnp | grep 9100
```

### Alle images bijwerken

```bash
docker compose pull && docker compose up -d
```

---

## 📝 Licentie

MIT
