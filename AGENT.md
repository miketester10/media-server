# AGENT.md - Istruzioni per LLM / Agent

Questo file permette a qualsiasi LLM o agent di replicare **esattamente** questo media server su qualsiasi macchina Linux con Docker.

## Obiettivo

Creare uno stack Docker completo per:
- Film e Serie TV
- Audio preferibilmente in italiano
- Qualità minima 1080p
- Accesso alle Web UI solo tramite rete Docker esterna chiamata `proxy-manager`
- Cartelle media in italiano (`Film` e `Serie TV`)

## Servizi da installare

1. **jellyfin** - Media server
2. **qbittorrent** - Client torrent
3. **radarr** - Automazione film
4. **sonarr** - Automazione serie TV
5. **prowlarr** - Gestore indexer
6. **jellyseerr** - Interfaccia richieste

## Struttura obbligatoria delle cartelle

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
│   └── jellyseerr/.gitkeep
└── data/
    ├── downloads/
    │   ├── incomplete/.gitkeep
    │   ├── completed/.gitkeep
    │   └── .gitkeep
    └── media/
        ├── Film/.gitkeep
        └── Serie TV/.gitkeep
```

## Passi che l'agent deve eseguire

### 1. Creare la struttura

```bash
mkdir -p media-server
cd media-server

mkdir -p config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,jellyseerr}
mkdir -p data/downloads/{incomplete,completed}
mkdir -p data/media/Film
mkdir -p "data/media/Serie TV"

touch config/jellyfin/.gitkeep
touch config/qbittorrent/.gitkeep
touch config/radarr/.gitkeep
touch config/sonarr/.gitkeep
touch config/prowlarr/.gitkeep
touch config/jellyseerr/.gitkeep
touch data/downloads/.gitkeep
touch data/downloads/incomplete/.gitkeep
touch data/downloads/completed/.gitkeep
touch data/media/Film/.gitkeep
touch "data/media/Serie TV/.gitkeep"
```

### 2. Creare il file `.env`

```env
PUID=1001
PGID=1001
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

**Nota**: L'agent deve chiedere o rilevare il PUID/PGID corretto dell'utente host (`id -u` e `id -g`).

### 3. Creare il `docker-compose.yml`

Il file deve contenere esattamente questi servizi con queste caratteristiche:

- **Rete**: `proxy-manager` (external: true)
- **Nessuna porta Web UI mappata sull'host** (solo `expose`)
- Solo qBittorrent espone le porte `6881:6881` e `6881:6881/udp`
- Volumi Radarr: `${DATA_DIR}/media/Film:/movies`
- Volumi Sonarr: `${DATA_DIR}/media/Serie TV:/tv`
- Tutti i servizi usano `${PUID}`, `${PGID}`, `${TZ}`

### 4. Creare `.gitignore`

Deve escludere:
- Tutto il contenuto di `config/**` tranne i `.gitkeep`
- Tutto il contenuto di `data/downloads/**` e `data/media/**` tranne i `.gitkeep`

### 5. Creare la rete Docker (se non esiste)

```bash
docker network create proxy-manager || true
```

### 6. Avviare lo stack

```bash
docker compose up -d
```

## Configurazione post-avvio (da spiegare all'utente)

L'agent deve guidare l'utente (o eseguire se ha accesso alle UI) su:

### qBittorrent
- Download folder: `/downloads/completed`
- Incomplete: `/downloads/incomplete`
- WEBUI_PORT: 8080
- Impostare username/password

### Radarr
- Root Folder: `/movies`
- Naming: `{Movie Title} ({Release Year})`
- Quality Profile: solo 1080p (Bluray/WEBDL/WEBRip/HDTV-1080p)
- Custom Format "Italiano" con regex `\b(ita|italian|iTALiAN|iTA|subita)\b` → score +100
- Custom Format "Non Italiano" con regex `\b(eng|english|sub|subs|subbed)\b` → score -200
- Minimum Custom Format Score: 50
- Download Client: qBittorrent (`qbittorrent:8080`)

### Sonarr
- Root Folder: `/tv`
- Episode Naming: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
- Season Folder: `Season {season:00}`
- Stessi Quality Profile e Custom Formats di Radarr
- Download Client: qBittorrent (`qbittorrent:8080`)

### Prowlarr
- Aggiungere indexer
- Collegare Radarr e Sonarr tramite Apps

### Jellyseerr
- Collegare Jellyfin (`http://jellyfin:8096`)
- Collegare Radarr (`http://radarr:7878`) con profilo `Italiano 1080p` e root `/movies`
- Collegare Sonarr (`http://sonarr:8989`) con profilo `Italiano 1080p` e root `/tv`

### Jellyfin
- Libreria Film → `/media/Film`
- Libreria Serie TV → `/media/Serie TV`

## Porte interne (documentazione)

| Servizio     | Hostname      | Porta |
|--------------|---------------|-------|
| jellyfin     | jellyfin      | 8096  |
| qbittorrent  | qbittorrent   | 8080  |
| radarr       | radarr        | 7878  |
| sonarr       | sonarr        | 8989  |
| prowlarr     | prowlarr      | 9696  |
| jellyseerr   | jellyseerr    | 5055  |

## Vincoli importanti

- Non esporre mai le Web UI sull'host (niente `ports:` per le UI).
- Usare sempre la rete esterna `proxy-manager`.
- Cartelle media devono chiamarsi `Film` e `Serie TV` sull'host.
- Preferenza lingua italiana + minimo 1080p deve essere configurata tramite Quality Profile + Custom Formats.
- I file di configurazione generati (`config/`) e i media (`data/`) non devono essere versionati.

## Output atteso dall'agent

Alla fine l'agent deve:
1. Aver creato tutta la struttura di cartelle
2. Aver scritto `docker-compose.yml`, `.env`, `.gitignore`, `README.md`, `AGENT.md`
3. Aver creato la rete `proxy-manager` (se mancante)
4. Aver avviato i container con `docker compose up -d`
5. Aver fornito all'utente le istruzioni chiare per la configurazione delle UI (Radarr, Sonarr, ecc.)
