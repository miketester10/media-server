# AGENT.md - Istruzioni per LLM / Agent

Replica esatta di questo media server su Linux + Docker.

## Obiettivo

- Film e Serie TV, audio italiano (Custom Formats), minimo 1080p
- Web UI solo su rete `proxy-manager`
- Cartelle `Film` / `Serie TV` con **anno** nel nome
- **Seerr**, **FlareSolverr**, **Violarr** (ICVDB)

## Servizi

jellyfin, qbittorrent, radarr, sonarr, prowlarr, seerr, flaresolverr, violarr (`icvdb-torznab`)

## Priorità indexer (obbligatorie da documentare all'utente)

| Indexer | Priority |
|---------|----------|
| Violarr (ICVDB) | **5** |
| Il Corsaro Blu | **10** |
| 1337x | **25** |
| Altri pubblici | **30–35** |
| Deboli | **40–50** |

Impostare in Prowlarr → Indexers → Priority, poi **Sync App Indexers** su Radarr e su Sonarr.  
Dettaglio: `docs/PRIORITA-INDEXER.md`

## Regex Custom Format (senza multi)

Italiano +100: `(?i)\b(ita|italian|italiano|subita)\b`  
Non Italiano -200: `(?i)\b(eng|english)\b`

## docker-compose vincoli

- Rete `proxy-manager` external; solo `expose` per le Web UI
- qBittorrent: porte 6881 TCP/UDP sull'host
- Radarr: `${DATA_DIR}/media/Film:/movies`
- Sonarr: `${DATA_DIR}/media/Serie TV:/tv`
- seerr: `ghcr.io/seerr-team/seerr:latest`, config `/app/config`
- violarr: `ghcr.io/xbit18/violarr:latest`, container_name `icvdb-torznab`, expose 8000, volume config `/data`
- flaresolverr: expose 8191

## Config post-avvio (punti chiave)

- Prowlarr: indexer + FlareSolverr `http://flaresolverr:8191` + tag `flaresolverr` dove serve
- Violarr → Prowlarr: URL Prowlarr `http://prowlarr:9696`, URL indexer `http://icvdb-torznab:8000/api`
- Priority tabella sopra + Sync App Indexers verso Radarr e Sonarr
- Radarr/Sonarr: naming con anno, Import Extra Files `srt,sub,idx,nfo,txt`, profile Italiano 1080p, regex sopra
- Seerr collegato a jellyfin/radarr/sonarr sulle hostname Docker

## Istruzioni utente

Sempre passo-passo complete; mai "stesso procedimento di...".  
Documentazione priorità: `docs/PRIORITA-INDEXER.md`.
