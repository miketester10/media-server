# Violarr + Custom Format regex (solo italiano)

## Regex Custom Format (senza MULTI)

**Italiano** (score +100), case-insensitive:

```regex
(?i)\b(ita|italian|italiano|subita)\b
```

**Non Italiano** (score -200):

```regex
(?i)\b(eng|english)\b
```

`MULTI` è stato **escluso** di proposito: spesso indica dual audio o lingue miste non solo italiane.

Applicare la stessa regex in **Radarr** e in **Sonarr** (Settings → Custom Formats).

---

## Violarr (indexer ICVDB per contenuti italiani)

**Violarr** è un wrapper Torznab self-hosted che espone ICVDB a Prowlarr.  
Documentazione ufficiale: https://xbit18.github.io/violarr/

### Nel docker-compose

- Image: `ghcr.io/xbit18/violarr:latest`
- Container name: `icvdb-torznab` (come da docs ufficiali)
- Porta interna: `8000`
- Volume: `config/violarr` → `/data`
- Rete: `proxy-manager` (solo `expose`, niente porte sull'host se usi reverse proxy)

### Configurazione passo-passo

1. Avvia lo stack: `docker compose up -d`
2. Attendi il primo avvio (può richiedere alcuni minuti): `docker compose logs -f violarr` oppure `docker logs -f icvdb-torznab`
3. Apri la WebUI di Violarr (tramite reverse proxy su porta 8000, hostname `icvdb-torznab` o `violarr` a seconda del compose).
4. Completa la prima configurazione in Violarr.
5. In **Prowlarr** copia l'API Key (Settings → General → Security).
6. Nella WebUI di Violarr, sezione **Prowlarr**:
   - **URL Prowlarr**: `http://prowlarr:9696`
   - **Chiave API**: quella di Prowlarr
   - **URL indexer visto da Prowlarr**: `http://icvdb-torznab:8000/api`
7. Salva impostazioni → **Verifica connessione** → deve risultare **Connesso**.
8. Premi **Aggiungi Violarr a Prowlarr**.
9. In Prowlarr deve comparire l'indexer **Violarr**. Esegui un Test.

### Nginx Proxy Manager (opzionale)

| Servizio | Hostname interno   | Porta |
|----------|--------------------|-------|
| Violarr  | icvdb-torznab      | 8000  |

---

## Nota

Violarr non sostituisce gli altri indexer: li **affianca**, ed è particolarmente utile per release in italiano collegate a ICVDB.
