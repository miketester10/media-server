# AGENT.md – Replica dello stack media-server

Guida utente: **[README.md](README.md)**

---

## Obiettivo

Film/serie IT 1080p, cartelle con anno, rete `proxy-manager`, pulizia downloads via **qBittorrent Seeding Limits**.

---

## Seerr – permessi (crash loop)

L’immagine Seerr **non** usa `PUID`/`PGID`. Node nel container gira come **UID 1000 / GID 1000**.

Se l’host crea `config/seerr` con utente 1001:1001, Seerr non scrive e il container resta in **Restarting**.

Obbligatorio documentare e applicare:

```bash
sudo chown -R 1000:1000 config/seerr
docker compose restart seerr
```

Radarr/Sonarr/qBittorrent/Prowlarr possono restare su PUID/PGID host (es. 1001); **solo** `config/seerr` deve essere 1000:1000.

---

## Altri valori fissi

- Regex IT: `(?i)\b(ita|italian|italiano|subita)\b` (no MULTI)
- qBit Seeding Limits: 5 min → Remove torrent and files
- Violarr: Italiano preferito + correzione sottotitoli IT
- Priority: Violarr 5, CorsaroBlu 10, 1337x 25
