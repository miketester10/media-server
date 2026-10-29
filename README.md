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
| **FlareSolverr** | Risolve i controlli Cloudflare degli indexer                            | 8191          |
| **Violarr**      | Indexer Torznab ICVDB (contenuti italiani)                              | 8000          |

Tutte le interfacce web sono accessibili **solo** attraverso la rete Docker `proxy-manager`.  
Sull'host viene esposta **solo** la porta torrent di qBittorrent (`6881`).  
**FlareSolverr** non ha interfaccia web pubblica da mettere nel reverse proxy.

---

## Flusso

1. **Seerr** → richiesta film/serie
2. **Radarr** / **Sonarr** → cercano via **Prowlarr** (italiano + 1080p)
3. **Violarr** e altri indexer forniscono i risultati; **FlareSolverr** aiuta se c'è Cloudflare
4. **qBittorrent** scarica
5. Radarr/Sonarr organizzano in `Film/Nome (Anno)/` o `Serie TV/Nome (Anno)/Season XX/`
6. Dopo l'import, Radarr/Sonarr **rimuovono** torrent e file da qBittorrent (niente seed prolungato)
7. **Jellyfin** riproduce

---

## Installazione rapida

```bash
docker network create proxy-manager
git clone https://github.com/miketester10/media-server.git
cd media-server
mkdir -p config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,seerr,violarr}
mkdir -p data/downloads/{incomplete,completed} data/media/Film "data/media/Serie TV"
# modifica .env (PUID/PGID)
docker compose up -d
```

### Nginx Proxy Manager

| Servizio    | Hostname interno | Porta |
|-------------|------------------|-------|
| Jellyfin    | jellyfin         | 8096  |
| qBittorrent | qbittorrent      | 8080  |
| Radarr      | radarr           | 7878  |
| Sonarr      | sonarr           | 8989  |
| Prowlarr    | prowlarr         | 9696  |
| Seerr       | seerr            | 5055  |
| Violarr     | icvdb-torznab    | 8000  |

---

## Custom Format regex (solo italiano, case-insensitive)

**Italiano** (+100):

```regex
(?i)\b(ita|italian|italiano|subita)\b
```

**Non Italiano** (-200):

```regex
(?i)\b(eng|english)\b
```

`MULTI` **non** è incluso.

---

## Priorità indexer

| Indexer | Priority |
|---------|----------|
| Violarr (ICVDB) | **5** |
| Il Corsaro Blu | **10** |
| 1337x | **25** |
| Altri pubblici | **30–35** |
| Poco affidabili | **40–50** |

Imposta in Prowlarr → Indexers → Priority, poi **Sync App Indexers** su Radarr e Sonarr.  
Guida: `docs/PRIORITA-INDEXER.md`

---

## Rimuovere i file da qBittorrent dopo l'import

Se non ti interessa seedare (indexer pubblici, nessun ratio da difendere), attiva la rimozione automatica:

### Radarr
1. Settings → **Download Clients** → apri **qBittorrent**
2. **Completed Download Handling** = **ON**
3. **Remove** = **ON**
4. Test → Save

### Sonarr
1. Settings → **Download Clients** → apri **qBittorrent**
2. **Completed Download Handling** = **ON**
3. **Remove** = **ON**
4. Test → Save

Effetto: download → import in `media/Film` o `Serie TV` → qBittorrent elimina torrent e file in `downloads`.

Guida completa passo-passo: **`docs/RIMUOVI-DOPO-IMPORT.md`**

---

## Prowlarr + FlareSolverr

1. API Key da Settings → General
2. Aggiungi indexer
3. FlareSolverr Host: `http://flaresolverr:8191`, tag `flaresolverr` sugli indexer con Cloudflare

---

## Violarr (ICVDB)

1. `docker logs -f icvdb-torznab` fino a pronto
2. WebUI Violarr → collega Prowlarr:
   - URL Prowlarr: `http://prowlarr:9696`
   - URL indexer: `http://icvdb-torznab:8000/api`
3. Aggiungi Violarr a Prowlarr → Priority **5** → Sync App Indexers

Docs: https://xbit18.github.io/violarr/ · `docs/VIOLARR-E-REGEX.md`

---

## Radarr / Sonarr (sintesi)

- Root: `/movies` e `/tv`
- Naming con anno
- Import Extra Files: `srt,sub,idx,nfo,txt`
- Profile `Italiano 1080p`
- Download client qBittorrent + **Remove ON** dopo import

---

## Seerr

- Jellyfin: `http://jellyfin:8096`
- Radarr: `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr: `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

---

## Jellyfin

- Film → `/media/Film`
- Serie TV → `/media/Serie TV`

---

## Risorse

- Video guida: https://youtu.be/-PQtE6Nb0Cw
- **Rimuovi da qBit dopo import:** `docs/RIMUOVI-DOPO-IMPORT.md`
- Priorità indexer: `docs/PRIORITA-INDEXER.md`
- Violarr + regex: `docs/VIOLARR-E-REGEX.md`
- Import Extra Files: `docs/IMPORT-EXTRA-FILES-E-RISORSE.md`
