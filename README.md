# Media Server – Guida completa passo-passo (Jellyfin + ARR)

Questa guida parte da **zero** e spiega **ogni click**. Non dare nulla per scontato.

Stack per film e serie TV in **italiano**, qualità minima **1080p**, tutto in Docker.

---

## Indice

1. [Cosa fa ogni programma](#1-cosa-fa-ogni-programma)
2. [Come funziona il flusso](#2-come-funziona-il-flusso)
3. [Prerequisiti](#3-prerequisiti)
4. [Installazione (cartelle, .env, Docker)](#4-installazione)
5. [Reverse proxy (Nginx Proxy Manager)](#5-reverse-proxy)
6. [Configurazione qBittorrent](#6-configurazione-qbittorrent)
7. [Configurazione Prowlarr](#7-configurazione-prowlarr)
8. [Configurazione FlareSolverr in Prowlarr](#8-flaresolverr)
9. [Configurazione Violarr (ICVDB)](#9-violarr)
10. [Priorità indexer](#10-priorità-indexer)
11. [Configurazione Radarr (film)](#11-radarr)
12. [Configurazione Sonarr (serie)](#12-sonarr)
13. [Rimuovere i file dopo il download (qBittorrent)](#13-rimuovere-dopo-import)
14. [Collegare Prowlarr a Radarr e Sonarr](#14-prowlarr-apps)
15. [Configurazione Seerr](#15-seerr)
16. [Configurazione Jellyfin](#16-jellyfin)
17. [Test finale](#17-test-finale)
18. [Documenti di dettaglio](#18-documenti-di-dettaglio)
19. [Comandi utili](#19-comandi-utili)

---

## 1. Cosa fa ogni programma

| Programma | A cosa serve | Porta interna |
|-----------|--------------|---------------|
| **Seerr** | Sito dove **cerchi e richiedi** film/serie | 5055 |
| **Radarr** | Prende le richieste **film**, cerca i torrent, organizza le cartelle | 7878 |
| **Sonarr** | Come Radarr ma per le **serie TV** | 8989 |
| **Prowlarr** | Elenco centralizzato dei **siti torrent** (indexer) | 9696 |
| **Violarr** | Indexer speciale per contenuti **italiani** (ICVDB) | 8000 |
| **FlareSolverr** | Aiuta Prowlarr quando un sito chiede “Non sono un robot” (Cloudflare) | 8191 |
| **qBittorrent** | **Scarica** i file torrent e, dopo pochi minuti, **cancella** i file da downloads | 8080 |
| **Jellyfin** | **Guardi** film e serie (come Netflix in casa) | 8096 |

---

## 2. Come funziona il flusso

1. Apri **Seerr** e clicchi **Request** su un film o una serie.
2. Seerr avvisa **Radarr** (film) o **Sonarr** (serie).
3. Radarr/Sonarr chiedono a **Prowlarr**: “trovami release in italiano, minimo 1080p”.
4. Prowlarr interroga gli indexer (Violarr, Il Corsaro Blu, 1337x, …). Se c’è Cloudflare, usa **FlareSolverr**.
5. Viene scelto un torrent e mandato a **qBittorrent**.
6. qBittorrent scarica in `data/downloads/completed`.
7. Radarr crea `data/media/Film/Nome Film (2024)/` e mette il file dentro (rinominato).
8. Sonarr crea `data/media/Serie TV/Nome Serie (2020)/Season 01/` e mette gli episodi.
9. Dopo circa **5 minuti**, **qBittorrent** (Seeding Limits) **cancella** torrent e file da `downloads`. Il file resta solo in `media/`.
10. **Jellyfin** vede i file nuovi e li puoi guardare.

> **Nota:** non è Radarr a svuotare `downloads`. Nelle UI recenti di Radarr l’opzione Remove sul client **non** elimina in modo affidabile i file torrent. Si configura in **qBittorrent → BitTorrent → Seeding Limits** (punto 6 e 13).

---

## 3. Prerequisiti

Sul server ti servono:

- Docker installato
- Docker Compose installato
- Una rete Docker chiamata `proxy-manager` (la creiamo sotto)
- (Consigliato) Nginx Proxy Manager già funzionante sulla stessa rete

```bash
docker --version
docker compose version
```

---

## 4. Installazione

### 4.1 Crea la rete Docker (una sola volta)

```bash
docker network create proxy-manager
```

### 4.2 Scarica il progetto

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server
```

### 4.3 Crea tutte le cartelle

```bash
mkdir -p config/jellyfin
mkdir -p config/qbittorrent
mkdir -p config/radarr
mkdir -p config/sonarr
mkdir -p config/prowlarr
mkdir -p config/seerr
mkdir -p config/violarr

mkdir -p data/downloads/incomplete
mkdir -p data/downloads/completed
mkdir -p data/media/Film
mkdir -p "data/media/Serie TV"
```

### 4.4 File `.env`

```bash
nano .env
```

```env
PUID=1000
PGID=1000
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Trova PUID/PGID con `id`. Salva in nano: `Ctrl+O`, Invio, `Ctrl+X`.

### 4.5 Avvia i container

```bash
docker compose up -d
docker compose ps
```

---

## 5. Reverse proxy

| Nome esempio | Forward Hostname | Forward Port |
|--------------|------------------|--------------|
| jellyfin.tuodominio.com | `jellyfin` | `8096` |
| qbittorrent.tuodominio.com | `qbittorrent` | `8080` |
| radarr.tuodominio.com | `radarr` | `7878` |
| sonarr.tuodominio.com | `sonarr` | `8989` |
| prowlarr.tuodominio.com | `prowlarr` | `9696` |
| requests.tuodominio.com | `seerr` | `5055` |
| violarr.tuodominio.com | `icvdb-torznab` | `8000` |

**Non** creare un proxy per FlareSolverr.

---

## 6. Configurazione qBittorrent

### 6.1 Base

1. Apri qBittorrent nel browser.
2. Al primo accesso imposta **username** e **password**. **Scrivili**: servono a Radarr e Sonarr.
3. Clicca l’**ingranaggio** (Impostazioni).
4. Sezione **Download**:
   - Cartella di salvataggio: `/downloads/completed`
   - Spunta file incompleti in altra cartella
   - Cartella incompleti: `/downloads/incomplete`
5. Sezione **Connessione**: porta `6881`
6. Sezione **Web UI**: controlla username e password

### 6.2 Seeding Limits (cancella i file dopo pochi minuti) — OBBLIGATORIO se non vuoi riempire downloads

Sempre nelle Impostazioni di qBittorrent:

1. Apri la sezione **BitTorrent**
2. Scorri fino a **Seeding Limits**
3. **Spunta** **When total seeding time reaches**
4. Scrivi **`5`** (minuti)
5. Nel menu **then** scegli **Remove torrent and files**  
   (in italiano: **Rimuovi torrent e file**)  
   **Non** lasciare `Stop torrent`
6. Clicca **Save** / **Salva**

| Campo | Valore |
|-------|--------|
| When total seeding time reaches | ON, **5** minuti |
| then | **Remove torrent and files** |

I 5 minuti danno tempo a Radarr/Sonarr di importare in `media/` prima della cancellazione. Se a volte i file spariscono troppo presto, alza a **10**.

Guida dedicata: [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md)

---

## 7. Configurazione Prowlarr

### 7.1 API Key

1. Prowlarr → **Settings** → **General** → **API Key** → copiala.

### 7.2 Indexer

1. **Indexers** → **+ Add Indexer**
2. Aggiungi **1337x**, **IlCorsaroBlu**, altri che usi
3. Per ciascuno: **Test** → se verde **Save**. Se fallisce per Cloudflare → punto 8

**Violarr** si collega al punto 9, non dalla lista generica.

---

## 8. FlareSolverr

1. Prowlarr → **Settings** → **Indexers** → FlareSolverr **+**
2. Name: `FlareSolverr` · Tags: `flaresolverr` · Host: `http://flaresolverr:8191`
3. Test → Save
4. Sull’indexer problematico (es. 1337x) aggiungi il tag `flaresolverr` → Test → Save

---

## 9. Violarr (ICVDB)

Docs: https://xbit18.github.io/violarr/

```bash
docker logs -f icvdb-torznab
```

1. Apri WebUI Violarr
2. Sezione Prowlarr:
   - URL Prowlarr: `http://prowlarr:9696`
   - API Key di Prowlarr
   - URL indexer: `http://icvdb-torznab:8000/api`
3. Verifica connessione → **Aggiungi Violarr a Prowlarr**
4. In Prowlarr → Indexers: Test su Violarr

Dettagli: [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md)

---

## 10. Priorità indexer

| Indexer | Priority |
|---------|----------|
| Violarr | **5** |
| Il Corsaro Blu | **10** |
| 1337x | **25** |

Prowlarr → Indexers → apri ciascuno → Priority → Save.  
Sync verso Radarr/Sonarr al punto 14.  
Guida: [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md)

---

## 11. Radarr (film)

### 11.1 Media Management

1. Root Folder: `/movies`
2. Rename Movies ON
3. Movie Folder Format: `{Movie Title} ({Release Year})`
4. Standard Movie Format: `{Movie Title} ({Release Year})`
5. Import Extra Files ON: `srt,sub,idx,nfo,txt`
6. Save

### 11.2 Profile `Italiano 1080p`

Solo qualità 1080p (Bluray/WEBDL/WEBRip/HDTV-1080p).

### 11.3 Custom Formats

**Italiano** (+100), regex **senza MULTI**:

```text
(?i)\b(ita|italian|italiano|subita)\b
```

**Non Italiano** (-200):

```text
(?i)\b(eng|english)\b
```

Nel profilo: Minimum Score `50`, Upgrade Until `100`.

### 11.4 qBittorrent

- Host `qbittorrent` · Port `8080` · Category `radarr`
- Test → Save
- **Settings → Download Clients**: **Completed Download Handling → Enable = ON** (serve all’import; **non** cancella i file da downloads)

---

## 12. Sonarr (serie)

Ripeti i passi completi (non “come Radarr”):

- Root: `/tv`
- Series Folder: `{Series Title} ({Series Year})`
- Season: `Season {season:00}`
- Episode: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`
- Stessi Custom Format e profile `Italiano 1080p`
- qBittorrent category `sonarr`
- Completed Download Handling **Enable = ON**

---

## 13. Rimuovere i file dopo il download (qBittorrent)

**Qui si fa in qBittorrent, non in Radarr.**

Se al punto 6.2 non l’hai già fatto:

1. qBittorrent → ingranaggio → **BitTorrent** → **Seeding Limits**
2. Spunta **When total seeding time reaches** → valore **5**
3. **then** → **Remove torrent and files**
4. Salva

In Radarr/Sonarr serve solo:

- Completed Download Handling **Enable** = ON (import automatico)

Guida completa con test e problemi comuni:  
**[`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md)**

---

## 14. Collegare Prowlarr a Radarr e Sonarr

### Radarr in Prowlarr

1. Copia API Key da Radarr → Settings → General
2. Prowlarr → Settings → Apps → + → Radarr
3. Prowlarr Server `http://prowlarr:9696` · Radarr Server `http://radarr:7878` · API Key
4. Test → Save → **Sync App Indexers**

### Sonarr in Prowlarr

1. Copia API Key da Sonarr → Settings → General
2. Prowlarr → Settings → Apps → + → Sonarr
3. Sonarr Server `http://sonarr:8989` · API Key
4. Test → Save → **Sync App Indexers**

Verifica Priority Violarr = 5 in Radarr e Sonarr → Settings → Indexers.

---

## 15. Seerr

- Jellyfin: `http://jellyfin:8096` (admin)
- Radarr: `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr: `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

---

## 16. Jellyfin

- Libreria Film → `/media/Film` (Movies)
- Libreria Serie TV → `/media/Serie TV` (Shows)

---

## 17. Test finale

1. Request film da Seerr
2. Download in qBittorrent (categoria `radarr`)
3. Import in `data/media/Film/Nome (Anno)/`
4. Dopo 5–10 minuti: torrent sparito da qBit, cartella downloads pulita, file ancora in Film/
5. Stesso test per una serie

---

## 18. Documenti di dettaglio

| File | Argomento |
|------|-----------|
| [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md) | **Eliminazione file: Seeding Limits qBittorrent** |
| [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md) | Priority + Sync |
| [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md) | Violarr + regex |
| [`docs/IMPORT-EXTRA-FILES-E-RISORSE.md`](docs/IMPORT-EXTRA-FILES-E-RISORSE.md) | srt/nfo + video |
| [`AGENT.md`](AGENT.md) | Replica con LLM |

Video di supporto: https://youtu.be/-PQtE6Nb0Cw

---

## 19. Comandi utili

```bash
docker compose up -d
docker compose ps
docker compose logs -f radarr
docker logs -f icvdb-torznab
docker compose pull && docker compose up -d
docker compose down
```

---

## Note finali

- `config/` e `data/` non vanno su GitHub
- Cartelle film/serie **con anno**
- Regex italiano **senza** `MULTI`
- **Pulizia downloads = qBittorrent Seeding Limits**, non Radarr
- Ordine setup: qBittorrent (base + Seeding Limits) → Prowlarr → FlareSolverr → Violarr → Radarr → Sonarr → Sync Apps → Seerr → Jellyfin
