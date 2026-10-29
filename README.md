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
13. [Rimuovere i file da qBittorrent dopo l’import](#13-rimuovere-dopo-import)
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
| **qBittorrent** | **Scarica** i file torrent | 8080 |
| **Jellyfin** | **Guardi** film e serie (come Netflix in casa) | 8096 |

Non devi memorizzare le porte a mente: le userai nel reverse proxy una volta sola.

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
9. Radarr/Sonarr **cancellano** il torrent da qBittorrent (così non riempi il disco in downloads).
10. **Jellyfin** vede i file nuovi e li puoi guardare.

---

## 3. Prerequisiti

Sul server ti servono:

- Docker installato
- Docker Compose installato
- Una rete Docker chiamata `proxy-manager` (la creiamo sotto)
- (Consigliato) Nginx Proxy Manager già funzionante sulla stessa rete

Controlla Docker:

```bash
docker --version
docker compose version
```

Se i comandi non esistono, installa Docker prima di continuare.

---

## 4. Installazione

### 4.1 Crea la rete Docker (una sola volta)

```bash
docker network create proxy-manager
```

Se dice che esiste già, va bene: continua.

### 4.2 Scarica il progetto

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server
```

### 4.3 Crea tutte le cartelle

Copia e incolla **tutto** questo blocco:

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

Apri il file:

```bash
nano .env
```

Deve contenere (modifica PUID e PGID):

```env
PUID=1000
PGID=1000
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Come trovare i tuoi numeri:

```bash
id
```

Esempio di output: `uid=1000(ubuntu) gid=1000(ubuntu)` → usa `1000` e `1000`.

Salva in nano: `Ctrl+O`, Invio, poi `Ctrl+X`.

### 4.5 Avvia i container

```bash
docker compose up -d
```

Controlla che siano su:

```bash
docker compose ps
```

Devono comparire almeno: `jellyfin`, `qbittorrent`, `radarr`, `sonarr`, `prowlarr`, `seerr`, `flaresolverr`, `icvdb-torznab` (Violarr).

---

## 5. Reverse proxy

In **Nginx Proxy Manager** crea un Proxy Host per ogni servizio con interfaccia web.

| Nome che vedi nel browser (esempio) | Forward Hostname / IP | Forward Port |
|-------------------------------------|------------------------|--------------|
| jellyfin.tuodominio.com | `jellyfin` | `8096` |
| qbittorrent.tuodominio.com | `qbittorrent` | `8080` |
| radarr.tuodominio.com | `radarr` | `7878` |
| sonarr.tuodominio.com | `sonarr` | `8989` |
| prowlarr.tuodominio.com | `prowlarr` | `9696` |
| requests.tuodominio.com | `seerr` | `5055` |
| violarr.tuodominio.com | `icvdb-torznab` | `8000` |

**Non** creare un proxy per FlareSolverr (non ha sito web).

Scheme: `http`. La rete del container NPM deve essere `proxy-manager`.

Da qui in poi, quando scriviamo “apri Radarr”, significa aprire l’URL che hai scelto (es. `https://radarr.tuodominio.com`).

---

## 6. Configurazione qBittorrent

1. Apri qBittorrent nel browser.
2. Al primo accesso imposta **username** e **password**. **Scrivili** da qualche parte: servono a Radarr e Sonarr.
3. Clicca l’**ingranaggio** (Impostazioni).
4. Sezione **Download**:
   - Cartella di salvataggio predefinita: scrivi esattamente  
     `/downloads/completed`
   - Spunta l’opzione per tenere i file incompleti in un’altra cartella.
   - Cartella incompleti:  
     `/downloads/incomplete`
5. Sezione **Connessione**:
   - Porta di ascolto: `6881`
6. Sezione **Web UI**: controlla username e password.
7. Clicca **Salva**.

---

## 7. Configurazione Prowlarr

### 7.1 Copia l’API Key

1. Apri Prowlarr.
2. **Settings** (ingranaggio) → **General**.
3. Sezione **Security** → campo **API Key**.
4. Copia la chiave e salvala in un blocco note (ti serve per Violarr e per le App).

### 7.2 Aggiungi gli indexer

1. Menu a sinistra → **Indexers**.
2. Clicca **+ Add Indexer**.
3. Cerca e aggiungi, uno alla volta:
   - **1337x**
   - **IlCorsaroBlu** (cerca “Corsaro” se il nome non è identico)
   - eventuali altri che usi
4. Per ogni indexer:
   - Il **Base URL** è già compilato. Se il Test fallisce, prova un altro URL dal menu a tendina.
   - Clicca **Test**. Se è verde, **Save**.
   - Se è rosso per Cloudflare, passerai da FlareSolverr al punto 8.

**Violarr** non si aggiunge dalla lista come gli altri: si collega al punto 9.

---

## 8. FlareSolverr

Serve quando un indexer (spesso 1337x) mostra il controllo Cloudflare.

1. In Prowlarr: **Settings** → **Indexers**.
2. Scorri fino a **FlareSolverr Servers** (o FlareSolverr).
3. Clicca **+**.
4. Compila:
   - **Name**: `FlareSolverr`
   - **Tags**: scrivi `flaresolverr` e premi Invio (crei il tag)
   - **Host**: `http://flaresolverr:8191`
5. **Test** → deve essere verde → **Save**.
6. Torna su **Indexers**.
7. Apri un indexer che falliva (es. 1337x).
8. Nel campo **Tags** aggiungi `flaresolverr`.
9. **Test** → se verde, **Save**.
10. Ripeti solo per gli indexer che ne hanno bisogno.

---

## 9. Violarr (ICVDB – contenuti italiani)

Documentazione ufficiale: https://xbit18.github.io/violarr/

### 9.1 Attendi l’avvio

```bash
docker logs -f icvdb-torznab
```

Quando i log dicono che è pronto, esci con `Ctrl+C`.

### 9.2 Prima configurazione

1. Apri la WebUI di Violarr (URL del reverse proxy sulla porta 8000).
2. Completa la procedura iniziale a schermo (se richiesta).

### 9.3 Collega Prowlarr da Violarr

1. Nella WebUI di Violarr apri la sezione **Prowlarr**.
2. Inserisci:
   - **URL Prowlarr**: `http://prowlarr:9696`
   - **Chiave API**: quella copiata al punto 7.1
   - **URL indexer visto da Prowlarr**: `http://icvdb-torznab:8000/api`
3. **Salva impostazioni Prowlarr**.
4. **Verifica connessione** → deve risultare **Connesso**.
5. Premi **Aggiungi Violarr a Prowlarr**.
6. Apri Prowlarr → **Indexers**: deve comparire **Violarr**. Fai **Test**.

Dettagli extra: [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md)

---

## 10. Priorità indexer

A parità di qualità e di “punteggio italiano”, vince l’indexer con **Priority** più bassa (1 = migliore).

| Indexer | Priority da impostare |
|---------|------------------------|
| Violarr (ICVDB) | **5** |
| Il Corsaro Blu | **10** |
| 1337x | **25** |
| Altri pubblici | **30–35** |
| Poco affidabili | **40–50** |

### Come impostarle

1. Prowlarr → **Indexers**.
2. Apri **Violarr** → campo **Priority** → `5` → **Save**.
3. Apri **Il Corsaro Blu** → Priority `10` → **Save**.
4. Apri **1337x** → Priority `25` → **Save**.
5. Ripeti per gli altri.

La sincronizzazione verso Radarr e Sonarr si fa al **punto 14** (dopo aver configurato Radarr e Sonarr).

Guida estesa: [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md)

---

## 11. Radarr (film)

Apri **Radarr**.

### 11.1 Cartelle e nomi (con anno)

1. **Settings** → **Media Management**.
2. **Root Folders** → **Add Root Folder** → percorso: `/movies` → Ok.
3. Attiva **Rename Movies** (ON).
4. Attiva **Replace Illegal Characters** (ON).
5. **Standard Movie Format**:
   ```text
   {Movie Title} ({Release Year})
   ```
6. **Movie Folder Format**:
   ```text
   {Movie Title} ({Release Year})
   ```
7. **Import Extra Files**: ON  
   Estensioni:
   ```text
   srt,sub,idx,nfo,txt
   ```
8. (Consigliato) **Use Hardlinks instead of Copy**: ON se downloads e media sono sullo stesso disco.
9. **Save** in alto a destra.

Risultato esempio: `Film/Il Padrino (1972)/`

### 11.2 Quality Profile (solo 1080p)

1. **Settings** → **Profiles**.
2. **+ Add**.
3. Nome: `Italiano 1080p`
4. Nella lista qualità **disattiva tutto** tranne:
   - Bluray-1080p
   - WEBDL-1080p
   - WEBRip-1080p
   - HDTV-1080p
5. **Save**.
6. Se possibile, imposta questo profilo come default.

### 11.3 Custom Formats (preferenza italiano)

1. **Settings** → **Custom Formats**.
2. **+ Add** → Nome: `Italiano`
3. **+** sotto Conditions → tipo **Release Title**.
4. Regular Expression (copia esatta):
   ```text
   (?i)\b(ita|italian|italiano|subita)\b
   ```
5. **Save**.
6. **+ Add** di nuovo → Nome: `Non Italiano`
7. Condition Release Title, regex:
   ```text
   (?i)\b(eng|english)\b
   ```
8. **Save**.
9. **Settings** → **Profiles** → apri `Italiano 1080p`.
10. Custom Formats:
    - `Italiano` → score **100**
    - `Non Italiano` → score **-200**
11. **Minimum Custom Format Score**: `50`
12. **Upgrade Until Custom Format Score**: `100`
13. **Save**.

### 11.4 Collegamento a qBittorrent

1. **Settings** → **Download Clients**.
2. **+ Add** → **qBittorrent**.
3. Compila:
   - Name: `qBittorrent`
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: (quello di qBittorrent)
   - Password: (quella di qBittorrent)
   - Category: `radarr`
4. **Test** → verde → **Save**.

Poi completa anche il **punto 13** (Remove dopo import).

---

## 12. Sonarr (serie TV)

Apri **Sonarr**. Non riassumere: fai tutti i passi anche se somigliano a Radarr.

### 12.1 Cartelle e nomi (con anno + stagioni)

1. **Settings** → **Media Management**.
2. **Root Folders** → **Add Root Folder** → `/tv` → Ok.
3. Attiva **Rename Episodes** (ON).
4. Attiva **Replace Illegal Characters** (ON).
5. **Standard Episode Format**:
   ```text
   {Series Title} - S{season:00}E{episode:00} - {Episode Title}
   ```
6. **Season Folder Format**:
   ```text
   Season {season:00}
   ```
7. **Series Folder Format**:
   ```text
   {Series Title} ({Series Year})
   ```
8. **Import Extra Files**: ON  
   Estensioni: `srt,sub,idx,nfo,txt`
9. Use Hardlinks instead of Copy: ON se possibile.
10. **Save**.

Esempio: `Serie TV/Breaking Bad (2008)/Season 01/`

### 12.2 Quality Profile

1. **Settings** → **Profiles** → **+ Add**.
2. Nome: `Italiano 1080p`
3. Solo: Bluray-1080p, WEBDL-1080p, WEBRip-1080p, HDTV-1080p.
4. **Save**.

### 12.3 Custom Formats

1. **Settings** → **Custom Formats**.
2. Crea `Italiano` con regex:
   ```text
   (?i)\b(ita|italian|italiano|subita)\b
   ```
3. Crea `Non Italiano` con regex:
   ```text
   (?i)\b(eng|english)\b
   ```
4. Nel profilo `Italiano 1080p`:
   - Italiano = **100**
   - Non Italiano = **-200**
   - Minimum Score = `50`
   - Upgrade Until = `100`
5. **Save**.

### 12.4 Collegamento a qBittorrent

1. **Settings** → **Download Clients** → **+ Add** → **qBittorrent**.
2. Host: `qbittorrent` · Port: `8080` · Category: `sonarr`
3. Username e password di qBittorrent.
4. **Test** → **Save**.

Poi fai il **punto 13** anche per Sonarr.

---

## 13. Rimuovere dopo import (niente seed lungo)

Così, finito l’import, qBittorrent cancella torrent e file in `downloads`.

### In Radarr

1. **Settings** → **Download Clients** → apri **qBittorrent**.
2. **Completed Download Handling** = **ON**.
3. **Remove** = **ON**.
4. **Test** → **Save**.

### In Sonarr

1. **Settings** → **Download Clients** → apri **qBittorrent**.
2. **Completed Download Handling** = **ON**.
3. **Remove** = **ON**.
4. **Test** → **Save**.

Guida dedicata: [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md)

---

## 14. Collegare Prowlarr a Radarr e a Sonarr

### 14.1 API Key di Radarr

1. Apri Radarr → **Settings** → **General**.
2. Copia **API Key**.

### 14.2 Aggiungi Radarr in Prowlarr

1. Apri Prowlarr → **Settings** → **Apps**.
2. **+ Add Application** → **Radarr**.
3. Compila:
   - Name: `Radarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: (quella di Radarr)
4. **Test** → verde → **Save**.
5. Sulla riga di Radarr clicca **Sync App Indexers**.

### 14.3 API Key di Sonarr

1. Apri Sonarr → **Settings** → **General**.
2. Copia **API Key**.

### 14.4 Aggiungi Sonarr in Prowlarr

1. Prowlarr → **Settings** → **Apps**.
2. **+ Add Application** → **Sonarr**.
3. Compila:
   - Name: `Sonarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key: (quella di Sonarr)
4. **Test** → **Save**.
5. **Sync App Indexers** sulla riga di Sonarr.

### 14.5 Verifica priorità

1. Radarr → Settings → Indexers → apri Violarr → Priority deve essere `5`.
2. Sonarr → stessa verifica.

Se i numeri non tornano, rifai Sync e consulta [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md).

---

## 15. Seerr (richieste)

1. Apri **Seerr** nel browser.
2. Segui la procedura guidata iniziale.
3. Collegamento **Jellyfin**:
   - URL: `http://jellyfin:8096`
   - Username e password di un utente **amministratore** Jellyfin
   - Test / Continua
4. Collegamento **Radarr**:
   - URL: `http://radarr:7878`
   - API Key di Radarr
   - Root Folder: `/movies`
   - Quality Profile: `Italiano 1080p`
   - Test e Salva
5. Collegamento **Sonarr**:
   - URL: `http://sonarr:8989`
   - API Key di Sonarr
   - Root Folder: `/tv`
   - Quality Profile: `Italiano 1080p`
   - Test e Salva
6. Completa la procedura fino alla fine.

---

## 16. Jellyfin

1. Apri **Jellyfin**.
2. Se è la prima volta, crea l’utente amministratore.
3. **Dashboard** → **Libraries** (Librerie).
4. **Add Media Library**:
   - Content type: **Movies**
   - Display name: `Film`
   - Cartella: `/media/Film`
   - Salva
5. **Add Media Library** di nuovo:
   - Content type: **Shows**
   - Display name: `Serie TV`
   - Cartella: `/media/Serie TV`
   - Salva

---

## 17. Test finale

### Film

1. In **Seerr** cerca un film e fai **Request**.
2. **Radarr** → Activity: compare in coda.
3. **qBittorrent**: download con categoria `radarr`.
4. A fine download: cartella `data/media/Film/Nome Film (Anno)/`.
5. Il torrent sparisce da qBittorrent (Remove attivo).
6. **Jellyfin**: il film appare (eventualmente “Scan library”).

### Serie

1. Request di una serie in Seerr.
2. **Sonarr** → Activity.
3. qBittorrent categoria `sonarr`.
4. Cartella `data/media/Serie TV/Nome (Anno)/Season 01/`.
5. Torrent rimosso da qBit.
6. Serie visibile in Jellyfin.

---

## 18. Documenti di dettaglio

Usa questi file quando ti serve approfondire **un solo argomento**:

| File | Argomento |
|------|-----------|
| [`docs/PRIORITA-INDEXER.md`](docs/PRIORITA-INDEXER.md) | Priority + Sync App Indexers |
| [`docs/RIMUOVI-DOPO-IMPORT.md`](docs/RIMUOVI-DOPO-IMPORT.md) | Remove da qBittorrent dopo import |
| [`docs/VIOLARR-E-REGEX.md`](docs/VIOLARR-E-REGEX.md) | Violarr + regex italiano |
| [`docs/IMPORT-EXTRA-FILES-E-RISORSE.md`](docs/IMPORT-EXTRA-FILES-E-RISORSE.md) | File `.srt` / `.nfo` + video guida |
| [`AGENT.md`](AGENT.md) | Istruzioni per replicare lo stack con un LLM |

Video di supporto (stack ARR in generale): https://youtu.be/-PQtE6Nb0Cw

---

## 19. Comandi utili

```bash
# Avvio
docker compose up -d

# Stato
docker compose ps

# Log di un servizio
docker compose logs -f radarr
docker compose logs -f sonarr
docker compose logs -f prowlarr
docker logs -f icvdb-torznab

# Aggiornamento immagini
docker compose pull
docker compose up -d

# Stop
docker compose down
```

---

## Note finali

- Le cartelle `config/` e `data/` **non** vanno su GitHub (sono nel `.gitignore`).
- I nomi cartella film/serie includono **sempre l’anno**.
- La regex italiano **non** include `MULTI` (evita dual audio generici).
- Ordine consigliato di configurazione: **qBittorrent → Prowlarr → FlareSolverr → Violarr → Radarr → Sonarr → Remove → Sync Apps → Seerr → Jellyfin**.
