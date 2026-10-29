# AGENT.md - Istruzioni per LLM / Agent

Questo file permette a qualsiasi LLM o agent di replicare **esattamente** questo media server su qualsiasi macchina Linux con Docker.

## Obiettivo

Stack Docker per:
- Film e Serie TV
- Preferenza audio italiano (Custom Formats case-insensitive)
- Qualità minima 1080p
- Web UI solo su rete Docker esterna `proxy-manager`
- Cartelle media `Film` e `Serie TV` con **anno** nel nome cartella
- **Seerr** per le richieste
- **FlareSolverr** per indexer protetti da Cloudflare

## Servizi

1. jellyfin
2. qbittorrent
3. radarr
4. sonarr
5. prowlarr
6. seerr
7. flaresolverr

## Struttura cartelle

```
media-server/
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
├── AGENT.md
├── config/
│   ├── jellyfin/.gitkeep
│   ├── qbittorrent/.gitkeep
│   ├── radarr/.gitkeep
│   ├── sonarr/.gitkeep
│   ├── prowlarr/.gitkeep
│   └── seerr/.gitkeep
└── data/
    ├── downloads/incomplete/.gitkeep
    ├── downloads/completed/.gitkeep
    └── media/Film/.gitkeep
    └── media/Serie TV/.gitkeep
```

## Passi agent

### 1. Struttura

```bash
mkdir -p media-server && cd media-server
mkdir -p config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,seerr}
mkdir -p data/downloads/{incomplete,completed}
mkdir -p data/media/Film "data/media/Serie TV"
touch config/jellyfin/.gitkeep config/qbittorrent/.gitkeep config/radarr/.gitkeep
touch config/sonarr/.gitkeep config/prowlarr/.gitkeep config/seerr/.gitkeep
touch data/downloads/.gitkeep data/downloads/incomplete/.gitkeep data/downloads/completed/.gitkeep
touch data/media/Film/.gitkeep "data/media/Serie TV/.gitkeep"
```

### 2. `.env`

```env
PUID=1001
PGID=1001
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Rilevare PUID/PGID con `id -u` / `id -g`.

### 3. docker-compose.yml

- Rete `proxy-manager` external
- Nessuna Web UI su host (solo `expose`)
- qBittorrent: porte `6881` TCP/UDP pubbliche
- Radarr volume: `${DATA_DIR}/media/Film:/movies`
- Sonarr volume: `${DATA_DIR}/media/Serie TV:/tv`
- **seerr**: image `ghcr.io/seerr-team/seerr:latest`, volume `${CONFIG_DIR}/seerr:/app/config`, expose `5055`
- **flaresolverr**: image `ghcr.io/flaresolverr/flaresolverr:latest`, expose `8191`
- Nessun riferimento a progetti legacy di request manager

### 4. Rete e avvio

```bash
docker network create proxy-manager || true
docker compose up -d
```

## Config post-avvio (istruzioni esplicite, senza riferimenti incrociati)

### qBittorrent

- Download: `/downloads/completed`
- Incomplete: `/downloads/incomplete`
- Porta: 6881

### Prowlarr

- Aggiungere indexer
- FlareSolverr Host: `http://flaresolverr:8191`, tag `flaresolverr`
- Assegnare tag solo agli indexer che falliscono per Cloudflare
- Apps: Radarr `http://radarr:7878`, Sonarr `http://sonarr:8989`

### Radarr

- Root: `/movies`
- Folder/File: `{Movie Title} ({Release Year})`
- Import Extra Files ON: `srt,sub,idx,nfo,txt`
- Profile `Italiano 1080p`: solo 1080p
- Custom Format Italiano regex: `(?i)\b(ita|italian|italiano|subita|multi)\b` score +100
- Custom Format Non Italiano regex: `(?i)\b(eng|english)\b` score -200
- Min score 50, Upgrade until 100
- Download client: qbittorrent:8080 category `radarr`

### Sonarr

- Root: `/tv`
- Series Folder: `{Series Title} ({Series Year})`
- Season: `Season {season:00}`
- Episode: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
- Import Extra Files ON: `srt,sub,idx,nfo,txt`
- Stessi Quality Profile e Custom Formats di Radarr (regex identiche, passi ripetuti per intero nell'UI)
- Download client: qbittorrent:8080 category `sonarr`

### Seerr

- Jellyfin: `http://jellyfin:8096`
- Radarr: `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr: `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

### Jellyfin

- Libreria Film: `/media/Film`
- Libreria Serie TV: `/media/Serie TV`

## Porte interne

| Servizio     | Hostname     | Porta |
|--------------|--------------|-------|
| jellyfin     | jellyfin     | 8096  |
| qbittorrent  | qbittorrent  | 8080  |
| radarr       | radarr       | 7878  |
| sonarr       | sonarr       | 8989  |
| prowlarr     | prowlarr     | 9696  |
| seerr        | seerr        | 5055  |
| flaresolverr | flaresolverr | 8191  |

## Vincoli

- Nessuna Web UI esposta sull'host
- Solo rete `proxy-manager`
- Naming con anno obbligatorio
- Regex sempre con flag `(?i)` per case-insensitive
- Non documentare né montare path di request manager legacy
- Istruzioni all'utente sempre complete e autonome (mai "stesso procedimento di...")
