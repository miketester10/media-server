# AGENT.md - Istruzioni per LLM / Agent

Questo file permette a qualsiasi LLM o agent di replicare **esattamente** questo media server su qualsiasi macchina Linux con Docker.

## Obiettivo

Creare uno stack Docker completo per:
- Film e Serie TV
- Audio preferibilmente in italiano
- Qualità minima 1080p
- Accesso alle Web UI solo tramite rete Docker esterna chiamata `proxy-manager`
- Cartelle media in italiano (`Film` e `Serie TV`)
- Nomi cartelle film e serie che includono **sempre l'anno**

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

Esempio struttura finale dopo i download:
```
data/media/Film/Il Padrino (1972)/
data/media/Serie TV/Breaking Bad (2008)/Season 01/
```

## Passi che l'agent deve eseguire

### 1. Creare la struttura

```bash
mkdir -p media-server
cd media-server

mkdir -p config/jellyfin
mkdir -p config/qbittorrent
mkdir -p config/radarr
mkdir -p config/sonarr
mkdir -p config/prowlarr
mkdir -p config/jellyseerr

mkdir -p data/downloads/incomplete
mkdir -p data/downloads/completed
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

L'agent deve chiedere o rilevare il PUID/PGID corretto dell'utente host con i comandi `id -u` e `id -g`.

### 3. Creare il `docker-compose.yml`

Il file deve contenere esattamente questi servizi con queste caratteristiche:

- Rete: `proxy-manager` con `external: true`
- Nessuna porta Web UI mappata sull'host (usare solo `expose`)
- Solo qBittorrent espone le porte `6881:6881` e `6881:6881/udp`
- Volume Radarr: `${DATA_DIR}/media/Film:/movies`
- Volume Sonarr: `${DATA_DIR}/media/Serie TV:/tv`
- Tutti i servizi usano le variabili `${PUID}`, `${PGID}`, `${TZ}`

### 4. Creare `.gitignore`

Deve escludere:
- Tutto il contenuto di `config/**` tranne i file `.gitkeep`
- Tutto il contenuto di `data/downloads/**` e `data/media/**` tranne i file `.gitkeep`

### 5. Creare la rete Docker (se non esiste)

```bash
docker network create proxy-manager || true
```

### 6. Avviare lo stack

```bash
docker compose up -d
```

## Configurazione post-avvio (istruzioni da dare all'utente in modo 100% esplicito)

L'agent deve spiegare ogni passaggio senza mai scrivere frasi del tipo "stesso procedimento di..." o "ripeti i passi di...". Ogni servizio deve avere i propri passi completi e autonomi.

### qBittorrent

1. Aprire l'interfaccia web di qBittorrent.
2. Impostare username e password al primo accesso.
3. Andare in Impostazioni (icona ingranaggio).
4. Sezione Download:
   - Cartella di salvataggio predefinita: `/downloads/completed`
   - Cartella file incompleti: `/downloads/incomplete`
5. Sezione Connessione: porta di ascolto `6881`.
6. Salvare.

### Radarr (Film) - naming con anno

1. Settings → Media Management.
2. Root Folder: aggiungere `/movies`.
3. Attivare Rename Movies e Replace Illegal Characters.
4. Standard Movie Format: `{Movie Title} ({Release Year})`
5. Movie Folder Format: `{Movie Title} ({Release Year})`
6. Settings → Profiles → creare profilo chiamato `Italiano 1080p`.
7. Nel profilo abilitare SOLO: Bluray-1080p, WEBDL-1080p, WEBRip-1080p, HDTV-1080p.
8. Settings → Custom Formats:
   - Creare formato `Italiano` con Condition Release Title e regex `\b(ita|italian|iTALiAN|iTA|subita)\b`
   - Creare formato `Non Italiano` con Condition Release Title e regex `\b(eng|english|sub|subs|subbed)\b`
9. Nel profilo `Italiano 1080p` assegnare score: Italiano = 100, Non Italiano = -200.
10. Minimum Custom Format Score = 50. Upgrade Until Custom Format Score = 100.
11. Settings → Download Clients → Add qBittorrent:
    - Host: `qbittorrent`
    - Port: `8080`
    - Category: `radarr`
    - Username e password di qBittorrent
12. Test e Save.

### Sonarr (Serie TV) - naming con anno

1. Settings → Media Management.
2. Root Folder: aggiungere `/tv`.
3. Attivare Rename Episodes e Replace Illegal Characters.
4. Standard Episode Format: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
5. Season Folder Format: `Season {season:00}`
6. Series Folder Format: `{Series Title} ({Series Year})`
7. Settings → Profiles → creare profilo chiamato `Italiano 1080p`.
8. Nel profilo abilitare SOLO: Bluray-1080p, WEBDL-1080p, WEBRip-1080p, HDTV-1080p.
9. Settings → Custom Formats:
   - Creare formato `Italiano` con Condition Release Title e regex `\b(ita|italian|iTALiAN|iTA|subita)\b`
   - Creare formato `Non Italiano` con Condition Release Title e regex `\b(eng|english|sub|subs|subbed)\b`
10. Nel profilo `Italiano 1080p` assegnare score: Italiano = 100, Non Italiano = -200.
11. Minimum Custom Format Score = 50. Upgrade Until Custom Format Score = 100.
12. Settings → Download Clients → Add qBittorrent:
    - Host: `qbittorrent`
    - Port: `8080`
    - Category: `sonarr`
    - Username e password di qBittorrent
13. Test e Save.

### Prowlarr

1. Settings → General → copiare l'API Key.
2. Indexers → Add Indexer → aggiungere gli indexer desiderati.
3. Settings → Apps → Add Application → Radarr:
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key di Radarr
4. Settings → Apps → Add Application → Sonarr:
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key di Sonarr
5. Test e Save per entrambe le app. Eseguire Sync App Indexers se disponibile.

### Jellyseerr

1. Collegare Jellyfin:
   - URL: `http://jellyfin:8096`
   - Credenziali di un utente amministratore di Jellyfin
2. Collegare Radarr:
   - URL: `http://radarr:7878`
   - API Key di Radarr
   - Root Folder: `/movies`
   - Quality Profile: `Italiano 1080p`
3. Collegare Sonarr:
   - URL: `http://sonarr:8989`
   - API Key di Sonarr
   - Root Folder: `/tv`
   - Quality Profile: `Italiano 1080p`
4. Completare la procedura guidata.

### Jellyfin

1. Dashboard → Libraries → Add Media Library.
2. Content type Movies, Display name `Film`, cartella `/media/Film`.
3. Add Media Library di nuovo.
4. Content type Shows, Display name `Serie TV`, cartella `/media/Serie TV`.
5. Salvare entrambe le librerie.

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

- Non esporre mai le Web UI sull'host (niente `ports:` per le interfacce web).
- Usare sempre la rete esterna `proxy-manager`.
- Le cartelle media sull'host devono chiamarsi esattamente `Film` e `Serie TV`.
- I nomi delle cartelle dei singoli film e delle singole serie devono includere l'anno:
  - Film: `{Movie Title} ({Release Year})`
  - Serie: `{Series Title} ({Series Year})`
- Preferenza lingua italiana + minimo 1080p deve essere configurata tramite Quality Profile + Custom Formats sia su Radarr sia su Sonarr (passi completi e separati per ciascun servizio).
- I file di configurazione generati (`config/`) e i media (`data/`) non devono essere versionati.
- Nelle istruzioni all'utente non usare mai frasi del tipo "stesso procedimento di Radarr" o "ripeti i passaggi precedenti": ogni sezione deve essere autonoma e completa.

## Output atteso dall'agent

Alla fine l'agent deve:
1. Aver creato tutta la struttura di cartelle
2. Aver scritto `docker-compose.yml`, `.env`, `.gitignore`, `README.md`, `AGENT.md`
3. Aver creato la rete `proxy-manager` (se mancante)
4. Aver avviato i container con `docker compose up -d`
5. Aver fornito all'utente le istruzioni chiare, complete e senza riferimenti incrociati per la configurazione di ogni singola interfaccia
