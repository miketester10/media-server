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
6. **Jellyfin** riproduce

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

`MULTI` **non** è incluso (evita dual audio / lingue miste non solo IT).

Usare le stesse regex in Radarr e Sonarr.

---

## Prowlarr + FlareSolverr

1. API Key da Settings → General
2. Aggiungi indexer dalla lista
3. FlareSolverr: Host `http://flaresolverr:8191`, tag `flaresolverr`
4. Assegna il tag solo agli indexer che falliscono per Cloudflare

---

## Violarr (ICVDB)

Documentazione: https://xbit18.github.io/violarr/

1. Attendi il primo avvio: `docker logs -f icvdb-torznab`
2. Apri la WebUI di Violarr
3. Completa la prima configurazione
4. In Violarr → Prowlarr:
   - URL Prowlarr: `http://prowlarr:9696`
   - API Key di Prowlarr
   - URL indexer: `http://icvdb-torznab:8000/api`
5. Verifica connessione → **Aggiungi Violarr a Prowlarr**
6. In Prowlarr deve comparire l'indexer Violarr → Test

Guida dettagliata: `docs/VIOLARR-E-REGEX.md`

---

## Radarr / Sonarr (sintesi)

- Root: `/movies` e `/tv`
- Naming con **anno** nelle cartelle
- Import Extra Files ON: `srt,sub,idx,nfo,txt`
- Profile `Italiano 1080p` (solo 1080p)
- Custom Formats come sopra
- Download client: `qbittorrent:8080` (categorie `radarr` / `sonarr`)
- Collegati da Prowlarr → Apps

---

## Seerr

- Jellyfin: `http://jellyfin:8096`
- Radarr: `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr: `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

---

## Jellyfin

- Libreria Film → `/media/Film`
- Libreria Serie TV → `/media/Serie TV`

---

## Risorse

- Video guida: https://youtu.be/-PQtE6Nb0Cw
- Import Extra Files: `docs/IMPORT-EXTRA-FILES-E-RISORSE.md`
- Violarr + regex: `docs/VIOLARR-E-REGEX.md`
