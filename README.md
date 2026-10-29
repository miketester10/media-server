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

> **Nota:** non è Radarr a svuotare `downloads`. Si configura in **qBittorrent → BitTorrent → Seeding Limits** (punto 6 e 13).

---

## 3. Prerequisiti

- Docker e Docker Compose
- Rete `proxy-manager`
- (Consigliato) Nginx Proxy Manager sulla stessa rete

```bash
docker --version
docker compose version
```

---

## 4. Installazione

### 4.1 Rete

```bash
docker network create proxy-manager
```

### 4.2 Clone

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server
```

### 4.3 Cartelle

```bash
mkdir -p config/jellyfin config/qbittorrent config/radarr config/sonarr config/prowlarr config/seerr config/violarr
mkdir -p data/downloads/incomplete data/downloads/completed data/media/Film "data/media/Serie TV"
```

### 4.4 `.env`

```env
PUID=1000
PGID=1000
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

### 4.5 Avvio

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

1. Apri qBittorrent → imposta username e password (servono a Radarr/Sonarr).
2. Ingranaggio → **Download**: completed `/downloads/completed`, incomplete `/downloads/incomplete`.
3. **Connessione**: porta `6881`.

### 6.2 Seeding Limits (pulizia downloads)

1. Impostazioni → **BitTorrent** → **Seeding Limits**
2. Spunta **When total seeding time reaches** → **5** minuti
3. **then** → **Remove torrent and files** (non `Stop torrent`)
4. Salva

Dettaglio: [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md)

---

## 7. Configurazione Prowlarr

1. Settings → General → copia **API Key**.
2. Indexers → aggiungi **1337x**, **IlCorsaroBlu**, ecc. → Test → Save.
3. Violarr si collega al punto 9.

---

## 8. FlareSolverr

1. Prowlarr → Settings → Indexers → FlareSolverr Host: `http://flaresolverr:8191`, tag `flaresolverr`
2. Assegna il tag agli indexer con Cloudflare (es. 1337x) → Test → Save

---

## 9. Violarr (ICVDB)

Documentazione ufficiale: https://xbit18.github.io/violarr/

### 9.1 Avvio e collegamento a Prowlarr

```bash
docker logs -f icvdb-torznab
```

1. Apri la WebUI di Violarr.
2. Sezione **Prowlarr**:
   - URL Prowlarr: `http://prowlarr:9696`
   - Chiave API: quella di Prowlarr
   - URL indexer: `http://icvdb-torznab:8000/api`
3. Verifica connessione → **Aggiungi Violarr a Prowlarr**.
4. In Prowlarr → Indexers: **Test** su Violarr.

### 9.2 Elaborazione risultati (italiano)

In Violarr apri **Impostazioni** → **Elaborazione risultati**.

1. Seleziona **Italiano preferito**  
   (mostra prima le release probabilmente italiane, **senza** eliminare le altre).

   | Opzione | Quando usarla |
   |---------|----------------|
   | Senza filtri | Non consigliata (vede anche cinese/eng senza priorità) |
   | **Italiano preferito** | **Consigliata** – bilanciata |
   | Solo italiano | Più stretta: esclude ciò che non ha indicatore IT esplicito |
   | Personalizzato | Solo se definisci regole tue |

2. Attiva **Correggi il riconoscimento dei sottotitoli italiani** (ON).  
   Evita che release con **solo sottotitoli** italiani vengano trattate come se avessero **audio** italiano.

3. Clicca **Salva elaborazione risultati**.

Il filtro “duro” resta nei Custom Format di Radarr/Sonarr (regex senza `MULTI`).

Dettagli extra: [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md)

---

## 10. Priorità indexer

| Indexer | Priority |
|---------|----------|
| Violarr | **5** |
| Il Corsaro Blu | **10** |
| 1337x | **25** |

Prowlarr → Indexers → Priority → Save → Sync al punto 14.  
Guida: [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md)

---

## 11. Radarr (film)

- Root `/movies`, naming con anno
- Import Extra Files: `srt,sub,idx,nfo,txt`
- Profile `Italiano 1080p` (solo 1080p)
- Custom Format Italiano: `(?i)\b(ita|italian|italiano|subita)\b` (+100)
- Non Italiano: `(?i)\b(eng|english)\b` (-200)
- qBittorrent: category `radarr` · Completed Download Handling **Enable ON**

---

## 12. Sonarr (serie)

- Root `/tv`, cartella con anno, `Season {season:00}`
- Stessi Custom Format e profile
- qBittorrent category `sonarr` · Enable import ON

---

## 13. Rimuovere i file dopo il download (qBittorrent)

**Si fa in qBittorrent**, non in Radarr:

1. BitTorrent → Seeding Limits
2. Tempo seed **5** minuti → **Remove torrent and files**

Guida: [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md)

---

## 14. Collegare Prowlarr a Radarr e Sonarr

- Apps → Radarr: `http://radarr:7878` + API Key → Sync App Indexers
- Apps → Sonarr: `http://sonarr:8989` + API Key → Sync App Indexers

---

## 15. Seerr

- Jellyfin `http://jellyfin:8096`
- Radarr `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

---

## 16. Jellyfin

- Film → `/media/Film`
- Serie TV → `/media/Serie TV`

---

## 17. Test finale

Request → download → import in `media/` → dopo 5–10 min downloads vuoto, file ancora in Film/Serie TV.

---

## 18. Documenti di dettaglio

| File | Argomento |
|------|-----------|
| [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md) | Seeding Limits qBittorrent |
| [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md) | Priority + Sync |
| [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md) | Violarr + regex |
| [`docs/IMPORT-EXTRA-FILES-E-RISORSE.md`](docs/IMPORT-EXTRA-FILES-E-RISORSE.md) | srt/nfo + video |
| [`AGENT.md`](AGENT.md) | Replica con LLM |

Video: https://youtu.be/-PQtE6Nb0Cw

---

## 19. Comandi utili

```bash
docker compose up -d
docker compose ps
docker compose logs -f radarr
docker logs -f icvdb-torznab
docker compose pull && docker compose up -d
```

---

## Note finali

- Regex italiano **senza** `MULTI`
- Pulizia downloads = **qBittorrent Seeding Limits**
- Violarr: **Italiano preferito** + correzione sottotitoli IT
- Ordine setup: qBittorrent → Prowlarr → FlareSolverr → Violarr → Radarr → Sonarr → Sync → Seerr → Jellyfin
