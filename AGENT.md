# AGENT.md – Replica dello stack media-server

Guida utente completa: **[README.md](README.md)**  
Non riassumere con “stesso procedimento di…”.

---

## Obiettivo

- Film/serie automatici, italiano (CF senza `MULTI`), min 1080p
- Cartelle `Film` / `Serie TV` con **anno**
- UI solo su rete `proxy-manager` (`expose`)
- **Pulizia downloads: qBittorrent Seeding Limits**, non Radarr
- Indexer priority: Violarr 5 > CorsaroBlu 10 > 1337x 25

---

## Servizi

jellyfin, qbittorrent, radarr, sonarr, prowlarr, seerr, flaresolverr, violarr (`icvdb-torznab`)

---

## Valori fissi

### Regex

- Italiano +100: `(?i)\b(ita|italian|italiano|subita)\b`
- Non Italiano -200: `(?i)\b(eng|english)\b`
- Mai `multi`

### Naming

- Film: `{Movie Title} ({Release Year})`
- Serie: `{Series Title} ({Series Year})` / `Season {season:00}`

### Import Extra Files

`srt,sub,idx,nfo,txt`

### qBittorrent – eliminazione file (fonte di verità)

Impostazioni → **BitTorrent** → **Seeding Limits**:

- When total seeding time reaches = **ON**, valore **5** minuti
- then = **Remove torrent and files** (non `Stop torrent`)

Radarr/Sonarr: solo **Completed Download Handling → Enable = ON** (import).  
Remove Completed sul client Radarr **non** sostituisce i Seeding Limits.

Dettaglio utente: `docs/RIMUOVI-DOPO-IMPORT.md`

### Altro

- Path qBit: `/downloads/completed`, `/downloads/incomplete`
- Categorie: `radarr` / `sonarr`
- FlareSolverr: `http://flaresolverr:8191`, tag `flaresolverr`
- Violarr → Prowlarr: `http://prowlarr:9696`, indexer `http://icvdb-torznab:8000/api`

---

## Vincoli agent

1. Documentare la pulizia downloads **solo** via qBittorrent Seeding Limits.
2. Non dire che Radarr “elimina i file dopo l’import” come metodo principale.
3. Istruzioni utente sempre passo-passo complete.
4. Fonte umana = README.md.
