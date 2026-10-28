# Media Server - Stack completo con Jellyfin

Stack Docker completo per gestire automaticamente **film** e **serie TV** con audio in **italiano** e qualità minima **1080p**.

## Servizi inclusi

| Servizio     | Ruolo                                      | Porta interna |
|--------------|--------------------------------------------|---------------|
| **Jellyfin**     | Media server (riproduzione)                | 8096          |
| **qBittorrent**  | Client torrent (download)                  | 8080          |
| **Radarr**       | Gestione automatica film                   | 7878          |
| **Sonarr**       | Gestione automatica serie TV               | 8989          |
| **Prowlarr**     | Gestore centralizzato indexer              | 9696          |
| **Jellyseerr**   | Interfaccia richieste (tipo Netflix)       | 5055          |

Tutte le Web UI sono accessibili **solo** tramite la rete Docker `proxy-manager` (es. Nginx Proxy Manager).  
Solo le porte torrent di qBittorrent (6881) sono esposte sull'host.

---

## Struttura del progetto

```
media-server/
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
├── AGENT.md
│
├── config/                     # Configurazioni dei servizi (non versionate)
│   ├── jellyfin/
│   ├── qbittorrent/
│   ├── radarr/
│   ├── sonarr/
│   ├── prowlarr/
│   └── jellyseerr/
│
└── data/
    ├── downloads/              # Dove scarica qBittorrent
    │   ├── incomplete/
    │   └── completed/
    │
    └── media/                  # Libreria definitiva per Jellyfin
        ├── Film/               # Film organizzati da Radarr
        └── Serie TV/           # Serie organizzate da Sonarr
```

---

## Flusso di funzionamento (spiegato semplice)

1. **Tu** apri **Jellyseerr** e cerchi un film o una serie.
2. Clicchi **Request** (puoi chiedere la versione in italiano).
3. **Jellyseerr** invia la richiesta a **Radarr** (film) o **Sonarr** (serie).
4. **Radarr/Sonarr** chiedono a **Prowlarr** di cercare release **in italiano** e **minimo 1080p**.
5. **Prowlarr** interroga gli indexer configurati e restituisce i risultati.
6. **Radarr/Sonarr** scelgono la migliore release e la mandano a **qBittorrent**.
7. **qBittorrent** scarica il file in `data/downloads/completed`.
8. Quando il download è finito:
   - **Radarr** crea la cartella `Film/Nome Film (Anno)/` e rinomina il file.
   - **Sonarr** crea `Serie TV/Nome Serie/Season 01/` e rinomina gli episodi.
9. **Jellyfin** vede immediatamente i nuovi file e li rende disponibili per la visione.

Tutto avviene in automatico dopo la configurazione iniziale.

---

## Installazione rapida

### 1. Prerequisiti

- Docker + Docker Compose installati
- Rete Docker esterna già creata:

```bash
docker network create proxy-manager
```

### 2. Clona il progetto

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server
```

### 3. Crea le cartelle e i file .gitkeep (se non presenti)

```bash
mkdir -p config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,jellyseerr}
mkdir -p data/downloads/{incomplete,completed}
mkdir -p data/media/Film
mkdir -p "data/media/Serie TV"

touch config/jellyfin/.gitkeep config/qbittorrent/.gitkeep config/radarr/.gitkeep
touch config/sonarr/.gitkeep config/prowlarr/.gitkeep config/jellyseerr/.gitkeep
touch data/downloads/.gitkeep data/downloads/incomplete/.gitkeep data/downloads/completed/.gitkeep
touch data/media/Film/.gitkeep "data/media/Serie TV/.gitkeep"
```

### 4. Configura il file `.env`

Apri `.env` e modifica se necessario:

```env
PUID=1001          # ID utente del tuo sistema
PGID=1001          # ID gruppo del tuo sistema
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Per trovare il tuo PUID/PGID: `id`

### 5. Avvia lo stack

```bash
docker compose up -d
```

### 6. Configura Nginx Proxy Manager

Crea un Proxy Host per ogni servizio:

| Servizio     | Hostname interno | Porta | Esempio dominio              |
|--------------|------------------|-------|------------------------------|
| Jellyfin     | jellyfin         | 8096  | jellyfin.tuodominio.com      |
| qBittorrent  | qbittorrent      | 8080  | qbittorrent.tuodominio.com   |
| Radarr       | radarr           | 7878  | radarr.tuodominio.com        |
| Sonarr       | sonarr           | 8989  | sonarr.tuodominio.com        |
| Prowlarr     | prowlarr         | 9696  | prowlarr.tuodominio.com      |
| Jellyseerr   | jellyseerr       | 5055  | requests.tuodominio.com      |

---

## Configurazione dettagliata dei servizi

### 1. qBittorrent

1. Apri l'interfaccia web di qBittorrent.
2. Vai su **Impostazioni** (ingranaggio).
3. **Download**:
   - Cartella di salvataggio predefinita: `/downloads/completed`
   - Tieni i file incompleti in: `/downloads/incomplete`
4. **Connessione**: Porta di ascolto `6881`
5. **Web UI**: Imposta username e password.
6. Salva.

### 2. Prowlarr (Indexer)

1. Apri Prowlarr.
2. **Settings → General** → copia l'**API Key**.
3. **Indexers → Add Indexer** → aggiungi gli indexer che preferisci.
4. Per ogni indexer imposta priorità più alta a quelli con buone release italiane.
5. **Settings → Apps** → aggiungerai Radarr e Sonarr dopo averli configurati.

### 3. Radarr (Film)

#### 3.1 Media Management
1. **Settings → Media Management**
2. **Root Folders** → aggiungi `/movies`
3. **Movie Naming**:
   - Standard Movie Format: `{Movie Title} ({Release Year})`
   - Movie Folder Format: `{Movie Title} ({Release Year})`
4. Attiva **Rename Movies** e **Replace Illegal Characters**.

#### 3.2 Quality Profile (minimo 1080p + Italiano)
1. **Settings → Profiles** → crea nuovo profilo `Italiano 1080p`
2. Abilita **solo**: Bluray-1080p, WEBDL-1080p, WEBRip-1080p, HDTV-1080p
3. Disabilita tutte le qualità inferiori a 1080p.
4. Imposta questo profilo come **Default**.

#### 3.3 Custom Formats (preferenza Italiano)
1. **Settings → Custom Formats → Add**
2. Crea formato **Italiano**: Condition → Release Title → Regex: `\b(ita|italian|iTALiAN|iTA|subita)\b`
3. Crea formato **Non Italiano**: Condition → Release Title → Regex: `\b(eng|english|sub|subs|subbed)\b`
4. Nel profilo `Italiano 1080p` assegna:
   - Italiano → **+100**
   - Non Italiano → **-200**
5. **Minimum Custom Format Score**: `50`
6. **Upgrade Until Custom Format Score**: `100`

#### 3.4 Download Client
1. **Settings → Download Clients → Add → qBittorrent**
2. Host: `qbittorrent` | Port: `8080` | Category: `radarr`
3. Username e Password di qBittorrent → Test e Salva.

#### 3.5 Collegamento a Prowlarr
1. In **Prowlarr → Settings → Apps → Add Application → Radarr**
2. Prowlarr Server: `http://prowlarr:9696`
3. Radarr Server: `http://radarr:7878`
4. API Key di Radarr → Sync App Indexers

### 4. Sonarr (Serie TV)

#### 4.1 Media Management
1. **Root Folders** → aggiungi `/tv`
2. **Episode Naming**:
   - Standard Episode Format: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
   - Season Folder Format: `Season {season:00}`
   - Series Folder Format: `{Series Title}`
3. Attiva **Rename Episodes**.

#### 4.2 Quality Profile + Custom Formats
Ripeti gli stessi passaggi di Radarr (profilo `Italiano 1080p`, Custom Formats +100 / -200, Minimum Score 50).

#### 4.3 Download Client
Host: `qbittorrent` | Port: `8080` | Category: `sonarr`

#### 4.4 Collegamento a Prowlarr
Stesso procedimento di Radarr (da Prowlarr → Apps → Sonarr).

### 5. Jellyseerr

1. Segui la procedura guidata.
2. **Jellyfin**: URL `http://jellyfin:8096` + credenziali admin.
3. **Radarr**: URL `http://radarr:7878`, Root `/movies`, Quality Profile `Italiano 1080p`.
4. **Sonarr**: URL `http://sonarr:8989`, Root `/tv`, Quality Profile `Italiano 1080p`.

### 6. Jellyfin

1. Libreria **Film** → cartella `/media/Film`
2. Libreria **Serie TV** → cartella `/media/Serie TV`

---

## Test finale

1. Vai su **Jellyseerr** e richiedi un film italiano.
2. Controlla: Radarr (Queue) → qBittorrent (download) → file in `data/media/Film/Nome Film (Anno)/`.
3. Ripeti con una serie TV.

---

## Comandi utili

```bash
docker compose up -d
docker compose down
docker compose pull && docker compose up -d
docker compose logs -f radarr
```

---

## Note importanti

- Le cartelle `config/` e `data/` non vengono versionate (grazie al `.gitignore`).
- I file `.gitkeep` mantengono la struttura delle cartelle vuote su GitHub.
- Cambia `PUID` e `PGID` nel file `.env` in base al tuo utente (`id`).
- Assicurati che la rete `proxy-manager` esista prima di avviare lo stack.
