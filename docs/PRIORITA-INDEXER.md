# Priorità indexer e sincronizzazione su Radarr / Sonarr

A **parità di Custom Format**, Radarr e Sonarr usano la **Priority** dell'indexer per scegliere quale torrent scaricare.

- Numero **più basso** = priorità **più alta**
- Range: **1** (massima) → **50** (minima)

---

## Tabella priorità consigliata (contenuti italiani)

| Indexer | Nome in Prowlarr | Priority | Motivo |
|---------|------------------|----------|--------|
| **ICVDB / Violarr** | Violarr (container `icvdb-torznab`) | **5** | Orientato ai contenuti italiani via ICVDB |
| **Il Corsaro Blu** | IlCorsaroBlu (o nome simile nella lista) | **10** | Tracker / indexer italiano, buone release IT |
| **1337x** | 1337x | **25** | Pubblico ampio; utile come fallback (spesso Cloudflare) |
| Altri indexer pubblici generici | (vari) | **30–35** | Fallback secondario |
| Indexer poco affidabili / rumorosi | (vari) | **40–50** | Ultima scelta |

### Come usare la tabella

1. In Prowlarr imposta esattamente questi numeri (o numeri vicini, purché l'ordine resti: Violarr < CorsaroBlu < 1337x).
2. Sincronizza su Radarr e Sonarr (vedi sotto).
3. Non mettere tutti gli indexer allo stesso valore: altrimenti la priorità non conta.

---

## Parte A – Impostare le priorità in Prowlarr

1. Apri **Prowlarr** nel browser.
2. Menu a sinistra → **Indexers**.
3. Clicca sul **nome** dell'indexer (es. Violarr).
4. Trova il campo **Indexer Priority** (o **Priority**).
5. Inserisci il numero dalla tabella (es. `5` per Violarr).
6. Clicca **Save**.
7. Ripeti per **Il Corsaro Blu** (`10`), **1337x** (`25`) e gli altri indexer.

---

## Parte B – Sincronizzare su Radarr

1. In Prowlarr → **Settings** → **Apps**.
2. Verifica che esista l'app **Radarr**. Se non c'è:
   - **+ Add Application** → **Radarr**
   - Name: `Radarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: quella di Radarr (Radarr → Settings → General → Security → API Key)
   - **Test** → deve essere verde → **Save**
3. Nella riga di Radarr clicca **Sync App Indexers**.
4. Apri **Radarr** → **Settings** → **Indexers**.
5. Apri un indexer e controlla che **Priority** sia uguale a quella impostata in Prowlarr.

---

## Parte C – Sincronizzare su Sonarr

1. In Prowlarr → **Settings** → **Apps**.
2. Verifica che esista l'app **Sonarr**. Se non c'è:
   - **+ Add Application** → **Sonarr**
   - Name: `Sonarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key: quella di Sonarr (Sonarr → Settings → General → Security → API Key)
   - **Test** → verde → **Save**
3. Nella riga di Sonarr clicca **Sync App Indexers**.
4. Apri **Sonarr** → **Settings** → **Indexers**.
5. Controlla che le **Priority** coincidano con Prowlarr.

---

## Parte D – Test

1. In **Radarr** apri un film → avvia una **Search** manuale.
2. A parità di score Custom Format, le release di Violarr (priority 5) dovrebbero risultare preferite rispetto a 1337x (priority 25).
3. Stesso test in **Sonarr** su una serie.

---

## Ordine di scelta (semplificato)

1. Custom Format score più alto
2. Qualità (secondo il Quality Profile)
3. **Indexer Priority** (numero più basso vince)
4. Altri fattori (seed, età release, ecc.)

---

## Se la sync non aggiorna le priority

1. Prowlarr → Apps → apri Radarr/Sonarr → **Test** → **Save**
2. Clicca di nuovo **Sync App Indexers**
3. In Radarr/Sonarr elimina eventuali indexer aggiunti a mano in passato e rifai solo la sync da Prowlarr
4. In Prowlarr assicurati che l'indexer sia **Enabled**

---

## Note su Cloudflare (1337x)

1337x spesso richiede **FlareSolverr**. In Prowlarr:
- Settings → Indexers → FlareSolverr Host: `http://flaresolverr:8191`
- Tag `flaresolverr` sull'indexer 1337x se il Test fallisce senza
