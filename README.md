# Media Server - Stack completo con Jellyfin

Stack Docker completo per gestire automaticamente **film** e **serie TV** con audio in **italiano** e qualità minima **1080p**.

## Servizi inclusi

| Servizio         | Cosa fa                                                                 | Porta interna |
|------------------|-------------------------------------------------------------------------|---------------|
| **Jellyfin**     | Media server (riproduzione)                                             | 8096          |
| **qBittorrent**  | Client torrent (download)                                               | 8080          |
| **Radarr**       | Gestione automatica dei film                                            | 7878          |
| **Sonarr**       | Gestione automatica delle serie TV                                      | 8989          |
| **Prowlarr**     | Gestore centralizzato degli indexer                                     | 9696          |
| **Seerr**        | Interfaccia per richiedere film e serie                                 | 5055          |
| **FlareSolverr** | Risolve i controlli Cloudflare (“Non sono un robot”) degli indexer     | 8191          |

Tutte le interfacce web sono accessibili **solo** attraverso la rete Docker `proxy-manager` (es. Nginx Proxy Manager).  
Sull'host viene esposta **solo** la porta torrent di qBittorrent (`6881`).  
**FlareSolverr non ha interfaccia web**: non va aggiunto al reverse proxy.

---

## Come funziona il flusso

1. Apri **Seerr** e richiedi un film o una serie.
2. Seerr invia la richiesta a **Radarr** (film) o **Sonarr** (serie).
3. Radarr/Sonarr chiedono a **Prowlarr** release in italiano e minimo 1080p.
4. Se un indexer ha Cloudflare, **FlareSolverr** supera il controllo.
5. **qBittorrent** scarica in `data/downloads/completed`.
6. Radarr crea `Film/Nome Film (Anno)/` e rinomina il file (con eventuali `.srt`, `.nfo`, ecc.).
7. Sonarr crea `Serie TV/Nome Serie (Anno)/Season 01/` e rinomina gli episodi.
8. **Jellyfin** mostra i nuovi contenuti.

---

## Struttura cartelle

```
media-server/
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
├── AGENT.md
├── config/
│   ├── jellyfin/
│   ├── qbittorrent/
│   ├── radarr/
│   ├── sonarr/
│   ├── prowlarr/
│   └── seerr/
└── data/
    ├── downloads/incomplete/
    ├── downloads/completed/
    └── media/
        ├── Film/Nome Film (2024)/
        └── Serie TV/Nome Serie (2020)/Season 01/
```

---

## Installazione

### 1. Rete Docker

```bash
docker network create proxy-manager
```

### 2. Clona e prepara

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server

mkdir -p config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,seerr}
mkdir -p data/downloads/{incomplete,completed}
mkdir -p data/media/Film
mkdir -p "data/media/Serie TV"
```

### 3. File `.env`

```env
PUID=1001
PGID=1001
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Trova PUID/PGID con: `id`

### 4. Avvio

```bash
docker compose up -d
docker compose ps
```

### 5. Nginx Proxy Manager

| Servizio    | Hostname interno | Porta |
|-------------|------------------|-------|
| Jellyfin    | jellyfin         | 8096  |
| qBittorrent | qbittorrent      | 8080  |
| Radarr      | radarr           | 7878  |
| Sonarr      | sonarr           | 8989  |
| Prowlarr    | prowlarr         | 9696  |
| Seerr       | seerr            | 5055  |

Non creare Proxy Host per FlareSolverr.

---

## Configurazione servizi (ordine obbligatorio)

### 1. qBittorrent

1. Apri la Web UI e imposta username/password.
2. Impostazioni → **Download**:
   - Cartella salvataggio: `/downloads/completed`
   - Incompleti: `/downloads/incomplete`
3. **Connessione**: porta `6881`
4. Salva.

### 2. Prowlarr

1. Settings → General → copia **API Key**.
2. Indexers → Add Indexer → aggiungi gli indexer (Base URL già compilato; se fallisce prova un altro dominio dal menu).
3. Settings → Indexers → sezione **FlareSolverr**:
   - Name: `FlareSolverr`
   - Tags: `flaresolverr`
   - Host: `http://flaresolverr:8191`
   - Test → Save
4. Per ogni indexer bloccato da Cloudflare: apri l'indexer → Tags → aggiungi `flaresolverr` → Test → Save.

### 3. Radarr (Film)

#### Media Management

1. Settings → Media Management
2. Root Folder: `/movies`
3. Rename Movies: ON · Replace Illegal Characters: ON
4. Standard Movie Format: `{Movie Title} ({Release Year})`
5. Movie Folder Format: `{Movie Title} ({Release Year})`
6. **Import Extra Files**: ON  
   Estensioni: `srt,sub,idx,nfo,txt`
7. Save

#### Quality Profile `Italiano 1080p`

Abilita solo: Bluray-1080p, WEBDL-1080p, WEBRip-1080p, HDTV-1080p.  
Disabilita 720p e inferiori. Imposta come default.

#### Custom Formats

**Italiano** – Release Title regex (case-insensitive):

```regex
(?i)\b(ita|italian|italiano|subita|multi)\b
```

Score: **+100**

**Non Italiano** – Release Title regex:

```regex
(?i)\b(eng|english)\b
```

Score: **-200**

Nel profilo `Italiano 1080p`:
- Minimum Custom Format Score: `50`
- Upgrade Until Custom Format Score: `100`

#### Download Client

- Host: `qbittorrent` · Port: `8080` · Category: `radarr`
- Username/password di qBittorrent → Test → Save

#### Collegamento Prowlarr → Radarr

In Prowlarr → Settings → Apps → Add → Radarr:
- Prowlarr Server: `http://prowlarr:9696`
- Radarr Server: `http://radarr:7878`
- API Key di Radarr → Test → Save → Sync App Indexers

### 4. Sonarr (Serie TV)

#### Media Management

1. Root Folder: `/tv`
2. Rename Episodes: ON · Replace Illegal Characters: ON
3. Standard Episode Format: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
4. Season Folder Format: `Season {season:00}`
5. Series Folder Format: `{Series Title} ({Series Year})`
6. **Import Extra Files**: ON  
   Estensioni: `srt,sub,idx,nfo,txt`
7. Save

#### Quality Profile + Custom Formats

Stessi valori di Radarr:
- Profilo `Italiano 1080p` (solo qualità 1080p)
- Custom Format Italiano: `(?i)\b(ita|italian|italiano|subita|multi)\b` → +100
- Custom Format Non Italiano: `(?i)\b(eng|english)\b` → -200
- Minimum Score 50 · Upgrade Until 100

#### Download Client

- Host: `qbittorrent` · Port: `8080` · Category: `sonarr` → Test → Save

#### Collegamento Prowlarr → Sonarr

In Prowlarr → Settings → Apps → Add → Sonarr:
- Prowlarr Server: `http://prowlarr:9696`
- Sonarr Server: `http://sonarr:8989`
- API Key di Sonarr → Test → Save → Sync App Indexers

### 5. Seerr

1. Apri Seerr e segui la procedura guidata.
2. **Jellyfin**: URL `http://jellyfin:8096` + utente admin.
3. **Radarr**: URL `http://radarr:7878`, API Key, Root `/movies`, profilo `Italiano 1080p`.
4. **Sonarr**: URL `http://sonarr:8989`, API Key, Root `/tv`, profilo `Italiano 1080p`.
5. Completa e salva.

### 6. Jellyfin

1. Libreria **Film** → cartella `/media/Film`
2. Libreria **Serie TV** → cartella `/media/Serie TV`

---

## Test finale

1. In **Seerr** richiedi un film.
2. Controlla Radarr (Queue) → qBittorrent (categoria `radarr`) → cartella `data/media/Film/Nome (Anno)/`.
3. Ripeti con una serie in Sonarr / `Serie TV/Nome (Anno)/Season 01/`.
4. In Prowlarr, Test su un indexer con tag `flaresolverr`: deve essere verde.

---

## Comandi utili

```bash
docker compose up -d
docker compose down
docker compose pull && docker compose up -d
docker compose logs -f radarr
docker compose logs -f seerr
docker compose logs -f flaresolverr
docker compose ps
```

---

## Risorse

- Video guida utile (stack ARR / media server): https://youtu.be/-PQtE6Nb0Cw
- Dettaglio Import Extra Files: file `docs/IMPORT-EXTRA-FILES-E-RISORSE.md` nel repo

---

## Note

- `config/` e `data/` non sono versionati (vedi `.gitignore`).
- Cartelle film/serie includono **sempre l'anno** nel nome.
- Regex Custom Format con `(?i)` = **case-insensitive** (ita, ITA, Italiano, MULTI, ecc.).
- `MULTI` spesso indica dual audio (es. ITA+ENG). Se vuoi solo italiano “puro”, togli `multi` dalla regex.
- FlareSolverr non richiede l'installazione di Cloudflare sul server.
