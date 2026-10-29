# AGENT.md – Replica dello stack media-server

Istruzioni per qualsiasi LLM/agent che deve **ricreare o aggiornare** questo progetto.

La guida utente completa e passo-passo (a prova di principiante) è solo in:

**[README.md](README.md)**

Quando spieghi qualcosa a un umano, **segui l’ordine del README** e non riassumere con “stesso procedimento di…”. Ogni servizio ha passi propri e completi.

---

## Obiettivo dello stack

- Film e serie TV automatici
- Preferenza **italiano** (Custom Format, senza `MULTI`)
- Qualità minima **1080p**
- Cartelle `Film` / `Serie TV` con **anno** nel nome
- Web UI solo su rete Docker esterna **`proxy-manager`** (`expose`, non `ports` per le UI)
- Dopo l’import: **rimozione** torrent da qBittorrent (Completed Download Handling + Remove)
- Indexer italiani prioritari: Violarr (ICVDB) > Il Corsaro Blu > 1337x

---

## Servizi Docker (obbligatori)

| Servizio | Image | container_name | Porta expose |
|----------|--------|----------------|--------------|
| jellyfin | jellyfin/jellyfin:latest | jellyfin | 8096 |
| qbittorrent | linuxserver/qbittorrent:latest | qbittorrent | 8080 (+ ports 6881) |
| radarr | lscr.io/linuxserver/radarr:latest | radarr | 7878 |
| sonarr | lscr.io/linuxserver/sonarr:latest | sonarr | 8989 |
| prowlarr | lscr.io/linuxserver/prowlarr:latest | prowlarr | 9696 |
| seerr | ghcr.io/seerr-team/seerr:latest | seerr | 5055 |
| flaresolverr | ghcr.io/flaresolverr/flaresolverr:latest | flaresolverr | 8191 |
| violarr | ghcr.io/xbit18/violarr:latest | **icvdb-torznab** | 8000 |

Rete: `proxy-manager` external.

Volume media:
- Radarr: `${DATA_DIR}/media/Film:/movies`
- Sonarr: `${DATA_DIR}/media/Serie TV:/tv`
- qBittorrent: `${DATA_DIR}/downloads:/downloads`
- Violarr: `${CONFIG_DIR}/violarr:/data`

---

## Struttura cartelle da creare

```text
media-server/
  docker-compose.yml
  .env
  .gitignore
  README.md
  AGENT.md
  docs/
  config/{jellyfin,qbittorrent,radarr,sonarr,prowlarr,seerr,violarr}/
  data/downloads/{incomplete,completed}/
  data/media/Film/
  data/media/Serie TV/
```

---

## Valori di configurazione “scritti nella pietra”

### Regex Custom Format

- Italiano (+100): `(?i)\b(ita|italian|italiano|subita)\b`
- Non Italiano (-200): `(?i)\b(eng|english)\b`
- **Mai** includere `multi`

### Naming

- Film folder/file: `{Movie Title} ({Release Year})`
- Serie folder: `{Series Title} ({Series Year})`
- Season: `Season {season:00}`
- Episode: `{Series Title} - S{season:00}E{episode:00} - {Episode Title}`

### Import Extra Files

`srt,sub,idx,nfo,txt`

### Priorità indexer

| Indexer | Priority |
|---------|----------|
| Violarr | 5 |
| Il Corsaro Blu | 10 |
| 1337x | 25 |

Poi **Sync App Indexers** da Prowlarr verso Radarr e verso Sonarr (passi separati e completi).

### Violarr → Prowlarr

- URL Prowlarr: `http://prowlarr:9696`
- URL indexer: `http://icvdb-torznab:8000/api`

### FlareSolverr in Prowlarr

- Host: `http://flaresolverr:8191`
- Tag: `flaresolverr` solo sugli indexer che falliscono Cloudflare

### qBittorrent path

- Completed: `/downloads/completed`
- Incomplete: `/downloads/incomplete`
- Categorie: `radarr` / `sonarr`

### Remove dopo import

Su **entrambi** Radarr e Sonarr, client qBittorrent:

- Completed Download Handling = ON
- Remove = ON

### Seerr

- Jellyfin `http://jellyfin:8096`
- Radarr `http://radarr:7878`, root `/movies`, profile `Italiano 1080p`
- Sonarr `http://sonarr:8989`, root `/tv`, profile `Italiano 1080p`

### Jellyfin libraries

- Film → `/media/Film`
- Serie TV → `/media/Serie TV`

---

## Documentazione secondaria (linkare, non sostituire il README)

| File | Uso |
|------|-----|
| docs/PRIORITA-INDEXER.md | Priority + sync dettagliato |
| docs/RIMUOVI-DOPO-IMPORT.md | Remove da qBit |
| docs/VIOLARR-E-REGEX.md | Violarr + regex |
| docs/IMPORT-EXTRA-FILES-E-RISORSE.md | Extra files + video |

---

## Vincoli per l’agent

1. Non esporre Web UI con `ports:` (solo `expose`), eccetto 6881 di qBittorrent.
2. Non reintrodurre nomi di request-manager legacy.
3. Non usare frasi tipo “stesso di Radarr” nelle guide utente: duplicare i passi per Sonarr.
4. Se aggiorni il compose, aggiorna tabella servizi in README e in questo file.
5. Fonte di verità per l’utente umano = **README.md** dall’inizio alla fine.
